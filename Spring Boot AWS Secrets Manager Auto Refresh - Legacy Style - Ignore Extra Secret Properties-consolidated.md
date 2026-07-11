# Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style (Ignore Extra Secret Properties) - Consolidated

## Version Relationship

This consolidated version merges the practical implementation depth of v1 and the production hardening guidance of v2.

- Includes all base improvements from v1.
- Includes all operational controls introduced in v2.

## Scope

Legacy style document for Spring Boot + AWS Secrets Manager credential rotation where extra secret JSON fields are ignored safely.

---

## Core Improvements Included

- Unknown-field safe deserialization with Jackson.
- AWSCURRENT version-stage reads.
- Required-field validation before applying refreshed credentials.
- Retry once on auth failures.
- Safe OpenSearch client swap and old RestClient cleanup.
- Last-known-good behavior guidance.
- Concurrency/refresh-storm control guidance.
- Operational validation checklist and suggested tests.

---

## PostgreSQL RDS

### Secret model

```java
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class PostgresSecret {
    private String host;
    private int port;
    private String dbname;
    private String username;
    private String password;

    // getters and setters
}
```

### Refresh service

```java
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;

@Service
public class PostgresSecretRefreshService {

    private final HikariDataSource dataSource;
    private final SecretsManagerClient secretsClient;
    private final ObjectMapper objectMapper = new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    public PostgresSecretRefreshService(HikariDataSource dataSource, SecretsManagerClient secretsClient) {
        this.dataSource = dataSource;
        this.secretsClient = secretsClient;
    }

    public synchronized void refreshCredentials() {
        try {
            String secretJson = secretsClient.getSecretValue(
                    GetSecretValueRequest.builder()
                            .secretId("prod/postgres/app")
                            .versionStage("AWSCURRENT")
                            .build()
            ).secretString();

            PostgresSecret secret = objectMapper.readValue(secretJson, PostgresSecret.class);
            requireValue(secret.getUsername(), "username");
            requireValue(secret.getPassword(), "password");

            dataSource.setUsername(secret.getUsername());
            dataSource.setPassword(secret.getPassword());
            dataSource.getHikariPoolMXBean().softEvictConnections();

        } catch (Exception e) {
            throw new RuntimeException("Failed to refresh PostgreSQL credentials", e);
        }
    }

    private void requireValue(String value, String field) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("Missing required field: " + field);
        }
    }
}
```

### Legacy connection provider

```java
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

@Component
public class RefreshableConnectionProvider {

    private final DataSource dataSource;
    private final PostgresSecretRefreshService refreshService;

    public RefreshableConnectionProvider(DataSource dataSource, PostgresSecretRefreshService refreshService) {
        this.dataSource = dataSource;
        this.refreshService = refreshService;
    }

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
}
```

---

## OpenSearch

### Secret model

```java
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class OpenSearchSecret {
    private String host;
    private String username;
    private String password;

    // getters and setters
}
```

### Provider with safe refresh swap

```java
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.hc.client5.http.auth.AuthScope;
import org.apache.hc.client5.http.auth.UsernamePasswordCredentials;
import org.apache.hc.client5.http.impl.auth.BasicCredentialsProvider;
import org.apache.http.HttpHost;
import org.opensearch.client.RestClient;
import org.opensearch.client.json.jackson.JacksonJsonpMapper;
import org.opensearch.client.opensearch.OpenSearchClient;
import org.opensearch.client.transport.OpenSearchTransport;
import org.opensearch.client.transport.rest_client.RestClientTransport;
import org.springframework.stereotype.Component;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;

import java.io.IOException;

@Component
public class OpenSearchClientProvider {

    private final SecretsManagerClient secretsClient;
    private final ObjectMapper objectMapper = new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    private volatile OpenSearchClient client;
    private volatile RestClient restClient;

    public OpenSearchClientProvider(SecretsManagerClient secretsClient) {
        this.secretsClient = secretsClient;
        refreshClient();
    }

    public OpenSearchClient getClient() {
        return client;
    }

    public synchronized void refreshClient() {
        RestClient oldRestClient = this.restClient;

        OpenSearchSecret secret = loadSecret();
        requireValue(secret.getHost(), "host");
        requireValue(secret.getUsername(), "username");
        requireValue(secret.getPassword(), "password");

        BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(
                AuthScope.ANY,
                new UsernamePasswordCredentials(secret.getUsername(), secret.getPassword().toCharArray())
        );

        RestClient newRestClient = RestClient.builder(HttpHost.create(secret.getHost()))
                .setHttpClientConfigCallback(httpClientBuilder ->
                        httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider)
                )
                .build();

        OpenSearchTransport transport = new RestClientTransport(newRestClient, new JacksonJsonpMapper());
        this.client = new OpenSearchClient(transport);
        this.restClient = newRestClient;

        if (oldRestClient != null) {
            try {
                oldRestClient.close();
            } catch (IOException ignored) {
                // best effort
            }
        }
    }

    private OpenSearchSecret loadSecret() {
        try {
            String secretJson = secretsClient.getSecretValue(
                    GetSecretValueRequest.builder()
                            .secretId("prod/opensearch/app")
                            .versionStage("AWSCURRENT")
                            .build()
            ).secretString();
            return objectMapper.readValue(secretJson, OpenSearchSecret.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to load OpenSearch secret", e);
        }
    }

    private void requireValue(String value, String field) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("Missing required field: " + field);
        }
    }
}
```

### Legacy OpenSearch service

```java
import org.opensearch.client.opensearch._types.OpenSearchException;
import org.opensearch.client.opensearch.core.SearchRequest;
import org.opensearch.client.opensearch.core.SearchResponse;
import org.springframework.stereotype.Service;

import java.io.IOException;

@Service
public class OpenSearchSearchService {

    private final OpenSearchClientProvider clientProvider;

    public OpenSearchSearchService(OpenSearchClientProvider clientProvider) {
        this.clientProvider = clientProvider;
    }

    public SearchResponse<MyDocument> search(SearchRequest request) throws IOException {
        try {
            return clientProvider.getClient().search(request, MyDocument.class);
        } catch (OpenSearchException ex) {
            if (ex.status() == 401 || ex.status() == 403) {
                clientProvider.refreshClient();
                return clientProvider.getClient().search(request, MyDocument.class);
            }
            throw ex;
        }
    }
}
```

---

## Operational Controls

- Keep last-known-good client/credentials if refresh fails.
- Prevent refresh storms with synchronized refresh and optional cooldown logic.
- Emit metrics for refresh attempts, failures, and retry outcomes.
- Ensure all DB/OpenSearch calls route through refresh wrappers.

---

## Feasibility Validation

### Technical feasibility: High

- Works with Spring Boot, AWS SDK v2, Hikari, and OpenSearch Java client.
- Supports secrets with additional metadata fields.

### Operational feasibility: Medium to High

- Reliable in long-running services when wrapper usage is consistent.
- Requires IAM/network reliability and observability maturity.

### Residual risks

- One request/connection may fail at rotation boundary before retry recovers.
- Active DB sessions can fail before pool turnover completes.
- Temporary Secrets Manager outage can delay refresh.

---

## Suggested Tests

1. Rotation success and auto-recovery test.
2. Extra-field tolerance test.
3. Corrupt-secret test with rollback-safe behavior.
4. Concurrent failure test.
5. Secrets Manager outage resilience test.
