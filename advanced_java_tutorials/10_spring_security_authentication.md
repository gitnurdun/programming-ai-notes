# Advanced Spring Security and Authentication

## Overview
Spring Security is a comprehensive security framework that provides authentication, authorization, and protection against common security vulnerabilities. Understanding its advanced features is crucial for building secure enterprise applications.

## Core Security Architecture

### Filter Chain and Security Context

```java
// Custom authentication filter
public class CustomAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    
    private final AuthenticationManager authenticationManager;
    private final UserRepository userRepository;
    
    public CustomAuthenticationFilter(AuthenticationManager authenticationManager,
                                    UserRepository userRepository) {
        super("/api/login", new AntPathRequestMatcher("/api/login", "POST"));
        this.authenticationManager = authenticationManager;
        this.userRepository = userRepository;
    }
    
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, 
                                              HttpServletResponse response) 
            throws AuthenticationException, IOException {
        
        String username = request.getParameter("username");
        String password = request.getParameter("password");
        String token = request.getHeader("X-Custom-Token");
        
        if (token != null) {
            // Custom token authentication
            return authenticationManager.authenticate(
                new CustomTokenAuthenticationToken(token, null));
        } else {
            // Username/password authentication
            CustomAuthenticationToken authRequest = 
                new CustomAuthenticationToken(username, password);
            return authenticationManager.authenticate(authRequest);
        }
    }
    
    @Override
    protected void successfulAuthentication(HttpServletRequest request,
                                          HttpServletResponse response,
                                          FilterChain chain,
                                          Authentication authResult) 
            throws IOException, ServletException {
        
        // Add custom headers or tokens
        response.setHeader("X-Auth-Token", generateToken(authResult));
        
        SecurityContextHolder.getContext().setAuthentication(authResult);
        chain.doFilter(request, response);
    }
    
    private String generateToken(Authentication authResult) {
        // Generate JWT or custom token
        return "custom-token-" + System.currentTimeMillis();
    }
}

// Custom authentication token
public class CustomAuthenticationToken extends AbstractAuthenticationToken {
    private final Object principal;
    private final Object credentials;
    
    public CustomAuthenticationToken(Object principal, Object credentials) {
        super(Collections.emptyList());
        this.principal = principal;
        this.credentials = credentials;
        setAuthenticated(false);
    }
    
    public CustomAuthenticationToken(Object principal, Object credentials, 
                                   Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        this.credentials = credentials;
        super.setAuthenticated(true);
    }
    
    @Override
    public Object getCredentials() {
        return this.credentials;
    }
    
    @Override
    public Object getPrincipal() {
        return this.principal;
    }
    
    @Override
    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException(
                "Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        }
        super.setAuthenticated(false);
    }
}
```

### Authentication Provider Implementation

