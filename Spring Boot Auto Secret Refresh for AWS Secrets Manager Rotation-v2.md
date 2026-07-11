# Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation - v2 (Production Blueprint)

## Version Relationship

v2 is cumulative.

- Includes all improvements from v1.
- Adds production-hardening blueprint items listed below.

## Why v2

This version hardens the approach for production by adding:

- Secret caching guidance to reduce Secrets Manager API calls.
- Strong refresh guards (single-flight refresh + validation before swap).
- Explicit rollback behavior (keep last known good credentials/client if refresh fails).
- Operational validation checklist and test plan.

## Cumulative Improvement Matrix

v1 baseline included in v2:

- Unknown-field safe deserialization.
- AWSCURRENT fetch behavior.
- Required-field validation before apply.
- OpenSearch old-client close during refresh.
- Retry-once refresh behavior.

v2 additional improvements:

- Last-known-good behavior guidance.
- Concurrency and refresh-storm control guidance.
- Production validation checklist and expanded test plan.

---

## Production Design

### 1. Refresh trigger

- PostgreSQL: trigger refresh on SQL auth failure (`SQLSTATE 28P01`) and retry once.
- OpenSearch: trigger refresh on `401/403` and retry once.

### 2. Secret read strategy

- Read with `versionStage("AWSCURRENT")`.
- Use client-side caching (AWS Secrets Manager caching component or local TTL cache).
- Keep timeout and retry policy for AWS SDK calls conservative.

### 3. Safe apply strategy

- Parse secret with unknown-field tolerance.
- Validate required fields before applying.
- Swap credentials/client only after complete new object is built.
- Keep old credentials/client if refresh fails.

### 4. Concurrency strategy

- Only one refresh in progress per secret at a time.
- Other threads wait for refresh completion or use current known good client.

---

## Minimal Reference Snippets

### Unknown-field tolerant model

```java
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class OpenSearchSecret {
    private String host;
    private String username;
    private String password;
}
```

### ObjectMapper guard

```java
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;

ObjectMapper mapper = new ObjectMapper()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

### Rollback-safe OpenSearch refresh

```java
public synchronized void refreshClient() {
    RestClient oldRestClient = this.restClient;

    OpenSearchSecret secret = loadAndValidate();
    RestClient newRestClient = buildRestClient(secret);
    OpenSearchClient newClient = new OpenSearchClient(
            new RestClientTransport(newRestClient, new JacksonJsonpMapper())
    );

    // swap only after successful build
    this.restClient = newRestClient;
    this.client = newClient;

    if (oldRestClient != null) {
        try {
            oldRestClient.close();
        } catch (IOException ignored) {
            // best effort
        }
    }
}
```

### PostgreSQL apply after validation

```java
public synchronized void refreshCredentials() {
    PostgresSecret secret = loadAndValidatePostgresSecret();

    dataSource.setUsername(secret.getUsername());
    dataSource.setPassword(secret.getPassword());
    dataSource.getHikariPoolMXBean().softEvictConnections();
}
```

---

## Feasibility Validation

### Technical feasibility: High

- Pattern is fully feasible for Java Spring Boot with AWS SDK v2.
- Hikari and OpenSearch Java client both support runtime credential refresh patterns.
- Unknown extra fields can be safely ignored via Jackson.

### Operational feasibility: Medium to High

- Requires disciplined usage: all DB/OpenSearch calls must route through wrapper/executor.
- Requires proper observability to avoid silent partial failures.
- Requires IAM and network reliability to Secrets Manager.

### Residual risks (must accept or mitigate)

- A small failure window still exists during active credential rotation.
- Existing in-use DB connections may fail before recycle.
- Secrets Manager or network outage can block refresh; app should retain last known good client.

---

## Validation Checklist

- [ ] Secret has required fields (`username`, `password`, `host` where applicable).
- [ ] Secret can include extra metadata fields without breaking parsing.
- [ ] `AWSCURRENT` version is retrieved.
- [ ] Retry count is exactly one after refresh.
- [ ] Old OpenSearch RestClient is closed after successful client swap.
- [ ] Metrics/logs capture refresh attempts, successes, failures, and retry outcomes.

---

## Recommended Tests

1. Rotation success test: rotate secret and verify auto-recovery with one retry.
2. Extra field test: add unrelated JSON fields and confirm no deserialization failure.
3. Bad secret test: remove required field and verify refresh fails safely without corrupting active client.
4. Concurrency test: force many parallel auth failures and verify only one refresh runs.
5. Secrets Manager outage test: simulate temporary failure and verify app remains on last known good credentials/client.
