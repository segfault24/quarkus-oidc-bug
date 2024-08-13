# OIDC named tenants fail to recover when unavailable at startup

### Description
When using `quarkus-oidc` with named tenants and `resolve-tenants-with-issuer`, if an OIDC server is not available when the application starts, requests with credentials issued by that OIDC server will never succeed.

This behavior prevents an application from self-healing and requires a manual restart to return to normal operation.

Variations attempted:
- single named tenant
- multiple named tenants
- discovery-enabled=false
- connection-delay=PT2M
- connection-retry-count=999

### Expected Behavior
Quarkus eventually reattempts to contact the OIDC server, and subsequent requests succeed with 200 Ok.

### Actual Behavior
Quarkus never reattempts to contact the OIDC server, and all requests fail with 401 Unauthorized forever.

### How to Reproduce
1. Start the application, and wait for it to become ready. It will produce warnings about the OIDC server not being available.
    ```shell script
    ./mvnw quarkus:dev
    ```
    ```
    2024-08-08 21:57:04,578 WARN  [io.qua.oid.run.OidcRecorder] (vert.x-eventloop-thread-1) Tenant 'keycloak-1': 'OIDC Server is not available'. OIDC server is not available yet, an attempt to connect will be made during the first request. Access to resources protected by this tenant may fail if OIDC server will not become available
    ...
    2024-08-08 21:57:04,647 INFO  [io.quarkus] (Quarkus Main Thread) quarkus-oidc-bug 1.0.0-SNAPSHOT on JVM (powered by Quarkus 3.13.1) started in 2.053s. Listening on: http://localhost:8080
    ```

2. In a second terminal, start the Keycloak container, and wait for it to become ready.
    ```shell script
    docker-compose up
    ```
    ```
    ...
    2024-08-08 21:57:37,093 INFO  [org.keycloak.services] (main) KC-SERVICES0032: Import finished successfully
    2024-08-08 21:57:37,228 INFO  [org.keycloak.services] (main) KC-SERVICES0009: Added user 'admin' to realm 'master'
    2024-08-08 21:57:37,295 INFO  [io.quarkus] (main) Keycloak 25.0.2 on JVM (powered by Quarkus 3.8.5) started in 7.546s. Listening on: http://0.0.0.0:8080. Management interface listening on http://0.0.0.0:9000.
    ```

3. In a third terminal, acquire an access token and make a request to the application. It will always respond with 401 Unauthorized.
    ```shell script
    ACCESS_TOKEN=$( \
        curl http://localhost:8081/realms/quarkus/protocol/openid-connect/token \
        -d "client_id=test-client" -d "username=alice" -d "password=alice" -d "grant_type=password" \
        | jq -r '.access_token')
    curl -v http://localhost:8080/hello -H "Authorization: bearer ${ACCESS_TOKEN}"
    ```

### Test Setup Validation
To validate that this test setup works under "normal" conditions, steps 1 & 2 can be swapped, in which case step 3 will succeed.