```java
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {
    
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;
    private final UserDetailsService userDetailsService;
    private final TwoFactorService twoFactorService;
    
    public CustomAuthenticationProvider(PasswordEncoder passwordEncoder,
                                      UserRepository userRepository,
                                      UserDetailsService userDetailsService,
                                      TwoFactorService twoFactorService) {
        this.passwordEncoder = passwordEncoder;
        this.userRepository = userRepository;
        this.userDetailsService = userDetailsService;
        this.twoFactorService = twoFactorService;
    }
    
    @Override
    public Authentication authenticate(Authentication authentication) 
            throws AuthenticationException {
        
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();
        
        // Load user
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        
        // Validate credentials
        if (!passwordEncoder.matches(password, user.getPassword())) {
            handleFailedLogin(user);
            throw new BadCredentialsException("Invalid credentials");
        }
        
        // Check account status
        validateUserStatus(user);
        
        // Check for 2FA requirement
        if (user.isTwoFactorEnabled()) {
            String twoFactorCode = extractTwoFactorCode(authentication);
            validateTwoFactorCode(user, twoFactorCode);
        }
        
        // Update last login
        updateLoginInfo(user);
        
        // Create authenticated token
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
        return new CustomAuthenticationToken(
            userDetails, password, userDetails.getAuthorities());
    }
    
    @Override
    public boolean supports(Class<?> authentication) {
        return CustomAuthenticationToken.class.isAssignableFrom(authentication);
    }
    
    private void handleFailedLogin(User user) {
        int failedAttempts = user.getFailedLoginAttempts() + 1;
        user.setFailedLoginAttempts(failedAttempts);
        
        if (failedAttempts >= 5) {
            user.setAccountLocked(true);
            user.setLockoutTime(LocalDateTime.now().plusMinutes(30));
        }
        
        userRepository.save(user);
    }
    
    private void validateUserStatus(User user) {
        if (!user.isEnabled()) {
            throw new DisabledException("User account is disabled");
        }
        
        if (user.isAccountLocked()) {
            if (LocalDateTime.now().isBefore(user.getLockoutTime())) {
                throw new LockedException("Account is temporarily locked");
            } else {
                // Unlock account after lockout period
                user.setAccountLocked(false);
                user.setFailedLoginAttempts(0);
                userRepository.save(user);
            }
        }
        
        if (user.getPasswordExpiryDate() != null && 
            user.getPasswordExpiryDate().isBefore(LocalDateTime.now())) {
            throw new CredentialsExpiredException("Password has expired");
        }
    }
    
    private String extractTwoFactorCode(Authentication authentication) {
        // Extract 2FA code from authentication details
        if (authentication.getDetails() instanceof WebAuthenticationDetails) {
            // Implementation depends on how you pass the 2FA code
            return null;
        }
        return null;
    }
    
    private void validateTwoFactorCode(User user, String code) {
        if (!twoFactorService.validateCode(user.getTwoFactorSecret(), code)) {
            throw new TwoFactorAuthenticationException("Invalid 2FA code");
        }
    }
    
    private void updateLoginInfo(User user) {
        user.setLastLoginTime(LocalDateTime.now());
        user.setFailedLoginAttempts(0);
        userRepository.save(user);
    }
}
```

## Advanced Configuration

### Security Configuration with Multiple Authentication Methods

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)
public class SecurityConfig {
    
    private final CustomAuthenticationProvider customAuthenticationProvider;
    private final UserRepository userRepository;
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    public SecurityConfig(CustomAuthenticationProvider customAuthenticationProvider,
                         UserRepository userRepository,
                         JwtService jwtService,
                         UserDetailsService userDetailsService) {
        this.customAuthenticationProvider = customAuthenticationProvider;
        this.userRepository = userRepository;
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/users/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error=true")
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout=true")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID", "auth_token")
            )
            .exceptionHandling(ex -> ex
                .accessDeniedHandler(customAccessDeniedHandler())
                .authenticationEntryPoint(authenticationEntryPoint())
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)
                .expiredUrl("/login?expired=true")
            )
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**") // For REST APIs
            )
            .addFilterBefore(customAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(jwtAuthenticationFilter(), CustomAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) 
            throws Exception {
        AuthenticationManagerBuilder auth = config.getAuthenticationManagerBuilder();
        auth.authenticationProvider(customAuthenticationProvider);
        return auth.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    @Bean
    public CustomAuthenticationFilter customAuthenticationFilter() {
        return new CustomAuthenticationFilter(
            authenticationManager(null), userRepository);
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter(jwtService, userDetailsService);
    }
    
    @Bean
    public AccessDeniedHandler customAccessDeniedHandler() {
        return new CustomAccessDeniedHandler();
    }
    
    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint() {
        return new CustomAuthenticationEntryPoint();
    }
}
```

### JWT Authentication Implementation

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) 
            throws ServletException, IOException {
        
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            
            try {
                String username = jwtService.extractUsername(token);
                
                if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                    
                    if (jwtService.validateToken(token, userDetails)) {
                        UsernamePasswordAuthenticationToken authToken = 
                            new UsernamePasswordAuthenticationToken(
                                userDetails, null, userDetails.getAuthorities());
                        authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                        SecurityContextHolder.getContext().setAuthentication(authToken);
                    }
                }
            } catch (Exception e) {
                logger.error("JWT token processing failed: " + e.getMessage());
            }
        }
        
        filterChain.doFilter(request, response);
    }
}

@Service
public class JwtService {
    
    private final String secret = "mySecretKey123456789012345678901234567890"; // At least 32 chars
    private final long jwtExpiration = 86400000; // 24 hours
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSignInKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
    
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }
    
    private Boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        return createToken(claims, userDetails.getUsername());
    }
    
    private String createToken(Map<String, Object> claims, String subject) {
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
                .signWith(getSignInKey(), SignatureAlgorithm.HS512)
                .compact();
    }
    
    private Key getSignInKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

## Method Security and Authorization

### Method-Level Security

```java
@Service
@Transactional
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
    
    @PreAuthorize("hasRole('ADMIN') or authentication.name == #username")
    public User getUserByUsername(String username) {
        return userRepository.findByUsername(username)
            .orElseThrow(() -> new UserNotFoundException(username));
    }
    
    @PreAuthorize("hasRole('ADMIN') or #user.id == authentication.principal.id")
    public User updateUser(User user, UserUpdateRequest request) {
        // Update user with validation
        user.setFirstName(request.getFirstName());
        user.setLastName(request.getLastName());
        user.setEmail(request.getEmail());
        return userRepository.save(user);
    }
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
    
    @PreAuthorize("hasRole('ADMIN') or hasPermission(#user, 'WRITE')")
    public void changeUserPassword(User user, String newPassword) {
        user.setPassword(passwordEncoder.encode(newPassword));
        userRepository.save(user);
    }
    
    @PostAuthorize("returnObject.owner == authentication.name or hasRole('ADMIN')")
    public List<Document> getUserDocuments(String username) {
        return documentRepository.findByOwner(username);
    }
    
    @Secured("ROLE_ADMIN")
    public void adminOnlyOperation() {
        // Only accessible to ADMIN role
    }
    
    @RolesAllowed({"ADMIN", "MANAGER"})
    public void managerOrAdminOperation() {
        // Accessible to ADMIN or MANAGER roles
    }
    
    @PreAuthorize("@userSecurityService.canAccessUser(authentication, #userId)")
    public User getUserWithCustomCheck(Long userId) {
        return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }
}

