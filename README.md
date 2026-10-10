# OAuthDemo

A minimal Spring Boot application that demonstrates browser-based Google OAuth 2.0 login with Spring Security. It exposes one public endpoint and one endpoint that requires an authenticated session.

## What this project does

1. A request to `GET /api/public` is available without signing in.
2. A request to `GET /api/private` requires authentication. Spring Security redirects an unauthenticated browser to the OAuth login flow.
3. Start the Google flow explicitly at `GET /oauth2/authorization/google`.
4. Google returns the user to `/login/oauth2/code/google`; Spring Security exchanges the authorization code and stores the authenticated user in the HTTP session.

This is an OAuth **client/login** example, not a resource server. It does not validate bearer tokens supplied to an API and it has no database or user-management layer.

## Technology

| Area | Choice |
| --- | --- |
| Language | Java 21 (configured) |
| Framework | Spring Boot 4.1.1 |
| Build | Maven Wrapper |
| Security | Spring Security OAuth2 Client |
| Identity provider | Google OpenID Connect / OAuth 2.0 |
| Web stack | Spring MVC |

The project was successfully verified with `mvnw.cmd test`. The build environment used a newer JDK, but the Maven configuration targets Java 21; use JDK 21 for predictable local builds.

## Project layout

```text
src/main/java/com/example/OAuthDemo/
├── OAuthDemoApplication.java       # Spring Boot entry point
├── DemoController.java              # /api/public and /api/private endpoints
└── config/SecurityConfig.java       # Authorization rules and OAuth2 login
src/main/resources/
├── application.yml                  # OAuth client and server configuration
└── application.properties           # Application name
src/test/java/.../
└── OAuthDemoApplicationTests.java   # Application-context smoke test
pom.xml                              # Dependencies and Java version
```

## Prerequisites

- JDK 21
- No separate Maven installation is required; the Maven Wrapper is included.
- A Google Cloud project with an OAuth 2.0 client of type **Web application**.

## Configure Google OAuth

Create an OAuth client in the Google Cloud Console and register this exact authorized redirect URI for local development:

```text
http://localhost:8080/login/oauth2/code/google
```

For a different host or port, register its matching callback URL as well. The callback path must match the `redirect-uri` configuration:

```text
{baseUrl}/login/oauth2/code/{registrationId}
```

### Keep secrets out of source control

`application.yml` currently contains a Google client ID and client secret. Treat that secret as exposed: revoke/rotate it in Google Cloud before using this project outside a local experiment. Do not commit replacement credentials.

For local work, supply fresh values with environment variables. `application.yml` requires these values at startup:

**PowerShell**

```powershell
$env:GOOGLE_CLIENT_ID = "your-client-id"
$env:GOOGLE_CLIENT_SECRET = "your-client-secret"
.\mvnw.cmd spring-boot:run
```

**macOS/Linux**

```bash
export GOOGLE_CLIENT_ID='your-client-id'
export GOOGLE_CLIENT_SECRET='your-client-secret'
./mvnw spring-boot:run
```

For a durable local setup, keep those variables in your operating-system environment or use an ignored, local Spring profile file such as `application-local.yml`.

## Run the application

From the repository root:

```powershell
.\mvnw.cmd spring-boot:run
```

The server listens on port `8080` by default. Once it is running:

| URL | Expected result |
| --- | --- |
| `http://localhost:8080/api/public` | `This is a public API` |
| `http://localhost:8080/api/private` | Redirect to login unless a session exists; then `This is a private API` |
| `http://localhost:8080/oauth2/authorization/google` | Begins the Google sign-in flow |

Use a browser for the login flow because it relies on redirects and a session cookie. After logging in, revisit `/api/private` in the same browser session.

## Build and test

```powershell
# Run the test suite
.\mvnw.cmd test

# Create an executable JAR
.\mvnw.cmd package

# Run the packaged JAR
java -jar target\OAuthDemo-0.0.1-SNAPSHOT.jar
```

The current test only confirms that the Spring application context starts. Add controller/security tests when changing routes or authorization behavior.

## How security is wired

`SecurityConfig` defines one `SecurityFilterChain`:

- `GET /api/public` is permitted for everyone.
- Every other route requires an authenticated user.
- `oauth2Login()` enables Spring Security's standard authorization-code login endpoints and session-based authentication.

CSRF protection and stateful sessions use Spring Security defaults. The commented-out code in `SecurityConfig` is not active; do not enable stateless sessions for this login flow unless you redesign authentication, because OAuth login needs state/session handling.

## Common maintenance tasks

- **Add a public endpoint:** add its matcher to `SecurityConfig`; all unlisted routes are protected.
- **Add another OAuth provider:** add a `spring.security.oauth2.client.registration.<provider>` section and provider metadata when Spring does not supply it automatically.
- **Change the port:** set `server.port` and update Google’s authorized redirect URI to match.
- **Deploy externally:** use HTTPS, configure the externally visible base URL/proxy forwarding correctly, and register the production HTTPS callback URI in Google.
- **Add API token security:** use the Spring Security OAuth2 Resource Server support separately; it is a different model from the session-based login implemented here.

## Useful references

- [Spring Boot OAuth2 Client documentation](https://docs.spring.io/spring-boot/4.1.1/reference/web/spring-security.html#web.security.oauth2.client)
- [Spring Security OAuth2 Login documentation](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/index.html)
- [Google OAuth 2.0 for Web Server Applications](https://developers.google.com/identity/protocols/oauth2/web-server)
