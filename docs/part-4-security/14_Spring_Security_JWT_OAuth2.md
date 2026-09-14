# JWT & OAuth2 with Spring Security

> **File:** `14_Spring_Security_JWT_OAuth2.md`
> **Part:** 4 — Security
> **Prerequisites:** `13_Spring_Security_Core.md`
> **Estimated Study Time:** 10–14 hours

---

## Table of Contents

1. [JWT Fundamentals](#1-jwt-fundamentals)
2. [JWT Structure](#2-jwt-structure)
3. [JWT vs Session Tokens](#3-jwt-vs-session-tokens)
4. [Generating & Validating JWT](#4-generating--validating-jwt)
5. [JWT Authentication Filter](#5-jwt-authentication-filter)
6. [Refresh Tokens](#6-refresh-tokens)
7. [OAuth2 Concepts](#7-oauth2-concepts)
8. [OAuth2 Grant Types](#8-oauth2-grant-types)
9. [Authorization Code Flow](#9-authorization-code-flow)
10. [Client Credentials Flow](#10-client-credentials-flow)
11. [Spring as OAuth2 Resource Server](#11-spring-as-oauth2-resource-server)
12. [Spring Authorization Server](#12-spring-authorization-server)
13. [Keycloak Integration](#13-keycloak-integration)
14. [Social Login (Google, GitHub)](#14-social-login-google-github)
15. [Interview Questions](#15-interview-questions)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. JWT Fundamentals

**JWT** (JSON Web Token, RFC 7519) is a compact, self-contained token format for securely transmitting claims between parties.

### Why JWT?

- **Stateless** — server doesn't need to store sessions
- **Self-contained** — all user info in the token
- **Scalable** — works across distributed services
- **Standard** — libraries everywhere

### When to Use

- ✅ REST APIs with multiple services
- ✅ Mobile apps
- ✅ Single Sign-On (SSO)
- ✅ Microservice-to-microservice auth

### When NOT to Use

- ❌ Traditional server-rendered web apps (use sessions)
- ❌ Long-lived sessions without refresh
- ❌ Storing sensitive data (JWT payload is base64, not encrypted)

---

## 2. JWT Structure

Three parts, separated by dots: `header.payload.signature`

### Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Base64url encoded.

### Payload (Claims)

```json
{
  "sub": "1234567890",
  "name": "Alice",
  "iat": 1516239022,
  "exp": 1516242622,
  "iss": "https://auth.example.com",
  "aud": "my-api",
  "roles": ["USER", "ADMIN"]
}
```

### Claims Types

| Type | Examples |
|------|----------|
| **Registered** | `iss` (issuer), `sub` (subject), `aud` (audience), `exp` (expiration), `nbf` (not before), `iat` (issued at), `jti` (ID) |
| **Public** | Custom but registered (e.g., `email` from IANA) |
| **Private** | App-specific (`roles`, `tenantId`) |

### Signature

```
HMACSHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secret
)
```

Or RS256 with private key.

### Signing Algorithms

| Algorithm | Type | Use Case |
|-----------|------|----------|
| `HS256` | Symmetric (shared secret) | Single service |
| `RS256` | Asymmetric (public/private) | Multi-service, public verification |
| `ES256` | Asymmetric (ECDSA) | Efficient alternative to RS256 |
| `none` | No signature | ⚠️ Never use |

**Recommendation:** RS256 for anything beyond single service.

### Example JWT

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFsaWNlIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Where to Store

| Storage | Pros | Cons |
|---------|------|------|
| **localStorage** | Easy | XSS vulnerable |
| **sessionStorage** | Similar | Lost on tab close |
| **httpOnly cookie** | XSS-safe | CSRF concern (mitigate with SameSite) |
| **memory (JS)** | XSS-hard | Lost on refresh |

**Best practice:** `httpOnly` + `Secure` + `SameSite=Strict` cookie, or in-memory with short-lived tokens + refresh.

---

## 3. JWT vs Session Tokens

| Aspect | Session Token | JWT |
|--------|--------------|-----|
| State | Server-side session store | Self-contained |
| Scalability | Needs shared session store | Stateless |
| Revocation | Easy (delete session) | Hard (blacklist needed) |
| Size | Small (session ID) | Larger (all claims) |
| Expiry | Server controls | In token, immutable |
| Payload | Opaque | Readable (base64) |
| Cross-service | Requires shared store | Native |
| Security | Session hijacking | Token theft |

### When to Prefer Sessions

- Single-server web app
- Need immediate revocation
- Small user base

### When to Prefer JWT

- Stateless microservices
- Cross-domain SSO
- Mobile apps
- Multiple services verifying tokens

---

## 4. Generating & Validating JWT

### Library: jjwt

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

### Secret Key

```java
@Component
public class JwtProperties {
    @Value("${jwt.secret}")
    private String secret;   // ≥ 256 bits for HS256
    @Value("${jwt.expiration:3600000}")   // 1 hour ms
    private long expiration;
}
```

```properties
jwt.secret=your-256-bit-secret-key-for-HS256-please-change-in-production
jwt.expiration=3600000
```

### Generating

```java
@Component
public class JwtService {
    
    private final SecretKey key;
    private final long expirationMs;
    
    public JwtService(JwtProperties props) {
        this.key = Keys.hmacShaKeyFor(props.getSecret().getBytes(StandardCharsets.UTF_8));
        this.expirationMs = props.getExpiration();
    }
    
    public String generateToken(UserDetails user) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMs);
        
        return Jwts.builder()
            .subject(user.getUsername())
            .issuer("my-app")
            .issuedAt(now)
            .expiration(expiry)
            .claim("authorities", user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList())
            .signWith(key, Jwts.SIG.HS256)
            .compact();
    }
}
```

### Validating & Parsing

```java
public Claims parse(String token) {
    return Jwts.parser()
        .verifyWith(key)
        .requireIssuer("my-app")
        .build()
        .parseSignedClaims(token)
        .getPayload();
}

public String extractUsername(String token) {
    return parse(token).getSubject();
}

public boolean isValid(String token, UserDetails user) {
    try {
        Claims c = parse(token);
        return c.getSubject().equals(user.getUsername())
            && c.getExpiration().after(new Date());
    } catch (JwtException e) {
        return false;
    }
}
```

### Common Exceptions

| Exception | Cause |
|-----------|-------|
| `ExpiredJwtException` | `exp` in past |
| `SignatureException` | Wrong key / tampered |
| `MalformedJwtException` | Bad format |
| `UnsupportedJwtException` | Algorithm unsupported |
| `IllegalArgumentException` | Empty/null token |

### RS256 (Asymmetric)

```java
// Private key (server signs)
PrivateKey privateKey = readPrivateKey("private.pem");

String token = Jwts.builder()
    .subject(user.getUsername())
    .signWith(privateKey, Jwts.SIG.RS256)
    .compact();

// Public key (anyone verifies)
PublicKey publicKey = readPublicKey("public.pem");
Claims claims = Jwts.parser()
    .verifyWith(publicKey)
    .build()
    .parseSignedClaims(token)
    .getPayload();
```

### JWK Set (Key Rotation)

Public keys exposed as JWK Set at `/.well-known/jwks.json`. Clients fetch and cache. Enables key rotation without downtime.

---

## 5. JWT Authentication Filter

### The Filter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    public JwtAuthenticationFilter(JwtService jwtService, 
                                    UserDetailsService uds) {
        this.jwtService = jwtService;
        this.userDetailsService = uds;
    }
    
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String header = req.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(req, res);
            return;
        }
        
        String token = header.substring(7);
        
        try {
            String username = jwtService.extractUsername(token);
            if (username != null && SecurityContextHolder.getContext()
                    .getAuthentication() == null) {
                UserDetails user = userDetailsService.loadUserByUsername(username);
                if (jwtService.isValid(token, user)) {
                    UsernamePasswordAuthenticationToken auth = 
                        new UsernamePasswordAuthenticationToken(
                            user, null, user.getAuthorities());
                    auth.setDetails(new WebAuthenticationDetailsSource()
                        .buildDetails(req));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (JwtException e) {
            log.warn("Invalid JWT: {}", e.getMessage());
            // Continue unauthenticated → 401 via AuthorizationFilter
        }
        
        chain.doFilter(req, res);
    }
}
```

### Registration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtFilter;
    
    public SecurityConfig(JwtAuthenticationFilter jwtFilter) {
        this.jwtFilter = jwtFilter;
    }
    
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, 
                UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

### Auth Controller

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    private final AuthenticationManager authManager;
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    public AuthController(AuthenticationManager am, JwtService js, UserDetailsService uds) {
        this.authManager = am;
        this.jwtService = js;
        this.userDetailsService = uds;
    }
    
    @PostMapping("/login")
    public AuthResponse login(@RequestBody @Valid LoginRequest req) {
        Authentication auth = authManager.authenticate(
            new UsernamePasswordAuthenticationToken(req.username(), req.password()));
        
        UserDetails user = userDetailsService.loadUserByUsername(req.username());
        String token = jwtService.generateToken(user);
        
        return new AuthResponse(token, "Bearer", 3600L);
    }
}

record LoginRequest(@NotBlank String username, @NotBlank String password) { }
record AuthResponse(String accessToken, String tokenType, long expiresIn) { }
```

### Client Usage

```bash
# Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pwd"}'
# → {"accessToken":"eyJ...","tokenType":"Bearer","expiresIn":3600}

# Use token
curl -H "Authorization: Bearer eyJ..." http://localhost:8080/api/users
```

---

## 6. Refresh Tokens

### Why Refresh Tokens?

- Access tokens are short-lived (15 min) → stolen tokens expire quickly
- Refresh tokens are long-lived (7–30 days) → used to get new access tokens
- Refresh tokens can be revoked (stored in DB)

### Flow

```
1. Login → access + refresh tokens
2. Client uses access token (short TTL)
3. Access token expires → 401
4. Client sends refresh token to /api/auth/refresh
5. Server validates refresh token (DB), issues new access token (+ optional rotated refresh)
6. Client retries with new access token
```

### Entity

```java
@Entity
public class RefreshToken {
    @Id @GeneratedValue
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String token;
    
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
    
    private Instant expiresAt;
    private boolean revoked;
}
```

### Service

```java
@Service
public class RefreshTokenService {
    
    private final RefreshTokenRepository repo;
    private final UserRepository userRepo;
    private final long refreshTtlMs = 7 * 24 * 60 * 60 * 1000L;   // 7 days
    
    public RefreshToken create(String username) {
        User user = userRepo.findByUsername(username).orElseThrow();
        RefreshToken rt = new RefreshToken();
        rt.setToken(UUID.randomUUID().toString());
        rt.setUser(user);
        rt.setExpiresAt(Instant.now().plusMillis(refreshTtlMs));
        return repo.save(rt);
    }
    
    public RefreshToken verify(String token) {
        RefreshToken rt = repo.findByToken(token)
            .orElseThrow(() -> new TokenRefreshException("Token not found"));
        if (rt.isRevoked()) throw new TokenRefreshException("Revoked");
        if (rt.getExpiresAt().isBefore(Instant.now())) 
            throw new TokenRefreshException("Expired");
        return rt;
    }
    
    public void revoke(String token) {
        repo.findByToken(token).ifPresent(rt -> {
            rt.setRevoked(true);
            repo.save(rt);
        });
    }
}
```

### Refresh Endpoint

```java
@PostMapping("/refresh")
public AuthResponse refresh(@RequestBody RefreshRequest req) {
    RefreshToken rt = refreshService.verify(req.refreshToken());
    UserDetails user = userDetailsService.loadUserByUsername(
        rt.getUser().getUsername());
    String newAccess = jwtService.generateToken(user);
    // Optional: rotate refresh token
    refreshService.revoke(req.refreshToken());
    RefreshToken newRt = refreshService.create(rt.getUser().getUsername());
    return new AuthResponse(newAccess, newRt.getToken(), "Bearer", 3600L);
}

record RefreshRequest(@NotBlank String refreshToken) { }
```

### Best Practices

- Store refresh tokens in DB (hashed)
- Rotate on use
- Revoke on logout
- Bind to device/IP (optional)
- Short refresh TTL for sensitive apps

---

## 7. OAuth2 Concepts

### What is OAuth2?

**OAuth2** is an **authorization** framework allowing an app to access resources on behalf of a user, without exposing the user's password.

### Roles

| Role | Description |
|------|-------------|
| **Resource Owner** | The user |
| **Client** | App requesting access |
| **Authorization Server** | Issues tokens (Keycloak, Auth0) |
| **Resource Server** | Hosts protected resources (your API) |

### Tokens

| Token | Purpose |
|-------|---------|
| **Access Token** | Authorizes API calls (short-lived) |
| **Refresh Token** | Gets new access tokens (long-lived) |
| **ID Token** | (OIDC) Contains user identity claims |

### OAuth2 vs OIDC

- **OAuth2** — authorization (access to resources)
- **OIDC** (OpenID Connect) — authentication (who the user is), built on OAuth2
- ID token = JWT with identity claims

### Scopes

Permissions the client requests: `read:users`, `write:orders`.

### Grants

Different flows depending on client type (see next section).

---

## 8. OAuth2 Grant Types

### 8.1 Authorization Code (with PKCE)

**Best for:** Web apps, SPAs, mobile apps.

```
1. Client redirects user to /authorize on auth server
2. User logs in and grants scopes
3. Auth server redirects back with a "code"
4. Client exchanges code for tokens (back channel)
```

**PKCE** protects against code interception — recommended for SPAs/mobile.

### 8.2 Client Credentials

**Best for:** Machine-to-machine (backend services).

```
1. Client sends client_id + client_secret to /token
2. Auth server returns access token
3. Client uses token to call API
```

No user involved.

### 8.3 Password (Resource Owner Password Credentials)

**Deprecated.** Client collects credentials and exchanges for token. Avoid.

### 8.4 Implicit (Deprecated)

Returned token directly in redirect. Replaced by Auth Code + PKCE.

### 8.5 Refresh Token

Uses refresh token to get new access token (see §6).

### Grant Comparison

| Grant | Client | User | Deprecated |
|-------|--------|------|-----------|
| Authorization Code + PKCE | Web, SPA, mobile | ✅ | No |
| Client Credentials | Backend | ❌ | No |
| Password | Trusted app | ✅ | Yes |
| Implicit | SPA (legacy) | ✅ | Yes |
| Refresh Token | Any with refresh | — | No |

---

## 9. Authorization Code Flow

### Complete Sequence

```
┌────────┐             ┌──────────┐         ┌─────────────┐    ┌──────────┐
│ User   │             │  Client  │         │ Auth Server │    │ Resource │
│        │             │  (App)   │         │ (Keycloak)  │    │ Server   │
└───┬────┘             └────┬─────┘         └──────┬──────┘    └────┬─────┘
    │                       │                      │                │
    │ 1. Click "Login"      │                      │                │
    ├──────────────────────►│                      │                │
    │                       │ 2. Redirect /authorize?response_type=code
    │                       ├─────────────────────►│                │
    │ 3. Login page         │                      │                │
    │◄──────────────────────┼──────────────────────┤                │
    │ 4. Credentials        │                      │                │
    ├──────────────────────►│                      │                │
    │                       │                      │ 5. Validate    │
    │                       │ 6. Redirect ?code=xyz│                │
    │                       │◄─────────────────────┤                │
    │                       │                      │                │
    │                       │ 7. POST /token       │                │
    │                       ├─────────────────────►│                │
    │                       │ 8. access + refresh + id_token          │
    │                       │◄─────────────────────┤                │
    │                       │                      │                │
    │                       │ 9. Call API with Bearer token           │
    │                       ├──────────────────────┼───────────────►│
    │                       │ 10. Validate token (JWKS)               │
    │                       │◄─────────────────────┼────────────────┤
    │                       │ 11. Response                            │
    │◄──────────────────────┤                      │                │
```

### PKCE Addition

Client generates `code_verifier` (random) → `code_challenge = SHA256(code_verifier)`. Sends challenge in step 2, verifier in step 7. Auth server verifies match.

### Spring Config (Client)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```properties
spring.security.oauth2.client.registration.keycloak.client-id=my-app
spring.security.oauth2.client.registration.keycloak.client-secret=secret
spring.security.oauth2.client.registration.keycloak.scope=openid,profile,email
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.keycloak.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
spring.security.oauth2.client.provider.keycloak.issuer-uri=http://localhost:8180/realms/myrealm
```

Spring Security handles the entire flow with `oauth2Login()`.

---

## 10. Client Credentials Flow

### Use Case

Microservice A calls Microservice B, both trusting a central auth server.

### Config (Spring OAuth2 Client)

```properties
spring.security.oauth2.client.registration.service.client-id=order-service
spring.security.oauth2.client.registration.service.client-secret=secret
spring.security.oauth2.client.registration.service.authorization-grant-type=client_credentials
spring.security.oauth2.client.registration.service.scope=read,write
spring.security.oauth2.client.provider.oauth.token-uri=http://localhost:8180/realms/myrealm/protocol/openid-connect/token
```

### Using WebClient

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient(
            OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(
                authorizedClientManager);
        oauth2.setDefaultClientRegistrationId("service");
        
        return WebClient.builder()
            .filter(oauth2)
            .baseUrl("http://inventory-service")
            .build();
    }
}

@Service
public class InventoryClient {
    private final WebClient webClient;
    
    public List<Item> getItems() {
        return webClient.get().uri("/api/items")
            .retrieve().bodyToFlux(Item.class).collectList().block();
    }
}
```

WebClient automatically obtains and refreshes tokens.

---

## 11. Spring as OAuth2 Resource Server

### What It Is

Your API acts as a **Resource Server** — validates incoming Bearer tokens (JWT or opaque) issued by an Authorization Server.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### JWT Resource Server

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8180/realms/myrealm
# or
spring.security.oauth2.resourceserver.jwt.jwk-set-uri=http://localhost:8180/realms/myrealm/protocol/openid-connect/certs
```

### SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthConverter())));
        return http.build();
    }
    
    private JwtAuthenticationConverter jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
        converter.setAuthoritiesClaimName("roles");   // custom claim
        converter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
        jwtConverter.setJwtGrantedAuthoritiesConverter(converter);
        return jwtConverter;
    }
}
```

### Accessing JWT Claims

```java
@GetMapping("/me")
public Map<String, Object> me(@AuthenticationPrincipal Jwt jwt) {
    return Map.of(
        "subject", jwt.getSubject(),
        "claims", jwt.getClaims(),
        "issuer", jwt.getIssuer()
    );
}
```

### Opaque Token Resource Server

For tokens that must be validated against the auth server:

```properties
spring.security.oauth2.resourceserver.opaquetoken.introspection-uri=http://auth/oauth2/introspect
spring.security.oauth2.resourceserver.opaquetoken.client-id=my-resource-server
spring.security.oauth2.resourceserver.opaquetoken.client-secret=secret
```

```java
.oauth2ResourceServer(oauth2 -> oauth2.opaqueToken(Customizer.withDefaults()));
```

### JWT Decoder Customization

```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri("http://auth/jwks").build();
    
    OAuth2TokenValidator<Jwt> validators = new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer("http://auth"),
        new AudienceValidator("my-api"),
        new CustomClaimValidator()
    );
    decoder.setJwtValidator(validators);
    return decoder;
}
```

### AudienceValidator

```java
public class AudienceValidator implements OAuth2TokenValidator<Jwt> {
    private final String audience;
    
    public AudienceValidator(String audience) { this.audience = audience; }
    
    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        if (jwt.getAudience().contains(audience)) {
            return OAuth2TokenValidatorResult.success();
        }
        return OAuth2TokenValidatorResult.failure(
            new OAuth2Error("invalid_token", "Audience mismatch", null));
    }
}
```

### Testing Resource Server

```java
@Test
void jwtAccess() throws Exception {
    mvc.perform(get("/api/users")
        .with(jwt().jwt(j -> j.subject("alice").claim("roles", List.of("USER")))))
        .andExpect(status().isOk());
}
```

---

## 12. Spring Authorization Server

### What It Is

A Spring project providing an OAuth2 Authorization Server implementation. Replaces deprecated `spring-security-oauth2` (which was for authorization server).

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

### Minimal Config

```java
@Configuration
@EnableWebSecurity
public class AuthServerConfig {
    
    @Bean
    @Order(1)
    public SecurityFilterChain authServerChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());   // OpenID Connect 1.0
        return http.build();
    }
    
    @Bean
    @Order(2)
    public SecurityFilterChain defaultChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
    
    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("my-client")
            .clientSecret("{noop}secret")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .redirectUri("http://localhost:8080/login/oauth2/code/my-client")
            .scope(OidcScopes.OPENID)
            .scope("read")
            .build();
        return new InMemoryRegisteredClientRepository(client);
    }
    
    @Bean
    public JWKSource<SecurityContext> jwkSource() { /* generate RSA key */ }
    
    @Bean
    public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
        return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
    }
    
    @Bean
    public AuthorizationServerSettings settings() {
        return AuthorizationServerSettings.builder()
            .issuer("http://localhost:9000")
            .build();
    }
}
```

### Endpoints

- `/.well-known/openid-configuration` — OIDC discovery
- `/oauth2/authorize` — Authorization endpoint
- `/oauth2/token` — Token endpoint
- `/oauth2/jwks` — JWK Set
- `/oauth2/revoke` — Revocation
- `/oauth2/introspect` — Introspection
- `/userinfo` — OIDC UserInfo

### When to Use

- You need to be your own auth provider
- Building a platform where you control users
- OIDC for SSO

### When Not to Use

- Already using Keycloak/Auth0/Okta — use those
- Building internal apps — external IdP simpler

---

## 13. Keycloak Integration

### What is Keycloak?

Open-source Identity and Access Management. Full OIDC + SAML + LDAP. Runs standalone.

### Run via Docker

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start-dev
    ports:
      - "8180:8080"
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
```

Visit `http://localhost:8180`, login with `admin/admin`.

### Setup

1. Create a **realm** (e.g., `myrealm`)
2. Create a **client** (`my-app`) with:
   - Client authentication: On
   - Valid redirect URIs: `http://localhost:8080/*`
   - Web origins: `http://localhost:8080`
3. Create **roles** (`USER`, `ADMIN`)
4. Create **users** and assign roles

### Application Config (Client)

```properties
spring.security.oauth2.client.registration.keycloak.client-id=my-app
spring.security.oauth2.client.registration.keycloak.client-secret=<from-keycloak>
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.keycloak.scope=openid,profile,email
spring.security.oauth2.client.provider.keycloak.issuer-uri=http://localhost:8180/realms/myrealm
```

### SecurityConfig (OAuth2 Client)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()
                .anyRequest().authenticated())
            .oauth2Login(oauth2 -> oauth2
                .defaultSuccessUrl("/dashboard", true));
        return http.build();
    }
}
```

### Role Mapping

Keycloak roles are in `realm_access.roles`. Custom converter:

```java
@Bean
public GrantedAuthoritiesMapper authoritiesMapper() {
    return authorities -> {
        Set<GrantedAuthority> mapped = new HashSet<>();
        authorities.forEach(a -> {
            if (a.getAuthority().startsWith("ROLE_")) {
                mapped.add(a);
            }
        });
        return mapped;
    };
}
```

Or use Keycloak's adapter / Spring's Keycloak OIDC (`spring-boot-starter-oauth2-client` with custom `OidcUserService`).

### Resource Server Using Keycloak

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8180/realms/myrealm
spring.security.oauth2.resourceserver.jwt.jwk-set-uri=http://localhost:8180/realms/myrealm/protocol/openid-connect/certs
```

### Role Extraction from Keycloak JWT

Keycloak puts roles in `realm_access.roles` (nested):

```java
@Bean
public JwtAuthenticationConverter jwtConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess == null) return List.of();
        List<String> roles = (List<String>) realmAccess.get("roles");
        return roles.stream()
            .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
            .collect(Collectors.toList());
    });
    return converter;
}
```

### Keycloak Advantages

- Full IAM (users, roles, groups, sessions)
- LDAP/AD integration
- Social login brokers (Google, GitHub)
- Admin UI
- Standards-compliant

---

## 14. Social Login (Google, GitHub)

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### Google Config

1. Google Cloud Console → APIs & Services → Credentials → OAuth 2.0 Client
2. Add redirect URI: `http://localhost:8080/login/oauth2/code/google`
3. Copy client ID + secret

```properties
spring.security.oauth2.client.registration.google.client-id=<client-id>
spring.security.oauth2.client.registration.google.client-secret=<client-secret>
spring.security.oauth2.client.registration.google.scope=openid,profile,email
```

### GitHub Config

1. GitHub → Settings → Developer settings → OAuth Apps → New
2. Callback: `http://localhost:8080/login/oauth2/code/github`

```properties
spring.security.oauth2.client.registration.github.client-id=<client-id>
spring.security.oauth2.client.registration.github.client-secret=<client-secret>
spring.security.oauth2.client.registration.github.scope=read:user,user:email
```

### SecurityConfig

```java
http.oauth2Login(oauth2 -> oauth2
    .loginPage("/login")
    .defaultSuccessUrl("/dashboard", true)
);
```

Default login page shows links for each registered provider.

### Custom OAuth2UserService

For user creation/update:

```java
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {
    
    private final UserRepository userRepo;
    
    @Override
    public OAuth2User loadUser(OAuth2UserRequest req) throws OAuth2AuthenticationException {
        OAuth2User user = super.loadUser(req);
        String email = user.getAttribute("email");
        String provider = req.getClientRegistration().getRegistrationId();
        
        userRepo.findByEmail(email)
            .orElseGet(() -> userRepo.save(new User(email, provider)));
        
        return user;
    }
}
```

```java
http.oauth2Login(oauth2 -> oauth2
    .userInfoEndpoint(u -> u.userService(customOAuth2UserService))
);
```

### Multiple Providers in One App

Register both Google and GitHub; Spring shows both as options.

### Scopes

| Provider | Common Scopes |
|----------|--------------|
| Google | `openid`, `profile`, `email` |
| GitHub | `read:user`, `user:email`, `repo` |
| Facebook | `public_profile`, `email` |
| Microsoft | `openid`, `profile`, `email`, `User.Read` |

---

## 15. Interview Questions

### Q1. What is JWT and how does it work?

**Answer:** JSON Web Token — a compact, URL-safe token with three parts: header (algorithm), payload (claims), and signature. Encoded base64url. Server signs with secret (HS256) or private key (RS256). Clients send it in `Authorization: Bearer <token>`. Server verifies signature and expiry without querying DB.

### Q2. What are the parts of a JWT?

**Answer:** Header (`alg`, `typ`), Payload (claims: `sub`, `exp`, `iss`, custom), Signature (`HMAC(header.payload, secret)` or RSA). Base64url encoded and dot-separated.

### Q3. Difference between HS256 and RS256?

**Answer:** HS256 uses a symmetric shared secret for sign + verify. RS256 uses a private key to sign and public key to verify. RS256 is better for distributed systems — many services verify with the public key without knowing the secret. HS256 is faster and simpler.

### Q4. How do you revoke a JWT?

**Answer:** JWTs are stateless — you can't revoke directly. Options: (1) **short expiry** (15 min), (2) **refresh tokens** with DB storage (revoke refresh), (3) **blacklist** (store revoked JTIs in Redis until exp), (4) rotate signing keys.

### Q5. Where should JWT be stored on the client?

**Answer:** Best: **httpOnly + Secure + SameSite** cookie (XSS-safe). Alternative: **in-memory** (XSS-hard, lost on refresh). Avoid `localStorage` (XSS vulnerable). Never in URL.

### Q6. What is OAuth2?

**Answer:** An **authorization** framework allowing apps to access resources on behalf of users without exposing credentials. Roles: Resource Owner, Client, Authorization Server, Resource Server. Uses access tokens (short-lived) and refresh tokens.

### Q7. What is OIDC?

**Answer:** OpenID Connect — an **authentication** layer on top of OAuth2. Adds an **ID token** (JWT) containing user identity claims, plus `userinfo` endpoint and discovery. OAuth2 = authorization; OIDC = authentication.

### Q8. Explain the Authorization Code flow.

**Answer:** Client redirects user to Authorization Server. User authenticates and grants scope. Server redirects back with `code`. Client exchanges code for tokens via back-channel. Code is single-use. PKCE adds a code_verifier/challenge for SPAs/mobile.

### Q9. What is PKCE and why use it?

**Answer:** Proof Key for Code Exchange. Client generates `code_verifier` (random) and `code_challenge = SHA256(verifier)`. Sends challenge in `/authorize`; sends verifier in `/token`. Server verifies. Prevents authorization code interception attacks (critical for public clients).

### Q10. When to use Client Credentials?

**Answer:** Machine-to-machine (no user). Backend service authenticates with client_id + secret, gets access token for API calls. Common in microservices.

### Q11. What is the difference between OAuth2 and JWT?

**Answer:** OAuth2 is an **authorization framework** (how to get tokens). JWT is a **token format** (how tokens are structured). OAuth2 access tokens can be JWTs or opaque. JWT can be used without OAuth2 (custom auth).

### Q12. What is the Resource Server?

**Answer:** Your API — hosts protected resources and validates access tokens. In Spring: `spring-boot-starter-oauth2-resource-server` + `oauth2ResourceServer()`.

### Q13. What is the Authorization Server?

**Answer:** Issues tokens. Examples: Keycloak, Auth0, Okta, Spring Authorization Server. Verifies user credentials, applies grant flow, returns tokens.

### Q14. What is the ID token?

**Answer:** An OIDC JWT containing user identity claims (`sub`, `email`, `name`, `picture`). Used by clients to know who the user is. Distinct from access token (for API calls) and refresh token (for renewing).

### Q15. How do you decode and validate a JWT in Spring?

**Answer:** Use `spring-security-oauth2-resource-server` with `issuer-uri` or `jwk-set-uri`. Spring fetches JWKs, validates signature, expiry, issuer, audience. Access claims via `@AuthenticationPrincipal Jwt jwt`.

### Q16. What is JWK Set?

**Answer:** JSON Web Key Set — a JSON document with public keys used to verify JWTs. Authorization Server exposes it at `/.well-known/jwks.json` or `/oauth2/jwks`. Enables key rotation without client changes.

### Q17. How do you handle logout with JWT?

**Answer:** JWTs are stateless. Options: (1) client discards token, (2) revoke refresh token in DB, (3) add token JTI to blacklist (Redis, TTL = remaining exp), (4) rotate signing keys (invalidates all tokens).

### Q18. What is the difference between access token and refresh token?

**Answer:** Access token: short-lived (15 min–1 hour), used for API calls, stateless (JWT). Refresh token: long-lived (days–weeks), used only to get new access tokens, stored in DB, can be revoked, rotated on use.

### Q19. How do you configure a client with Spring Security?

**Answer:** Add `spring-boot-starter-oauth2-client`, configure `spring.security.oauth2.client.registration.<id>.*` and `.provider.<id>.*`, then `http.oauth2Login()`. Spring handles the entire auth code flow.

### Q20. What is Keycloak and why use it?

**Answer:** Open-source IAM server providing OIDC + SAML + LDAP. Centralizes user management, roles, groups. Reduces code — your apps become OAuth2 clients/resource servers. Good for multi-app platforms.

### Q21. How do you test JWT-secured endpoints?

**Answer:** Use Spring Security test with `.with(jwt().jwt(j -> j.subject("alice").claim("roles", List.of("USER"))))`. Or generate tokens with a test `JwtService`.

### Q22. What claims should you never put in a JWT?

**Answer:** Sensitive data (passwords, PII, API keys) — JWT payload is base64 (readable by anyone). Large data (increases token size, hurts performance). Frequently changing data (defeats stateless caching).

### Q23. How do you prevent JWT replay attacks?

**Answer:** (1) Short expiry, (2) HTTPS only, (3) `jti` claim + blacklist, (4) bind to client (fingerprint/DPoP), (5) rotate tokens, (6) use refresh tokens with rotation.

### Q24. What is audience validation?

**Answer:** Checking the `aud` claim matches the intended recipient (your API). Prevents a token issued for another service from being accepted. Implement `OAuth2TokenValidator` with `AudienceValidator`.

### Q25. Can you use JWT with Spring Security without OAuth2?

**Answer:** Yes. Generate JWTs yourself (via jjwt) after successful authentication, add a `JwtAuthenticationFilter` to validate them and set `SecurityContext`. No Authorization Server needed. This is "custom JWT auth".

---

## 16. Cheat Sheet

### JWT Dependencies

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

### OAuth2 Dependencies

```xml
<!-- Client -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<!-- Resource Server -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>

<!-- Authorization Server -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

### JWT Generate/Parse (jjwt 0.12+)

```java
// Generate
String token = Jwts.builder()
    .subject(user.getUsername())
    .issuer("my-app")
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + 3600_000))
    .claim("roles", List.of("USER"))
    .signWith(key, Jwts.SIG.HS256)
    .compact();

// Parse
Claims claims = Jwts.parser()
    .verifyWith(key)
    .build()
    .parseSignedClaims(token)
    .getPayload();
```

### Resource Server Config

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://auth/realms/myrealm
```

```java
http.oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()));
```

### OAuth2 Client Config

```properties
spring.security.oauth2.client.registration.google.client-id=...
spring.security.oauth2.client.registration.google.client-secret=...
spring.security.oauth2.client.registration.google.scope=openid,profile,email
```

```java
http.oauth2Login(Customizer.withDefaults());
```

### Grant Types

| Grant | Use Case |
|-------|----------|
| Authorization Code + PKCE | Web, SPA, mobile |
| Client Credentials | Backend-to-backend |
| Refresh Token | Renew access tokens |
| ~~Password~~ | Deprecated |
| ~~Implicit~~ | Deprecated |

### Key Endpoints (Auth Server)

```
/.well-known/openid-configuration
/oauth2/authorize
/oauth2/token
/oauth2/jwks
/oauth2/introspect
/oauth2/revoke
/userinfo
```

### Token Types

| Token | Purpose | Lifetime | Format |
|-------|---------|----------|--------|
| Access | API calls | 15 min – 1 hr | JWT or opaque |
| Refresh | Renew access | Days | Opaque (DB) |
| ID (OIDC) | User identity | Same as access | JWT |

### Standard Claims

```
iss  issuer
sub  subject
aud  audience
exp  expiration
nbf  not before
iat  issued at
jti  JWT ID
```

### Test JWT

```java
mvc.perform(get("/api/users")
    .with(jwt().jwt(j -> j.subject("alice").claim("roles", List.of("USER")))))
    .andExpect(status().isOk());
```

### Security Pitfalls

- ❌ `alg: none` — always verify algorithm
- ❌ Storing JWT in localStorage
- ❌ Long-lived access tokens
- ❌ No audience/issuer validation
- ❌ Accepting tokens without signature check
- ❌ Sharing secrets in code/repo
- ❌ Using deprecated Password/Implicit grants
- ❌ Not rotating refresh tokens
- ❌ No HTTPS in production
- ❌ Trusting JWT claims without verification

### Cross-References

- **Previous:** `13_Spring_Security_Core.md`
- **Next:** `15_Spring_Security_Method_Level.md`
- **Related:** `03_Spring_MVC.md`, `19_Spring_Microservices_Cloud.md`
- **Interview:** `24_Spring_Interview_Questions.md` (§Security)

---

## 🔗 Navigation

- **📖 [Table of Contents](../../../README.md)**
- **← Previous:** [13_Spring_Security_Core.md](./13_Spring_Security_Core.md)
- **Next →:** [15_Spring_Security_Method_Level.md](./15_Spring_Security_Method_Level.md)
- **Related:** [13_Spring_Security_Core.md](./13_Spring_Security_Core.md), [15_Spring_Security_Method_Level.md](./15_Spring_Security_Method_Level.md), [19_Spring_Microservices_Cloud.md](./19_Spring_Microservices_Cloud.md)

---

*Part of the [Spring Study Guide](../../../README.md) — ⭐ star the repo if it helped!*
