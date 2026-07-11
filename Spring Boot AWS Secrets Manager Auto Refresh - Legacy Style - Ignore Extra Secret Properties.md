# Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style (Ignore Extra Secret Properties)

## Goal

When AWS Secrets Manager rotates passwords for PostgreSQL RDS or OpenSearch, the Spring Boot application should fetch the latest secret and continue working without restarting the container.

This version also ignores extra JSON fields in Secrets Manager that are not used for RDS or OpenSearch connections.

This legacy style avoids using `java.util.function.Supplier`.

---

## Part 1: PostgreSQL RDS

### Step 1: Secret format in AWS Secrets Manager

Secret name:

```text
prod/postgres/app
```

Secret value:

```json
{
    "host": "your-rds-endpoint.amazonaws.com",
    "port": 5432,
    "dbname": "appdb",
    "username": "app_user",
    "password": "current-password",
    "rotation_note": "safe to ignore",
    "owner_email": "dba@example.com"
}
```

### Step 2: Create Java model

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

### Step 3: Create refresh service

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

    public PostgresSecretRefreshService(
            HikariDataSource dataSource,
            SecretsManagerClient secretsClient
    ) {
        this.dataSource = dataSource;
        this.secretsClient = secretsClient;
    }

    public synchronized void refreshCredentials() {
        try {
            String secretJson = secretsClient.getSecretValue(
                    GetSecretValueRequest.builder()
                            .secretId("prod/postgres/app")
                            .build()
            ).secretString();

            PostgresSecret secret = objectMapper.readValue(secretJson, PostgresSecret.class);

            dataSource.setUsername(secret.getUsername());
            dataSource.setPassword(secret.getPassword());

            dataSource.getHikariPoolMXBean().softEvictConnections();

        } catch (Exception e) {
            throw new RuntimeException(
                    "Failed to refresh PostgreSQL credentials from AWS Secrets Manager",
                    e
            );
        }
    }
}
```

### Step 4: Create legacy connection provider

```java
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

@Component
public class RefreshableConnectionProvider {

    private final DataSource dataSource;
    private final PostgresSecretRefreshService refreshService;

    public RefreshableConnectionProvider(
            DataSource dataSource,
            PostgresSecretRefreshService refreshService
    ) {
        this.dataSource = dataSource;
        this.refreshService = refreshService;
    }

    public Connection getConnection() throws SQLException {
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

### Step 5: Replace existing JDBC code

Before:

```java
try (Connection conn = dataSource.getConnection()) {
    // existing JDBC logic
}
```

After:

```java
try (Connection conn = refreshableConnectionProvider.getConnection()) {
    // existing JDBC logic
}
```

### PostgreSQL flow

```text
Application calls getConnection()
Old password fails
PostgreSQL returns SQLSTATE 28P01
Application fetches new secret from AWS Secrets Manager
Application updates Hikari username/password
Application evicts old pool connections
Application retries getConnection()
Connection works without restart
```

---

## Part 2: OpenSearch

### Step 1: Secret format in AWS Secrets Manager

Secret name:

```text
prod/opensearch/app
```

Secret value:

```json
{
    "host": "https://your-opensearch-domain.us-east-1.es.amazonaws.com",
    "username": "admin",
    "password": "current-password",
    "team": "search-platform",
    "expires_hint": "2026-12-31"
}
```

### Step 2: Create Java model

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

### Step 3: Create refreshable OpenSearch client provider

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

@Component
public class OpenSearchClientProvider {

    private final SecretsManagerClient secretsClient;
        private final ObjectMapper objectMapper = new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    private volatile OpenSearchClient client;

    public OpenSearchClientProvider(SecretsManagerClient secretsClient) {
        this.secretsClient = secretsClient;
        this.client = buildClientFromSecret();
    }

    public OpenSearchClient getClient() {
        return client;
    }

    public synchronized void refreshClient() {
        this.client = buildClientFromSecret();
    }

    private OpenSearchClient buildClientFromSecret() {
        try {
            String secretJson = secretsClient.getSecretValue(
                    GetSecretValueRequest.builder()
                            .secretId("prod/opensearch/app")
                            .build()
            ).secretString();

            OpenSearchSecret secret = objectMapper.readValue(secretJson, OpenSearchSecret.class);

            BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
            credentialsProvider.setCredentials(
                    AuthScope.ANY,
                    new UsernamePasswordCredentials(
                            secret.getUsername(),
                            secret.getPassword().toCharArray()
                    )
            );

            RestClient restClient = RestClient.builder(
                            HttpHost.create(secret.getHost())
                    )
                    .setHttpClientConfigCallback(httpClientBuilder ->
                            httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider)
                    )
                    .build();

            OpenSearchTransport transport =
                    new RestClientTransport(restClient, new JacksonJsonpMapper());

            return new OpenSearchClient(transport);

        } catch (Exception e) {
            throw new RuntimeException(
                    "Failed to build OpenSearch client from AWS Secrets Manager",
                    e
            );
        }
    }
}
```

### Step 4: Create legacy OpenSearch service method

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

### Step 5: Replace existing OpenSearch usage

Before:

```java
SearchResponse<MyDocument> response =
        openSearchClient.search(request, MyDocument.class);
```

After:

```java
SearchResponse<MyDocument> response =
        openSearchSearchService.search(request);
```

### OpenSearch flow

```text
Application calls OpenSearch
Old password fails
OpenSearch returns 401 or 403
Application fetches new secret from AWS Secrets Manager
Application rebuilds OpenSearch client
Application retries request once
Request works without restart
```

---

## Important Rules

### Ignore extra secret properties

If the secret contains extra keys that your connection code does not use, deserialization should still succeed.

Use one or both safeguards:

- Annotate models with `@JsonIgnoreProperties(ignoreUnknown = true)`.
- Configure `ObjectMapper` with `FAIL_ON_UNKNOWN_PROPERTIES = false`.

This keeps refresh logic resilient when secret payloads evolve with metadata fields.

### Retry only once

Do not retry forever.

Good:

```text
try -> fail -> refresh secret -> retry once
```

Bad:

```text
while true -> retry forever
```

### Make refresh methods synchronized

This avoids multiple threads refreshing at the same time.

```java
public synchronized void refreshCredentials() {
    // refresh logic
}

public synchronized void refreshClient() {
    // refresh logic
}
```

---

## Final Summary

### PostgreSQL

Replace this:

```java
dataSource.getConnection()
```

With this:

```java
refreshableConnectionProvider.getConnection()
```

### OpenSearch

Replace direct client usage:

```java
openSearchClient.search(...)
```

With a service method:

```java
openSearchSearchService.search(...)
```

This allows the application to fetch rotated AWS Secrets Manager credentials and continue working without restarting the Spring Boot container.
