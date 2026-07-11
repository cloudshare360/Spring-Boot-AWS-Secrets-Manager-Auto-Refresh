# Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation

## Goal

When AWS Secrets Manager rotates PostgreSQL RDS or OpenSearch passwords, the Spring Boot application should fetch the latest secret and continue working without restarting the container.

## Key Idea

Do not rely only on startup-loaded properties. Add logic around connection/client creation:

1. Try normal connection.
2. If authentication fails, fetch the latest secret from AWS Secrets Manager.
3. Update the datasource/client.
4. Retry once.

AWS Secrets Manager provides `GetSecretValue` to retrieve the current secret value. AWS recommends caching secrets client-side for better performance and cost.

---

## Part 1: PostgreSQL RDS

### Step 1: Store PostgreSQL secret in AWS Secrets Manager

Example secret name:

```text
prod/postgres/app
```

Example secret value:

```json
{
  "host": "your-rds-endpoint.amazonaws.com",
  "port": 5432,
  "dbname": "appdb",
  "username": "app_user",
  "password": "current-password"
}
```

### Step 2: Make sure your app has IAM permission

The app’s IAM role needs:

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:region:account-id:secret:prod/postgres/app-*"
}
```

### Step 3: Create secret model

```java
public class PostgresSecret {
    private String host;
    private int port;
    private String dbname;
    private String username;
    private String password;

    // getters and setters
}
```

### Step 4: Create PostgreSQL refresh service

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;

@Service
public class PostgresSecretRefreshService {

    private final HikariDataSource dataSource;
    private final SecretsManagerClient secretsClient;
    private final ObjectMapper objectMapper = new ObjectMapper();

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
            throw new RuntimeException("Failed to refresh PostgreSQL credentials from Secrets Manager", e);
        }
    }
}
```

`softEvictConnections()` evicts idle Hikari connections and marks active connections for eviction when they return to the pool.

### Step 5: Wrap `DataSource.getConnection()`

Since your current application directly calls:

```java
dataSource.getConnection();
```

Replace it with:

```java
getConnectionWithSecretRefresh();
```

Implementation:

```java
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

    public Connection getConnectionWithSecretRefresh() throws SQLException {
        try {
            return dataSource.getConnection();
        } catch (SQLException ex) {
            if (isPostgresAuthFailure(ex)) {
                refreshService.refreshCredentials();
                return dataSource.getConnection(); // retry once
            }

            throw ex;
        }
    }

    private boolean isPostgresAuthFailure(SQLException ex) {
        return "28P01".equals(ex.getSQLState());
    }
}
```

`28P01` is the PostgreSQL SQLSTATE for invalid password/authentication failure.

### Step 6: Replace old usage

Before:

```java
try (Connection conn = dataSource.getConnection()) {
    // database logic
}
```

After:

```java
try (Connection conn = refreshableConnectionProvider.getConnectionWithSecretRefresh()) {
    // database logic
}
```

### PostgreSQL behavior after rotation

Flow:

```text
Password rotates in AWS Secrets Manager
↓
Application tries to create new DB connection
↓
PostgreSQL returns authentication failure
↓
Application catches SQLSTATE 28P01
↓
Application fetches latest secret
↓
Application updates Hikari username/password
↓
Application evicts old pool connections
↓
Application retries connection
↓
New connection works without container restart
```

---

## Part 2: OpenSearch

### Step 1: Store OpenSearch secret in AWS Secrets Manager

Example secret name:

```text
prod/opensearch/app
```

Example secret value:

```json
{
  "host": "https://your-opensearch-domain.us-east-1.es.amazonaws.com",
  "username": "admin",
  "password": "current-password"
}
```

### Step 2: Create secret model

```java
public class OpenSearchSecret {
    private String host;
    private String username;
    private String password;

    // getters and setters
}
```

