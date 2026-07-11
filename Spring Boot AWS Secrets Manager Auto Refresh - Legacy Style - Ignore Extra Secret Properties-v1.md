# Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style (Ignore Extra Secret Properties) - v1

## Version Relationship

This is the first improved version.

- v1 introduces core hardening fixes over the base guide.
- v2 is intended to include all v1 improvements plus additional production controls.

## Scope

This version keeps the legacy style (no java.util.function.Supplier requirement in the application usage path), ignores unrelated secret properties, and fixes implementation risks in the earlier draft.

## Improvements in v1

- Fixes indentation issues in Java snippets.
- Uses AWSCURRENT when reading secrets.
- Validates required fields before applying refreshed values.
- Keeps unknown JSON fields safe with Jackson ignore-unknown settings.
- Closes old OpenSearch RestClient during refresh to avoid connection leaks.

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

## Feasibility

- Technical feasibility: High.
- Operational feasibility: Medium to high when all DB/OpenSearch calls use these wrappers.
- Known limitation: one failed request/connection can still happen during rotation before retry succeeds.
