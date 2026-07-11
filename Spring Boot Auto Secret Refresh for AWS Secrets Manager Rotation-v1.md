# Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation - v1

## Version Relationship

This is the first improved version.

- v1 introduces practical hardening fixes.
- v2 includes all v1 improvements plus additional production blueprint controls.

## What Is Improved in v1

- Adds safe handling for extra JSON fields in secrets.
- Fixes missing imports in code snippets.
- Adds required-field validation before applying new credentials.
- Uses `versionStage("AWSCURRENT")` for deterministic reads.
- Closes old OpenSearch `RestClient` to avoid connection leaks.

---

## Goal

When AWS Secrets Manager rotates PostgreSQL RDS or OpenSearch credentials, the Spring Boot app refreshes credentials in memory and retries once, without container restart.

---

## PostgreSQL

### Secret model (ignore unknown keys)

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
            validate(secret.getUsername(), "username");
            validate(secret.getPassword(), "password");

            dataSource.setUsername(secret.getUsername());
            dataSource.setPassword(secret.getPassword());
            dataSource.getHikariPoolMXBean().softEvictConnections();
        } catch (Exception e) {
            throw new RuntimeException("Failed to refresh PostgreSQL credentials", e);
        }
    }

    private void validate(String value, String field) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("Missing required field: " + field);
        }
    }
}
```

### Connection wrapper (retry once on auth failure)

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

    public Connection getConnectionWithSecretRefresh() throws SQLException {
        try {
            return dataSource.getConnection();
        } catch (SQLException ex) {
            if (isPostgresAuthFailure(ex)) {
                refreshService.refreshCredentials();
                return dataSource.getConnection();
            }
            throw ex;
        }
    }

    private boolean isPostgresAuthFailure(SQLException ex) {
        return "28P01".equals(ex.getSQLState());
    }
}
```

---

## OpenSearch

### Secret model (ignore unknown keys)

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

### Provider with leak-safe refresh

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
        RestClient oldClient = this.restClient;

        OpenSearchSecret secret = loadSecret();
        validate(secret.getHost(), "host");
        validate(secret.getUsername(), "username");
        validate(secret.getPassword(), "password");

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

        if (oldClient != null) {
            try {
                oldClient.close();
            } catch (IOException ignored) {
                // best effort cleanup
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

    private void validate(String value, String field) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("Missing required field: " + field);
        }
    }
}
```

### Execute wrapper (retry once)

```java
import org.opensearch.client.opensearch._types.OpenSearchException;
import org.springframework.stereotype.Service;

@Service
public class OpenSearchExecutor {

    private final OpenSearchClientProvider clientProvider;

    public OpenSearchExecutor(OpenSearchClientProvider clientProvider) {
        this.clientProvider = clientProvider;
    }

    public <T> T execute(java.util.function.Supplier<T> operation) {
        try {
            return operation.get();
        } catch (OpenSearchException ex) {
            if (ex.status() == 401 || ex.status() == 403) {
                clientProvider.refreshClient();
                return operation.get();
            }
            throw ex;
        }
    }
}
```

---

## Feasibility (v1)

- Works for long-running Spring Boot containers where credentials rotate in Secrets Manager.
- Works best when all DB/OpenSearch access paths use the refresh wrappers.
- Rotation gap is reduced to one failed request/connection attempt, then recovery.

Limits:

- In-flight requests can still fail during rotation window.
- Existing active DB connections are not instantly re-authenticated.
- This pattern is runtime refresh, not zero-failure failover.