### Step 3: Create refreshable OpenSearch client provider

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.hc.client5.http.auth.AuthScope;
import org.apache.hc.client5.http.auth.UsernamePasswordCredentials;
import org.apache.hc.client5.http.impl.auth.BasicCredentialsProvider;
import org.opensearch.client.opensearch.OpenSearchClient;
import org.opensearch.client.transport.OpenSearchTransport;
import org.opensearch.client.transport.rest_client.RestClientTransport;
import org.opensearch.client.json.jackson.JacksonJsonpMapper;
import org.apache.http.HttpHost;
import org.opensearch.client.RestClient;
import org.springframework.stereotype.Component;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;

@Component
public class OpenSearchClientProvider {

    private final SecretsManagerClient secretsClient;
    private final ObjectMapper objectMapper = new ObjectMapper();

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

            RestClient restClient = RestClient.builder(HttpHost.create(secret.getHost()))
                    .setHttpClientConfigCallback(httpClientBuilder ->
                            httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider)
                    )
                    .build();

            OpenSearchTransport transport =
                    new RestClientTransport(restClient, new JacksonJsonpMapper());

            return new OpenSearchClient(transport);

        } catch (Exception e) {
            throw new RuntimeException("Failed to build OpenSearch client from Secrets Manager", e);
        }
    }
}
```

OpenSearch Java clients support basic authentication credentials when creating the client.

### Step 4: Retry OpenSearch operation once on auth failure

```java
import org.opensearch.client.opensearch._types.OpenSearchException;
import org.springframework.stereotype.Service;

import java.util.function.Supplier;

@Service
public class OpenSearchExecutor {

    private final OpenSearchClientProvider clientProvider;

    public OpenSearchExecutor(OpenSearchClientProvider clientProvider) {
        this.clientProvider = clientProvider;
    }

    public <T> T execute(Supplier<T> operation) {
        try {
            return operation.get();
        } catch (OpenSearchException ex) {
            if (ex.status() == 401 || ex.status() == 403) {
                clientProvider.refreshClient();
                return operation.get(); // retry once
            }

            throw ex;
        }
    }
}
```

### Step 5: Use provider instead of fixed client

Before:

```java
openSearchClient.search(request, MyDocument.class);
```

After:

```java
openSearchExecutor.execute(() ->
    openSearchClientProvider.getClient().search(request, MyDocument.class)
);
```

---

## Important Notes

### 1. Retry only once

Do not keep retrying forever.

Bad:

```java
while (true) {
    retry();
}
```

Good:

```text
try once → refresh secret → retry once → fail if still broken
```

This avoids infinite loops during real outages.

### 2. Make refresh methods synchronized

This prevents multiple threads from refreshing credentials at the same time.

```java
public synchronized void refreshCredentials() {
    // refresh logic
}
```

### 3. Existing active PostgreSQL connections do not instantly change

After this code runs:

```java
dataSource.setUsername(newUsername);
dataSource.setPassword(newPassword);
dataSource.getHikariPoolMXBean().softEvictConnections();
```

New connections should use the new password. Existing active connections finish their current work and are replaced when returned to the pool.

### 4. This does not require container restart

The application can continue running because it updates the in-memory datasource/client.

---

## Final Recommended Flow

### PostgreSQL

```text
dataSource.getConnection()
↓
SQLException 28P01
↓
Fetch latest PostgreSQL secret from AWS Secrets Manager
↓
Update HikariDataSource username/password
↓
softEvictConnections()
↓
Retry dataSource.getConnection()
```

### OpenSearch

```text
OpenSearch request
↓
401 or 403 auth failure
↓
Fetch latest OpenSearch secret from AWS Secrets Manager
↓
Rebuild OpenSearch client
↓
Retry request once
```

---

## Best Minimal Change

For your current application, the most important change is replacing direct calls to:

```java
dataSource.getConnection()
```

with:

```java
refreshableConnectionProvider.getConnectionWithSecretRefresh()
```

That gives you automatic credential refresh without restarting the Spring Boot container.
