# Security & Performance Review: `epb-data-warehouse`

**Date:** 27 March 2026

---

## Table of Contents

1. [Critical — Security](#critical--security)
2. [High — Security](#high--security)
3. [Medium — Security](#medium--security)
4. [Critical — Performance](#critical--performance)
5. [High — Performance](#high--performance)
6. [Summary Table](#summary-table)

---

## Critical — Security

### 1. SQL Injection — `lib/helper/generate_json_samples.rb` (lines 38–41)

```ruby
sql = "DELETE FROM assessment_documents WHERE assessment_id ='#{assessment_id}'"
ActiveRecord::Base.connection.exec_query(sql)
sql = "DELETE  FROM assessment_search WHERE assessment_id ='#{assessment_id}'"
ActiveRecord::Base.connection.exec_query(sql)
```

Raw string interpolation into SQL with no parameterisation. A maliciously crafted `assessment_id` can execute arbitrary SQL. Every other gateway in the project correctly uses parameterised bindings — this is the single outlier and must be fixed.

**Fix:** Use parameterised bindings identical to the rest of the codebase:

```ruby
def self.delete_rows(assessment_id:)
  bindings = [
    ActiveRecord::Relation::QueryAttribute.new(
      "assessment_id", assessment_id, ActiveRecord::Type::String.new
    )
  ]
  ActiveRecord::Base.connection.exec_query(
    "DELETE FROM assessment_documents WHERE assessment_id = $1", "SQL", bindings
  )
  ActiveRecord::Base.connection.exec_query(
    "DELETE FROM assessment_search WHERE assessment_id = $1", "SQL", bindings
  )
  nil
end
```

---

### 2. Bearer Token Authentication via Full DynamoDB Table Scan — `lib/gateway/user_credentials_gateway.rb` (line 13)

```ruby
resp = @table.scan(
  filter_expression: "BearerToken = :bearer_token",
  ...
)
```

`DynamoDB::Table#scan` reads **every item in the table** to match the filter — O(n) per authentication request. This is both a **critical performance bottleneck** and a **timing side-channel** (response time leaks table size). Authentication cost grows unboundedly as the table grows.

**Fix:** Use `get_item` with `bearer_token` as the partition key, making authentication O(1):

```ruby
def bearer_token_exists?(bearer_token)
  resp = @table.get_item(key: { "BearerToken" => bearer_token })
  !resp.item.nil?
end
```

---

### 3. Sensitive Exception Messages Leaked to HTTP Clients — `lib/controller/base_controller.rb` and others

```ruby
json_api_response code: 500, data: { errors: [{ code: e.message }] }
```

`e.message` from `StandardError` subclasses is returned verbatim in 500 responses. This can expose DB schema names, internal file paths, query fragments, and other internal details to external callers.

**Fix:** Return a generic message and rely on Sentry for the real detail:

```ruby
rescue StandardError => e
  report_to_sentry(e)
  json_api_response code: 500, data: { errors: [{ code: "INTERNAL_SERVER_ERROR", title: "An unexpected error occurred" }] }
end
```

---

### 4. Bearer Tokens Stored in Plaintext in DynamoDB

The `EPB_DATA_USER_CREDENTIAL_TABLE_NAME` table stores raw bearer tokens. If the DynamoDB table is accessed by an attacker (misconfigured IAM policy, compromised credentials), all API access tokens are exposed in clear text.

**Fix:** Store only a salted hash (e.g. SHA-256 with a server-side secret) of each bearer token. On authentication, hash the incoming token and compare against stored hashes.

---

## High — Security

### 5. Missing and Misconfigured Security Headers — `lib/middleware/headers_policy.rb`

The middleware **actively deletes** two headers and sets a critically mis-configured HSTS:

```ruby
headers["Strict-Transport-Security"] = "max-age=300; includeSubDomains; preload"  # 5 minutes only
headers.delete "x-frame-options"    # deliberately removed
headers.delete "x-xss-protection"   # deliberately removed
```

Issues:
- `max-age=300` makes HSTS effectively useless — browsers discard the policy after 5 minutes. The minimum for a production service is `max-age=31536000` (1 year).
- No `X-Content-Type-Options: nosniff` header.
- No `Content-Security-Policy` (CSP) header.

**Fix:**

```ruby
headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
headers["X-Content-Type-Options"] = "nosniff"
headers["Content-Security-Policy"] = "default-src 'none'"
```

---

### 6. Dual Authentication Systems with No Rate Limiting — `lib/controller/base_controller.rb`

Two independent authentication mechanisms coexist:

1. The `before` block: a custom DynamoDB bearer-token lookup for all paths not in `excluded_paths`
2. The `auth_token_has_all` condition: OAuth scope checking via `epb-auth-tools`

The `excluded_paths` list bypasses the DynamoDB check entirely for unauthenticated read endpoints. There is **no rate limiting** on these open endpoints, making them trivially DoS-able.

**Fix:** Add `rack-attack` middleware with per-IP throttling for all endpoints, including unauthenticated ones.

---

### 7. SQL String Interpolation in Materialized View Refresh — `lib/gateway/materialized_views_gateway.rb` (lines 16–17)

```ruby
str = concurrently ? "CONCURRENTLY" : ""
sql = "REFRESH MATERIALIZED VIEW #{str} #{name}"
```

`name` is validated against `pg_matviews`, which mitigates the risk. However, `name` still goes into the SQL string unquoted. If the validation logic is ever bypassed, this becomes injectable.

**Fix:** Quote the view name using `connection.quote_table_name`:

```ruby
sql = "REFRESH MATERIALIZED VIEW #{str} #{ActiveRecord::Base.connection.quote_table_name(name)}"
```

---

### 8. Silent Stub Clients in Non-Production Environments — `lib/gateway/s3_gateway.rb`, `lib/gateway/user_credentials_gateway.rb`

```ruby
case ENV["APP_ENV"]
when "production"
  Aws::S3::Client.new(region: "eu-west-2", credentials: Aws::ECSCredentials.new)
else
  Aws::S3::Client.new(stub_responses: true)
end
```

Any environment where `APP_ENV` is not exactly `"production"` (e.g. `staging`, `integration`) silently uses stubbed AWS clients, meaning actual S3/DynamoDB calls are no-ops and authentication stops working without any error or warning.

**Fix:** Use an explicit allowlist — only stub for `"test"` or `"local"`, raise a `ConfigurationError` for any other unrecognised value:

```ruby
case ENV["APP_ENV"]
when "production", "staging", "integration"
  Aws::S3::Client.new(region: "eu-west-2", credentials: Aws::ECSCredentials.new)
when "test", "local", nil
  Aws::S3::Client.new(stub_responses: true)
else
  raise Errors::ConfigurationError, "Unrecognised APP_ENV: #{ENV['APP_ENV']}"
end
```

---

## Medium — Security

### 9. Error Detail Exposure via Exception Messages — `lib/errors.rb`

```ruby
def initialize(rrn)
  super("The assessment #{rrn} could not be found on the API.")
end
```

The RRN (certificate reference) is embedded in exception messages. Several controllers pass `e.message` directly to responses, leaking internal identifiers. See finding #3.

---

### 10. No Request Rate Limiting

There is no `Rack::Attack` or equivalent middleware. The unauthenticated count/statistics endpoints (`/api/domestic/count`, `/api/avg-co2-emissions`, etc.) are fully open to scraping and denial-of-service traffic.

---

### 11. Weak Assessment ID Pseudonymisation — `lib/helper/hashed_assessment_id.rb`

```ruby
def self.hash_rrn(rrn)
  rrn_array = rrn.split("-")
  rrn_array.unshift(rrn_array.last)
  rrn_array << rrn_array[1]
  Digest::SHA256.hexdigest rrn_array.join("-")
end
```

SHA-256 of a deterministically rearranged RRN is not a one-way function in practice — RRNs follow a known format, making the output pre-computable via lookup table. If the hash is used to pseudonymise PII, it can be reversed trivially.

**Fix:** Use HMAC-SHA256 with a server-side secret key, or a purpose-built pseudonymisation key:

```ruby
def self.hash_rrn(rrn)
  OpenSSL::HMAC.hexdigest("SHA256", ENV.fetch("RRN_HASH_SECRET"), rrn)
end
```

---

## Critical — Performance

### 12. DynamoDB Full-Table Scan on Every Authenticated HTTP Request

(See Security finding #2.) Every API call that is not on `excluded_paths` triggers a full DynamoDB table scan. Under any meaningful traffic this will become the primary throttle point and will balloon DynamoDB read costs unboundedly as the token table grows.

---

### 13. Connection Pool Destroyed After Every HTTP Response — `lib/controller/base_controller.rb` (line 65)

```ruby
ActiveRecord::Base.connection_handler.clear_active_connections!(:all)
```

This **returns all connections to the pool** after every single request, completely defeating connection pooling. In Puma's threaded model (`threads 2, 4`), every thread must re-establish a new DB connection on every request — typically adding 50–200ms of latency each time. Under load this causes connection exhaustion and compounding latency spikes.

**Fix:** Remove this call entirely. ActiveRecord manages connection checkout and checkin automatically per-thread in a multi-threaded server.

---

## High — Performance

### 14. ONS Lookup Data Fetched from DB on Every Search Request — `lib/helper/search_params.rb`

```ruby
lower_councils = Container.ons_gateway.councils.map { |c| c[:lower_name] }
lower_constituencies = Container.ons_gateway.constituencies.map { |c| c[:lower_name] }
```

These hit the database for the full council/constituency list on every search request that includes those parameters. The ONS data is static (updated quarterly at most) and should be cached in memory, only refreshed on deploy.

---

### 15. Subprocess Spawned for Every XML Parse — `lib/use_case/parse_xml_certificate.rb` (line 16)

```ruby
Parallel.map([0]) { |_|
  parse.call
}.first
```

`Parallel.map` with a single-element array forks a subprocess for **every certificate** imported. The fork/spawn overhead far exceeds the XML parse time for most documents and provides no concurrency benefit.

**Fix:** Call `parse.call` directly, or always pass `use_subprocess: false` unless a documented memory-isolation requirement exists.

---

### 16. Default Page Size of 5,000 Records — `lib/controller/api_base_controller.rb`

```ruby
params[:page_size] = (params["page_size"] || 5000).to_i
```

Requests that do not specify a page size default to returning 5,000 rows. On large tables this can produce multi-megabyte responses and saturate DB I/O on every unparameterised call.

---

### 17. `logger.error` Used for All Timing Measurements — `lib/helper/stopwatch.rb` (line 16)

```ruby
logger.error "#{message} in #{stopwatch.elapsed_time}s"
```

Every timing measurement — even for successful, normal operations — is logged at `ERROR` severity. This pollutes Sentry and CloudWatch with noise, causing alert fatigue and masking real errors.

**Fix:** Use `logger.info` or `logger.debug` for routine timing. Reserve `logger.error` for actual failures.

---

### 18. S3 Presigned URL Expiry of 30 Seconds — `lib/use_case/get_presigned_url.rb` (line 10)

```ruby
def execute(file_name:, expires_in: 30)
```

A 30-second expiry is too short for redirects to large ZIP files. Slow clients or transient S3 latency will result in `403 Request has expired` errors for legitimate users.

**Fix:** Increase the default to 300–900 seconds.

---

### 19. No DB Connection Pool Tuning to Match Puma Threads — `config/puma.rb`

`config/puma.rb` sets `threads 2, 4` (up to 4 threads per worker), but there is no `pool:` override in the database connection configuration. If multiple Puma workers are spawned (e.g. via `WEB_CONCURRENCY`), each opens its own default pool of 5 connections, potentially overwhelming the database with connections.

**Fix:** Set the ActiveRecord pool size to match the maximum thread count, e.g. in `database.yml` or `DATABASE_URL`: `pool=<%= ENV.fetch("RAILS_MAX_THREADS", 4) %>`.

---

## Summary Table

| # | Severity | Category | Location |
|---|----------|----------|----------|
| 1 | Critical | SQL Injection | `lib/helper/generate_json_samples.rb:38–41` |
| 2 | Critical | Auth DoS / O(n) Scan | `lib/gateway/user_credentials_gateway.rb:13` |
| 3 | Critical | Info Disclosure via Exceptions | Multiple controllers |
| 4 | Critical | Bearer Tokens in Plaintext | DynamoDB table |
| 5 | High | Misconfigured Security Headers | `lib/middleware/headers_policy.rb` |
| 6 | High | No Rate Limiting | Entire application |
| 7 | High | SQL Interpolation (view name) | `lib/gateway/materialized_views_gateway.rb:17` |
| 8 | High | Silent Stub Clients in Non-Prod | `lib/gateway/s3_gateway.rb`, `user_credentials_gateway.rb` |
| 9 | Medium | Error Detail Exposure | `lib/errors.rb` + controllers |
| 10 | Medium | No Rate Limiting (open endpoints) | Entire application |
| 11 | Medium | Weak Pseudonymisation | `lib/helper/hashed_assessment_id.rb` |
| 12 | Critical | DynamoDB Scan per Request | `lib/gateway/user_credentials_gateway.rb` |
| 13 | Critical | Connection Pool Destruction | `lib/controller/base_controller.rb:65` |
| 14 | High | Hot-Path DB Queries (ONS data) | `lib/helper/search_params.rb` |
| 15 | High | Subprocess per XML Import | `lib/use_case/parse_xml_certificate.rb:16` |
| 16 | High | Unbounded Default Page Size | `lib/controller/api_base_controller.rb` |
| 17 | Medium | Wrong Log Level for Timing | `lib/helper/stopwatch.rb:16` |
| 18 | Medium | S3 URL Expiry Too Short | `lib/use_case/get_presigned_url.rb:10` |
| 19 | Medium | DB Pool / Thread Mismatch | `config/puma.rb` + ActiveRecord config |

---

## Recommended Priority Order

1. Fix the **SQL injection** in `lib/helper/generate_json_samples.rb` — the only location not using parameterised bindings.
2. Replace the **DynamoDB `scan` with `get_item`** — critical for both security and performance; degrades linearly with every new token.
3. **Remove `clear_active_connections!(:all)`** from the response path — it actively destroys the connection pool on every request.
4. **Raise the HSTS `max-age`** from 300 to `31536000` and add `X-Content-Type-Options: nosniff`.
5. **Stop forwarding `e.message` to HTTP clients** in 500 responses — log via Sentry, return generic strings to callers.
