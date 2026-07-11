# Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style (Ignore Extra Secret Properties) - v2

## Version Relationship

v2 is cumulative.

- Includes all improvements from v1.
- Adds additional production-hardening controls.

## Purpose

This is the production-hardened legacy variant focused on safe rotation recovery, resilience to secret schema drift, and operational validation.

## What v2 Adds Over v1

- Explicit last-known-good behavior (do not replace working client with invalid refresh result).
- Single-refresh discipline and concurrency guidance.
- Observability requirements and validation matrix.
- Failure mode checklist for realistic operations.

## Cumulative Improvement Matrix

v1 baseline included in v2:

- Unknown-field safe deserialization.
- AWSCURRENT fetch behavior.
- Required-field validation before apply.
- OpenSearch old-client close during refresh.
- Retry-once refresh behavior for legacy wrappers.

v2 additional improvements:

- Last-known-good behavior guidance.
- Concurrency and refresh-storm control guidance.
- Observability requirements and operational validation checklist.

---

## Recommended Runtime Strategy

### 1. Secret parsing rules

- Use @JsonIgnoreProperties(ignoreUnknown = true) on secret models.
- Configure ObjectMapper with FAIL_ON_UNKNOWN_PROPERTIES false.
- Validate required keys before applying credentials.

### 2. Refresh trigger rules

- PostgreSQL: refresh on SQLSTATE 28P01, retry once.
- OpenSearch: refresh on 401 or 403, retry once.
- Never loop forever.

### 3. Safe swap rules

- Build and validate new OpenSearch client fully before replacing current one.
- If refresh fails, keep existing client and report error metrics.
- Close old RestClient only after successful swap.

### 4. Consistency rules

- Read AWSCURRENT version stage.
- Keep request timeout and retry policy bounded for Secrets Manager SDK calls.
- Ensure all call sites route through wrapper services.

---

## Legacy-Friendly Reference Skeleton

```java
public synchronized void refreshClient() {
    RestClient previous = this.restClient;

    OpenSearchSecret nextSecret = loadAndValidateSecret();
    RestClient nextRestClient = buildRestClient(nextSecret);
    OpenSearchClient nextClient = buildOpenSearchClient(nextRestClient);

    this.restClient = nextRestClient;
    this.client = nextClient;

    if (previous != null) {
        try {
            previous.close();
        } catch (IOException ignored) {
            // best effort cleanup
        }
    }
}
```

```java
public Connection getConnection() throws SQLException {
    try {
        return dataSource.getConnection();
    } catch (SQLException ex) {
        if ("28P01".equals(ex.getSQLState())) {
            refreshService.refreshCredentials();
            return dataSource.getConnection();
        }
        throw ex;
    }
}
```

---

## Feasibility Validation

### Technical feasibility: High

- Compatible with Spring Boot + AWS SDK v2 + Hikari + OpenSearch Java client.
- Extra secret fields are safely tolerated.

### Operational feasibility: Medium to High

- Reliable if wrappers are used consistently.
- Requires metrics, logs, and alarms for refresh outcomes.

### Residual risks

- In-flight failures can occur during rotation boundary.
- Temporary Secrets Manager/network outage can delay refresh.
- Active DB sessions may fail before pool eviction replaces them.

---

## Validation Checklist

- [ ] Secrets with additional metadata fields parse successfully.
- [ ] Required fields missing cause refresh abort (without corrupting active runtime client).
- [ ] AWSCURRENT is fetched.
- [ ] OpenSearch old RestClient is closed after successful swap.
- [ ] Exactly one retry occurs after auth failure.
- [ ] Concurrent auth failures do not trigger refresh storms.

---

## Suggested Tests

1. Rotation recovery test for PostgreSQL and OpenSearch.
2. Extra-field tolerance test (metadata keys added to secrets).
3. Corrupt-secret rollback test (missing password).
4. Concurrent failure test under load.
5. Secrets Manager unavailability test with fallback to last-known-good runtime state.