// Custom security service
@Component("userSecurityService")
public class UserSecurityService {
    
    public boolean canAccessUser(Authentication authentication, Long userId) {
        // Custom security logic
        String authenticatedUser = authentication.getName();
        User user = userRepository.findById(userId).orElse(null);
        
        if (user == null) return false;
        
        // Allow access if user is admin or accessing own data
        return authentication.getAuthorities().stream()
            .anyMatch(auth -> auth.getAuthority().equals("ROLE_ADMIN")) ||
            user.getUsername().equals(authenticatedUser);
    }
}
```

### Custom Permission Evaluator

```java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {
    
    private final UserRepository userRepository;
    private final DocumentRepository documentRepository;
    
    public CustomPermissionEvaluator(UserRepository userRepository, 
                                   DocumentRepository documentRepository) {
        this.userRepository = userRepository;
        this.documentRepository = documentRepository;
    }
    
    @Override
    public boolean hasPermission(Authentication authentication, Object targetDomainObject, 
                               Object permission) {
        
        if (authentication == null || targetDomainObject == null || permission == null) {
            return false;
        }
        
        String username = authentication.getName();
        String perm = permission.toString();
        
        if (targetDomainObject instanceof Document) {
            Document document = (Document) targetDomainObject;
            return checkDocumentPermission(username, document, perm);
        }
        
        return false;
    }
    
    @Override
    public boolean hasPermission(Authentication authentication, Serializable targetId, 
                               String targetType, Object permission) {
        
        if (authentication == null || targetId == null || targetType == null || permission == null) {
            return false;
        }
        
        String username = authentication.getName();
        String perm = permission.toString();
        
        if ("document".equals(targetType)) {
            Document document = documentRepository.findById((Long) targetId).orElse(null);
            if (document != null) {
                return checkDocumentPermission(username, document, perm);
            }
        }
        
        return false;
    }
    
    private boolean checkDocumentPermission(String username, Document document, String permission) {
        User user = userRepository.findByUsername(username).orElse(null);
        if (user == null) return false;
        
        switch (permission.toLowerCase()) {
            case "read":
                return document.isPublic() || document.getOwner().equals(username) || 
                       document.getReadAccessUsers().contains(username);
            case "write":
                return document.getOwner().equals(username) || 
                       document.getWriteAccessUsers().contains(username);
            case "delete":
                return document.getOwner().equals(username);
            default:
                return false;
        }
    }
}
```

## OAuth2 and OpenID Connect

### OAuth2 Client Configuration

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**", "/oauth2/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/oauth2/authorization/google")
                .defaultSuccessUrl("/dashboard")
                .failureUrl("/login?error=true")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                )
                .successHandler(customAuthenticationSuccessHandler())
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );
        
        return http.build();
    }
    
    @Bean
    public OAuth2UserService<OidcUserRequest, OidcUser> customOAuth2UserService() {
        return new CustomOAuth2UserService();
    }
    
    @Bean
    public AuthenticationSuccessHandler customAuthenticationSuccessHandler() {
        return new CustomAuthenticationSuccessHandler();
    }
    
    private JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}

// Custom OAuth2 user service
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OidcUserRequest, OidcUser> {
    
    private final UserRepository userRepository;
    
    public CustomOAuth2UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public OidcUser loadUser(OidcUserRequest userRequest) throws OAuth2AuthenticationException {
        
        OAuth2UserService<OidcUserRequest, OidcUser> delegate = 
            new OidcUserService();
        OidcUser oidcUser = delegate.loadUser(userRequest);
        
        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String userNameAttributeName = userRequest.getClientRegistration()
            .getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName();
        
        OAuth2UserInfo oAuth2UserInfo = OAuth2UserInfoFactory
            .getOAuth2UserInfo(registrationId, oidcUser.getAttributes());
        
        User user = processUserRegistration(registrationId, oAuth2UserInfo);
        
        return new CustomOAuth2User(user, oidcUser.getAttributes(), userNameAttributeName);
    }
    
    private User processUserRegistration(String registrationId, OAuth2UserInfo oAuth2UserInfo) {
        Optional<User> userOptional = userRepository.findByEmail(oAuth2UserInfo.getEmail());
        User user;
        
        if (userOptional.isPresent()) {
            user = userOptional.get();
            if (!user.getProvider().equals(AuthProvider.valueOf(registrationId.toUpperCase()))) {
                throw new OAuth2AuthenticationProcessingException(
                    "Looks like you're signed up with " + 
                    user.getProvider() + " account. Please use that account to login.");
            }
            user = updateExistingUser(user, oAuth2UserInfo);
        } else {
            user = registerNewUser(registrationId, oAuth2UserInfo);
        }
        
        return user;
    }
    
    private User registerNewUser(String registrationId, OAuth2UserInfo oAuth2UserInfo) {
        User user = new User();
        user.setProvider(AuthProvider.valueOf(registrationId.toUpperCase()));
        user.setProviderId(oAuth2UserInfo.getId());
        user.setName(oAuth2UserInfo.getName());
        user.setEmail(oAuth2UserInfo.getEmail());
        user.setImageUrl(oAuth2UserInfo.getImageUrl());
        user.setRoles(Set.of(UserRole.USER));
        
        return userRepository.save(user);
    }
    
    private User updateExistingUser(User existingUser, OAuth2UserInfo oAuth2UserInfo) {
        existingUser.setName(oAuth2UserInfo.getName());
        existingUser.setImageUrl(oAuth2UserInfo.getImageUrl());
        return userRepository.save(existingUser);
    }
}
```

### OAuth2 Resource Server with JWT

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtDecoder(jwtDecoder()))
            );
        
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withJwkSetUri("https://your-auth-server/.well-known/jwks.json")
            .jwsKeySelector(new JwsKeySelector() {
                @Override
                public List<Key> selectJwk(JwsHeader jwsHeader, SignedJWT signedJWT) 
                        throws KeySourceException {
                    // Custom key selection logic
                    return List.of(); // Implementation depends on your auth server
                }
            })
            .build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        
        // Extract authorities from custom claim
        authoritiesConverter.setAuthoritiesClaimName("authorities");
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        
        return converter;
    }
}
```

## Security Testing

### Security Testing with Spring Security Test

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@AutoConfigureTestDatabase
@Transactional
class SecurityControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void testAdminAccess() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void testUserAccess() throws Exception {
        mockMvc.perform(get("/api/user/profile"))
            .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void testAdminAccessDenied() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isForbidden());
    }
    
    @Test
    void testUnauthenticatedAccess() throws Exception {
        mockMvc.perform(get("/api/user/profile"))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void testUserAccessWithCustomAttribute() throws Exception {
        // Test with custom security attributes
        mockMvc.perform(get("/api/users/1")
            .requestAttr("custom-attribute", "value"))
            .andExpect(status().isOk());
    }
}

// Integration test with real authentication
@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class SecurityIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testJwtAuthentication() {
        // Given
        User testUser = new User();
        testUser.setUsername("testuser");
        testUser.setPassword("$2a$12$..."); // BCrypt encoded password
        testUser.setRoles(Set.of("USER"));
        userRepository.save(testUser);
        
        // When - Login to get JWT
        LoginRequest loginRequest = new LoginRequest("testuser", "password");
        ResponseEntity<LoginResponse> loginResponse = restTemplate.postForEntity(
            "/api/auth/login", loginRequest, LoginResponse.class);
        
        assertThat(loginResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        String jwtToken = loginResponse.getBody().getToken();
        
        // Then - Use JWT for protected endpoint
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(jwtToken);
        HttpEntity<String> entity = new HttpEntity<>(headers);
        
        ResponseEntity<String> response = restTemplate.exchange(
            "/api/user/profile", HttpMethod.GET, entity, String.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

## Advanced Security Features

### Security Event Listeners

```java
@Component
public class SecurityEventListener {
    
    private final Logger logger = LoggerFactory.getLogger(SecurityEventListener.class);
    private final AuditLogService auditLogService;
    
    public SecurityEventListener(AuditLogService auditLogService) {
        this.auditLogService = auditLogService;
    }
    
    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        Authentication auth = event.getAuthentication();
        logger.info("Authentication success for user: {}", auth.getName());
        
        auditLogService.logEvent(AuditEvent.AUTHENTICATION_SUCCESS, 
                               auth.getName(), 
                               "User authenticated successfully");
    }
    
    @EventListener
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        AuthenticationException exception = event.getException();
        Authentication authentication = event.getAuthentication();
        
        logger.warn("Authentication failure for user: {}, reason: {}", 
                   authentication.getName(), exception.getMessage());
        
        auditLogService.logEvent(AuditEvent.AUTHENTICATION_FAILURE, 
                               authentication.getName(), 
                               "Authentication failed: " + exception.getMessage());
    }
    
    @EventListener
    public void handleConcurrentSessionExpired(SessionDestroyedEvent event) {
        logger.info("Session destroyed: {}", event.getId());
        auditLogService.logEvent(AuditEvent.SESSION_EXPIRED, 
                               extractUsername(event), 
                               "Session expired due to concurrent login");
    }
    
    @EventListener
    public void handleAuthorizationFailure(AuthorizationFailureEvent event) {
        Authentication authentication = event.getAuthentication();
        Object object = event.getSource();
        
        logger.warn("Authorization failure for user: {} on resource: {}", 
                   authentication.getName(), object.toString());
        
        auditLogService.logEvent(AuditEvent.AUTHORIZATION_FAILURE, 
                               authentication.getName(), 
                               "Access denied to: " + object.toString());
    }
    
    private String extractUsername(SessionDestroyedEvent event) {
        // Extract username from session or other context
        return "unknown";
    }
}
```

## Conclusion

Advanced Spring Security provides:
- Comprehensive authentication and authorization mechanisms
- Flexible security configuration options
- JWT and OAuth2 integration
- Method-level security with custom evaluators
- Advanced testing capabilities
- Security event monitoring and auditing

Mastering these features is essential for:
- Senior Java security roles
- Enterprise application security
- Authentication and authorization system design
- Compliance with security standards
- Protection against common vulnerabilities