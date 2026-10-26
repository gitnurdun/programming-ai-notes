# Advanced Security Best Practices and Implementation

## Overview
Security is paramount in modern Java applications. This tutorial covers comprehensive security best practices including authentication, authorization, data protection, secure coding, and implementation of security frameworks. Understanding these concepts is crucial for building secure, production-ready applications.

## Authentication and Authorization

### Advanced Authentication Patterns

```java
import java.security.*;
import java.security.spec.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import javax.crypto.*;
import javax.crypto.spec.*;
import java.util.function.Predicate;
import java.util.stream.Collectors;

public class AdvancedAuthentication {
    
    // JWT token implementation with security best practices
    public static class JwtTokenService {
        private final String secretKey;
        private final String issuer;
        private final long expirationTime; // in milliseconds
        private final Set<String> blacklistedTokens = ConcurrentHashMap.newKeySet();
        private final Map<String, Integer> tokenUsageCount = new ConcurrentHashMap<>();
        private final int maxUsageCount; // Prevent token replay attacks
        
        public JwtTokenService(String secretKey, String issuer, long expirationTime, int maxUsageCount) {
            this.secretKey = secretKey;
            this.issuer = issuer;
            this.expirationTime = expirationTime;
            this.maxUsageCount = maxUsageCount;
        }
        
        public String generateToken(String subject, Map<String, Object> claims) {
            try {
                long issuedAt = System.currentTimeMillis();
                long expiration = issuedAt + expirationTime;
                
                // Create JWT header
                String header = Base64.getUrlEncoder().encodeToString(
                    ("{\"typ\":\"JWT\",\"alg\":\"HS512\"}").getBytes());
                
                // Create JWT payload
                StringBuilder payloadBuilder = new StringBuilder();
                payloadBuilder.append("{\"sub\":\"").append(subject).append("\"");
                payloadBuilder.append(",\"iss\":\"").append(issuer).append("\"");
                payloadBuilder.append(",\"iat\":").append(issuedAt);
                payloadBuilder.append(",\"exp\":").append(expiration);
                
                // Add custom claims
                for (Map.Entry<String, Object> entry : claims.entrySet()) {
                    payloadBuilder.append(",\"").append(entry.getKey()).append("\":");
                    if (entry.getValue() instanceof String) {
                        payloadBuilder.append("\"").append(entry.getValue()).append("\"");
                    } else {
                        payloadBuilder.append(entry.getValue());
                    }
                }
                
                payloadBuilder.append("}");
                
                String payload = Base64.getUrlEncoder().encodeToString(payloadBuilder.toString().getBytes());
                
                // Create signature
                String signature = createSignature(header, payload);
                
                return header + "." + payload + "." + signature;
            } catch (Exception e) {
                throw new RuntimeException("Failed to generate JWT token", e);
            }
        }
        
        public JwtClaims validateToken(String token) {
            if (token == null || token.split("\\.").length != 3) {
                throw new IllegalArgumentException("Invalid token format");
            }
            
            if (blacklistedTokens.contains(token)) {
                throw new SecurityException("Token has been blacklisted");
            }
            
            String[] parts = token.split("\\.");
            String header = new String(Base64.getUrlDecoder().decode(parts[0]));
            String payload = new String(Base64.getUrlDecoder().decode(parts[1]));
            String signature = parts[2];
            
            // Verify signature
            if (!verifySignature(header, payload, signature)) {
                throw new SecurityException("Invalid token signature");
            }
            
            // Parse claims
            JwtClaims claims = parseClaims(payload);
            
            // Check expiration
            if (claims.getExpirationTime() < System.currentTimeMillis()) {
                throw new SecurityException("Token has expired");
            }
            
            // Check issuer
            if (!issuer.equals(claims.getIssuer())) {
                throw new SecurityException("Invalid token issuer");
            }
            
            // Track token usage to prevent replay attacks
            String tokenHash = hashToken(token);
            int usageCount = tokenUsageCount.compute(tokenHash, (key, count) -> 
                count == null ? 1 : count + 1);
            
            if (usageCount > maxUsageCount) {
                blacklistToken(token);
                throw new SecurityException("Token usage limit exceeded");
            }
            
            return claims;
        }
        
        public void blacklistToken(String token) {
            blacklistedTokens.add(token);
            
            // Clean up old blacklisted tokens periodically
            if (blacklistedTokens.size() > 10000) { // Limit size
                blacklistedTokens.clear();
                tokenUsageCount.clear();
            }
        }
        
        private String createSignature(String header, String payload) throws Exception {
            String data = header + "." + payload;
            Mac mac = Mac.getInstance("HmacSHA512");
            SecretKeySpec secretKeySpec = new SecretKeySpec(secretKey.getBytes(), "HmacSHA512");
            mac.init(secretKeySpec);
            
            byte[] signatureBytes = mac.doFinal(data.getBytes());
            return Base64.getUrlEncoder().encodeToString(signatureBytes);
        }
        
        private boolean verifySignature(String header, String payload, String signature) {
            try {
                String expectedSignature = createSignature(header, payload);
                return MessageDigest.isEqual(
                    Base64.getUrlDecoder().decode(signature),
                    Base64.getUrlDecoder().decode(expectedSignature)
                );
            } catch (Exception e) {
                return false;
            }
        }
        
        private JwtClaims parseClaims(String payloadJson) {
            // Simple JSON parsing (in production, use a proper JSON library)
            Map<String, Object> claims = new HashMap<>();
            
            // Extract subject
            int subStart = payloadJson.indexOf("\"sub\":\"") + 7;
            int subEnd = payloadJson.indexOf("\"", subStart);
            String subject = payloadJson.substring(subStart, subEnd);
            
            // Extract issuer
            int issStart = payloadJson.indexOf("\"iss\":\"") + 7;
            int issEnd = payloadJson.indexOf("\"", issStart);
            String issuer = payloadJson.substring(issStart, issEnd);
            
            // Extract issued at
            int iatStart = payloadJson.indexOf("\"iat\":") + 6;
            int iatEnd = payloadJson.indexOf(",", iatStart);
            if (iatEnd == -1) iatEnd = payloadJson.indexOf("}", iatStart);
            long issuedAt = Long.parseLong(payloadJson.substring(iatStart, iatEnd));
            
            // Extract expiration
            int expStart = payloadJson.indexOf("\"exp\":") + 6;
            int expEnd = payloadJson.indexOf(",", expStart);
            if (expEnd == -1) expEnd = payloadJson.indexOf("}", expStart);
            long expiration = Long.parseLong(payloadJson.substring(expStart, expEnd));
            
            // Extract other claims
            // This is a simplified parser - use proper JSON parsing in production
            
            return new JwtClaims(subject, issuer, issuedAt, expiration, claims);
        }
        
        private String hashToken(String token) {
            try {
                MessageDigest digest = MessageDigest.getInstance("SHA-256");
                byte[] hash = digest.digest(token.getBytes());
                return Base64.getUrlEncoder().encodeToString(hash);
            } catch (NoSuchAlgorithmException e) {
                throw new RuntimeException("SHA-256 not available", e);
            }
        }
        
        // Inner class for JWT claims
        public static class JwtClaims {
            private final String subject;
            private final String issuer;
            private final long issuedAt;
            private final long expirationTime;
            private final Map<String, Object> claims;
            
            public JwtClaims(String subject, String issuer, long issuedAt, long expirationTime, 
                           Map<String, Object> claims) {
                this.subject = subject;
                this.issuer = issuer;
                this.issuedAt = issuedAt;
                this.expirationTime = expirationTime;
                this.claims = claims;
            }
            
            // Getters
            public String getSubject() { return subject; }
            public String getIssuer() { return issuer; }
            public long getIssuedAt() { return issuedAt; }
            public long getExpirationTime() { return expirationTime; }
            public Map<String, Object> getClaims() { return claims; }
            
            public Object getClaim(String key) { return claims.get(key); }
            public String getStringClaim(String key) { return (String) claims.get(key); }
            public Long getLongClaim(String key) { return (Long) claims.get(key); }
            public Boolean getBooleanClaim(String key) { return (Boolean) claims.get(key); }
        }
    }
    
    // Multi-factor authentication implementation
    public static class MultiFactorAuth {
        private final TOTPService totpService;
        private final EmailService emailService;
        private final SMSService smsService;
        private final Map<String, AuthSession> activeSessions = new ConcurrentHashMap<>();
        private final int maxAttempts = 5;
        private final long lockoutDuration = 30 * 60 * 1000; // 30 minutes
        
        public MultiFactorAuth(TOTPService totpService, EmailService emailService, SMSService smsService) {
            this.totpService = totpService;
            this.emailService = emailService;
            this.smsService = smsService;
        }
        
        public AuthResult authenticate(String username, String password, String mfaCode, 
                                     MFAType mfaType) {
            // First, authenticate user credentials
            if (!validateUserCredentials(username, password)) {
                recordFailedAttempt(username);
                return AuthResult.failed("Invalid username or password");
            }
            
            // Check account lockout
            AuthSession session = activeSessions.get(username);
            if (session != null && session.isLockedOut()) {
                return AuthResult.failed("Account is temporarily locked due to too many failed attempts");
            }
            
            // Validate MFA code
            boolean mfaValid = false;
            switch (mfaType) {
                case TOTP:
                    mfaValid = totpService.validateCode(username, mfaCode);
                    break;
                case SMS:
                    mfaValid = smsService.validateCode(username, mfaCode);
                    break;
                case EMAIL:
                    mfaValid = emailService.validateCode(username, mfaCode);
                    break;
            }
            
            if (mfaValid) {
                // Reset failed attempts counter
                if (session != null) {
                    session.resetFailedAttempts();
                }
                
                // Generate session token
                String sessionToken = generateSessionToken(username);
                return AuthResult.success(sessionToken);
            } else {
                recordFailedMfaAttempt(username);
                return AuthResult.failed("Invalid MFA code");
            }
        }
        
        public String generateAndSendMFA(String username, MFAType mfaType) {
            String code = generateMfaCode();
            
            switch (mfaType) {
                case SMS:
                    smsService.sendCode(username, code);
                    break;
                case EMAIL:
                    emailService.sendCode(username, code);
                    break;
                case TOTP:
                    // TOTP codes are generated on-demand, no need to send
                    return code;
            }
            
            // Store code for validation (with expiration)
            storeMfaCode(username, code, System.currentTimeMillis() + 5 * 60 * 1000); // 5 minutes
            
            return code;
        }
        
        private boolean validateUserCredentials(String username, String password) {
            // Implement secure credential validation
            // Use constant-time comparison to prevent timing attacks
            User user = getUserByUsername(username);
            if (user == null) return false;
            
            return securePasswordCompare(user.getHashedPassword(), password);
        }
        
        private void recordFailedAttempt(String username) {
            AuthSession session = activeSessions.computeIfAbsent(username, 
                k -> new AuthSession(username));
            session.incrementFailedAttempts();
        }
        
        private void recordFailedMfaAttempt(String username) {
            AuthSession session = activeSessions.computeIfAbsent(username, 
                k -> new AuthSession(username));
            session.incrementMfaFailedAttempts();
        }
        
        private boolean securePasswordCompare(String hashedPassword, String password) {
            // Implement secure password comparison
            // This is a simplified example - use proper Bcrypt comparison in production
            return java.util.Arrays.equals(
                hashedPassword.getBytes(),
                hashPassword(password).getBytes()
            );
        }
        
        private String generateSessionToken(String username) {
            // Generate secure session token
            String tokenData = username + ":" + System.currentTimeMillis() + ":" + 
                              java.util.UUID.randomUUID().toString();
            return java.util.Base64.getEncoder().encodeToString(
                java.security.MessageDigest.getInstance("SHA-256")
                    .digest(tokenData.getBytes())
            );
        }
        
        private String generateMfaCode() {
            // Generate 6-digit numeric code
            Random random = new Random();
            return String.format("%06d", random.nextInt(1000000));
        }
        
        private void storeMfaCode(String username, String code, long expiration) {
            // Store MFA code with expiration in a secure, temporary storage
            // In production, use Redis or similar with automatic expiration
        }
        
        private User getUserByUsername(String username) {
            // Retrieve user from database
            return new User(username, "hashed_password"); // Simplified
        }
        
        private String hashPassword(String password) {
            // Use proper password hashing (Bcrypt, Argon2, etc.)
            return password; // Simplified for example
        }
        
        public enum MFAType {
            TOTP, SMS, EMAIL
        }
        
        public static class AuthResult {
            private final boolean success;
            private final String sessionToken;
            private final String errorMessage;
            
            private AuthResult(boolean success, String sessionToken, String errorMessage) {
                this.success = success;
                this.sessionToken = sessionToken;
                this.errorMessage = errorMessage;
            }
            
            public static AuthResult success(String sessionToken) {
                return new AuthResult(true, sessionToken, null);
            }
            
            public static AuthResult failed(String errorMessage) {
                return new AuthResult(false, null, errorMessage);
            }
            
            public boolean isSuccess() { return success; }
            public String getSessionToken() { return sessionToken; }
            public String getErrorMessage() { return errorMessage; }
        }
        
        private static class AuthSession {
            private final String username;
            private AtomicInteger failedAttempts = new AtomicInteger(0);
            private AtomicInteger mfaFailedAttempts = new AtomicInteger(0);
            private long lastFailedAttempt = 0;
            
            public AuthSession(String username) {
                this.username = username;
            }
            
            public void incrementFailedAttempts() {
                failedAttempts.incrementAndGet();
                lastFailedAttempt = System.currentTimeMillis();
            }
            
            public void incrementMfaFailedAttempts() {
                mfaFailedAttempts.incrementAndGet();
                lastFailedAttempt = System.currentTimeMillis();
            }
            
            public void resetFailedAttempts() {
                failedAttempts.set(0);
                mfaFailedAttempts.set(0);
            }
            
            public boolean isLockedOut() {
                return failedAttempts.get() >= 5 && 
                       (System.currentTimeMillis() - lastFailedAttempt) < (30 * 60 * 1000);
            }
        }
    }
    
    // TOTP (Time-based One-Time Password) service
    public static class TOTPService {
        private final Map<String, String> userSecrets = new ConcurrentHashMap<>();
        
        public String generateSecret(String username) {
            // Generate a random secret key for the user
            byte[] secretBytes = new byte[20];
            new SecureRandom().nextBytes(secretBytes);
            String secret = Base32.encode(secretBytes);
            userSecrets.put(username, secret);
            return secret;
        }
        
        public boolean validateCode(String username, String code) {
            String secret = userSecrets.get(username);
            if (secret == null) return false;
            
            try {
                // Validate code using current time window
                long currentCode = generateCode(secret, System.currentTimeMillis() / 1000);
                long previousCode = generateCode(secret, (System.currentTimeMillis() - 30000) / 1000);
                long nextCode = generateCode(secret, (System.currentTimeMillis() + 30000) / 1000);
                
                return String.valueOf(currentCode).equals(code) ||
                       String.valueOf(previousCode).equals(code) ||
                       String.valueOf(nextCode).equals(code);
            } catch (Exception e) {
                return false;
            }
        }
        
        private long generateCode(String secret, long timeIndex) throws Exception {
            // Implement TOTP algorithm (RFC 6238)
            byte[] key = Base32.decode(secret);
            byte[] data = new byte[8];
            
            long value = timeIndex / 30; // 30-second time step
            for (int i = 8; i-- > 0; value >>>= 8) {
                data[i] = (byte) value;
            }
            
            Mac mac = Mac.getInstance("HmacSHA1");
            SecretKeySpec spec = new SecretKeySpec(key, "HmacSHA1");
            mac.init(spec);
            byte[] hash = mac.doFinal(data);
            
            int offset = hash[20 - 1] & 0xF;
            long truncatedHash = hash[offset] & 0x7F;
            for (int i = 1; i < 4; i++) {
                truncatedHash <<= 8;
                truncatedHash |= hash[offset + i] & 0xFF;
            }
            
            truncatedHash %= 1000000; // 6-digit code
            return truncatedHash;
        }
        
        // Base32 encoding/decoding utilities
        private static class Base32 {
            private static final String CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567";
            
            public static String encode(byte[] data) {
                StringBuilder result = new StringBuilder();
                int buffer = data[0] & 0xFF;
                int next = 1;
                int bitsLeft = 8;
                
                while (bitsLeft > 0 || next < data.length) {
                    if (bitsLeft < 5) {
                        if (next < data.length) {
                            buffer <<= 8;
                            buffer |= (data[next++] & 0xFF);
                            bitsLeft += 8;
                        } else {
                            int pad = 5 - bitsLeft;
                            buffer <<= pad;
                            bitsLeft += pad;
                        }
                    }
                    
                    int index = (buffer >> (bitsLeft - 5)) & 0x1F;
                    result.append(CHARS.charAt(index));
                    bitsLeft -= 5;
                }
                
                return result.toString();
            }
            
            public static byte[] decode(String encoded) {
                // Simplified decoder - use proper implementation in production
                return encoded.getBytes();
            }
        }
    }
    
    // Supporting services
    public static class EmailService {
        public void sendCode(String username, String code) {
            // Send MFA code via email
            System.out.println("Sending MFA code " + code + " to user " + username + " via email");
        }
        
        public boolean validateCode(String username, String code) {
            // Validate the code sent via email
            return true; // Simplified
        }
    }
    
    public static class SMSService {
        public void sendCode(String username, String code) {
            // Send MFA code via SMS
            System.out.println("Sending MFA code " + code + " to user " + username + " via SMS");
        }
        
        public boolean validateCode(String username, String code) {
            // Validate the code sent via SMS
            return true; // Simplified
        }
    }
    
    public static class User {
        private String username;
        private String hashedPassword;
        
        public User(String username, String hashedPassword) {
            this.username = username;
            this.hashedPassword = hashedPassword;
        }
        
        // Getters
        public String getUsername() { return username; }
        public String getHashedPassword() { return hashedPassword; }
    }
}
```

## Authorization and Access Control

### Advanced Authorization Framework

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Predicate;
import java.util.stream.Collectors;

public class AdvancedAuthorization {
    
    // Role-Based Access Control (RBAC) implementation
    public static class RoleBasedAccessControl {
        private final Map<String, Set<String>> userRoles = new ConcurrentHashMap<>();
        private final Map<String, Set<String>> rolePermissions = new ConcurrentHashMap<>();
        private final Map<String, Set<String>> userPermissions = new ConcurrentHashMap<>();
        private final Map<String, Resource> resources = new ConcurrentHashMap<>();
        
        // Add role to user
        public void assignRole(String username, String role) {
            userRoles.computeIfAbsent(username, k -> ConcurrentHashMap.newKeySet()).add(role);
            recalculateUserPermissions(username);
        }
        
        // Remove role from user
        public void removeRole(String username, String role) {
            Set<String> roles = userRoles.get(username);
            if (roles != null) {
                roles.remove(role);
                recalculateUserPermissions(username);
            }
        }
        
        // Add permission to role
        public void assignPermission(String role, String permission) {
            rolePermissions.computeIfAbsent(role, k -> ConcurrentHashMap.newKeySet()).add(permission);
            // Update all users with this role
            updateUserPermissionsWithRole(role);
        }
        
        // Remove permission from role
        public void removePermission(String role, String permission) {
            Set<String> permissions = rolePermissions.get(role);
            if (permissions != null) {
                permissions.remove(permission);
                // Update all users with this role
                updateUserPermissionsWithRole(role);
            }
        }
        
        // Check if user has permission
        public boolean hasPermission(String username, String resource, String action) {
            String permission = resource + ":" + action;
            
            // Check user's direct permissions
            Set<String> userPerms = userPermissions.get(username);
            if (userPerms != null && userPerms.contains(permission)) {
                return true;
            }
            
            // Check through roles
            Set<String> roles = userRoles.get(username);
            if (roles != null) {
                for (String role : roles) {
                    Set<String> rolePerms = rolePermissions.get(role);
                    if (rolePerms != null && rolePerms.contains(permission)) {
                        return true;
                    }
                }
            }
            
            return false;
        }
        
        // Check with resource ownership
        public boolean hasPermission(String username, String resource, String action, 
                                   String resourceOwner) {
            if (username.equals(resourceOwner)) {
                // User is the owner, allow all actions
                return true;
            }
            
            return hasPermission(username, resource, action);
        }
        
        // Bulk permission check
        public Map<String, Boolean> checkPermissions(String username, 
                                                   List<ResourceAction> resourceActions) {
            return resourceActions.stream()
                .collect(Collectors.toMap(
                    ResourceAction::toString,
                    ra -> hasPermission(username, ra.getResource(), ra.getAction())
                ));
        }
        
        // Hierarchical role support
        public void addRoleHierarchy(String parentRole, String childRole) {
            // Child role inherits all permissions from parent role
            Set<String> parentPerms = rolePermissions.get(parentRole);
            if (parentPerms != null) {
                rolePermissions.computeIfAbsent(childRole, k -> ConcurrentHashMap.newKeySet())
                              .addAll(parentPerms);
            }
        }
        
        // Permission inheritance based on resource hierarchy
        public void addResourceHierarchy(String parentResource, String childResource) {
            resources.put(childResource, new Resource(childResource, parentResource));
        }
        
        private void recalculateUserPermissions(String username) {
            Set<String> permissions = ConcurrentHashMap.newKeySet();
            
            // Add permissions from roles
            Set<String> roles = userRoles.get(username);
            if (roles != null) {
                for (String role : roles) {
                    Set<String> rolePerms = rolePermissions.get(role);
                    if (rolePerms != null) {
                        permissions.addAll(rolePerms);
                    }
                }
            }
            
            userPermissions.put(username, permissions);
        }
        
        private void updateUserPermissionsWithRole(String role) {
            for (String username : userRoles.keySet()) {
                if (userRoles.get(username).contains(role)) {
                    recalculateUserPermissions(username);
                }
            }
        }
        
        // Resource class for hierarchical permissions
        public static class Resource {
            private final String name;
            private final String parentResource;
            
            public Resource(String name, String parentResource) {
                this.name = name;
                this.parentResource = parentResource;
            }
            
            public String getName() { return name; }
            public String getParentResource() { return parentResource; }
        }
        
        public static class ResourceAction {
            private final String resource;
            private final String action;
            
            public ResourceAction(String resource, String action) {
                this.resource = resource;
                this.action = action;
            }
            
            public String getResource() { return resource; }
            public String getAction() { return action; }
            
            @Override
            public String toString() {
                return resource + ":" + action;
            }
        }
    }
    
    // Attribute-Based Access Control (ABAC) implementation
    public static class AttributeBasedAccessControl {
        private final List<Policy> policies = new ArrayList<>();
        
        public void addPolicy(Policy policy) {
            policies.add(policy);
        }
        
        public boolean evaluate(String subject, String resource, String action, 
                              Map<String, Object> environment) {
            for (Policy policy : policies) {
                if (policy.matches(subject, resource, action, environment)) {
                    return policy.getEffect() == Policy.Effect.PERMIT;
                }
            }
            return false; // Default deny
        }
        
        // Policy class
        public static class Policy {
            public enum Effect { PERMIT, DENY }
            
            private final String id;
            private final Effect effect;
            private final List<Condition> conditions;
            
            public Policy(String id, Effect effect, List<Condition> conditions) {
                this.id = id;
                this.effect = effect;
                this.conditions = conditions != null ? new ArrayList<>(conditions) : new ArrayList<>();
            }
            
            public boolean matches(String subject, String resource, String action, 
                                 Map<String, Object> environment) {
                for (Condition condition : conditions) {
                    if (!condition.evaluate(subject, resource, action, environment)) {
                        return false;
                    }
                }
                return true;
            }
            
            public Effect getEffect() { return effect; }
            public String getId() { return id; }
        }
        
        // Condition interface
        public interface Condition {
            boolean evaluate(String subject, String resource, String action, 
                           Map<String, Object> environment);
        }
        
        // Built-in conditions
        public static class SubjectCondition implements Condition {
            private final String attribute;
            private final Object expectedValue;
            
            public SubjectCondition(String attribute, Object expectedValue) {
                this.attribute = attribute;
                this.expectedValue = expectedValue;
            }
            
            @Override
            public boolean evaluate(String subject, String resource, String action, 
                                  Map<String, Object> environment) {
                // In a real implementation, this would look up subject attributes
                return true; // Simplified
            }
        }
        
        public static class ResourceCondition implements Condition {
            private final String attribute;
            private final Object expectedValue;
            
            public ResourceCondition(String attribute, Object expectedValue) {
                this.attribute = attribute;
                this.expectedValue = expectedValue;
            }
            
            @Override
            public boolean evaluate(String subject, String resource, String action, 
                                  Map<String, Object> environment) {
                // Check resource attributes
                return true; // Simplified
            }
        }
        
        public static class ActionCondition implements Condition {
            private final String expectedAction;
            private final boolean match;
            
            public ActionCondition(String expectedAction, boolean match) {
                this.expectedAction = expectedAction;
                this.match = match;
            }
            
            @Override
            public boolean evaluate(String subject, String resource, String action, 
                                  Map<String, Object> environment) {
                boolean matches = expectedAction.equals(action);
                return match ? matches : !matches;
            }
        }
        
        public static class TimeCondition implements Condition {
            private final long startTime;
            private final long endTime;
            
            public TimeCondition(long startTime, long endTime) {
                this.startTime = startTime;
                this.endTime = endTime;
            }
            
            @Override
            public boolean evaluate(String subject, String resource, String action, 
                                  Map<String, Object> environment) {
                long currentTime = System.currentTimeMillis();
                return currentTime >= startTime && currentTime <= endTime;
            }
        }
    }
    
    // Access control decision point
    public static class AccessControlDecisionPoint {
        private final RoleBasedAccessControl rbac;
        private final AttributeBasedAccessControl abac;
        
        public AccessControlDecisionPoint() {
            this.rbac = new RoleBasedAccessControl();
            this.abac = new AttributeBasedAccessControl();
        }
        
        public AuthorizationResult checkAccess(String subject, String resource, String action) {
            return checkAccess(subject, resource, action, new HashMap<>());
        }
        
        public AuthorizationResult checkAccess(String subject, String resource, String action,
                                             Map<String, Object> environment) {
            // First check RBAC
            boolean rbacAllowed = rbac.hasPermission(subject, resource, action);
            
            // Then check ABAC
            boolean abacAllowed = abac.evaluate(subject, resource, action, environment);
            
            // Combine results (ABAC can override RBAC in some cases)
            boolean allowed = rbacAllowed || abacAllowed;
            
            return new AuthorizationResult(allowed, rbacAllowed, abacAllowed);
        }
        
        // Context-aware access control
        public AuthorizationResult checkAccessWithContext(String subject, String resource, String action,
                                                       AccessContext context) {
            // Enhance environment with context
            Map<String, Object> environment = new HashMap<>();
            environment.put("timestamp", System.currentTimeMillis());
            environment.put("ipAddress", context.getIpAddress());
            environment.put("userAgent", context.getUserAgent());
            environment.put("location", context.getLocation());
            
            return checkAccess(subject, resource, action, environment);
        }
        
        public static class AuthorizationResult {
            private final boolean allowed;
            private final boolean rbacResult;
            private final boolean abacResult;
            
            public AuthorizationResult(boolean allowed, boolean rbacResult, boolean abacResult) {
                this.allowed = allowed;
                this.rbacResult = rbacResult;
                this.abacResult = abacResult;
            }
            
            public boolean isAllowed() { return allowed; }
            public boolean getRbacResult() { return rbacResult; }
            public boolean getAbacResult() { return abacResult; }
        }
        
        public static class AccessContext {
            private final String ipAddress;
            private final String userAgent;
            private final String location;
            private final Map<String, String> additionalContext;
            
            public AccessContext(String ipAddress, String userAgent, String location) {
                this.ipAddress = ipAddress;
                this.userAgent = userAgent;
                this.location = location;
                this.additionalContext = new HashMap<>();
            }
            
            public String getIpAddress() { return ipAddress; }
            public String getUserAgent() { return userAgent; }
            public String getLocation() { return location; }
            
            public void addContext(String key, String value) {
                additionalContext.put(key, value);
            }
            
            public String getContext(String key) {
                return additionalContext.get(key);
            }
        }
    }
}
```

## Cryptography and Data Protection

### Advanced Cryptographic Implementation

```java
import java.security.*;
import java.security.spec.*;
import javax.crypto.*;
import javax.crypto.spec.*;
import java.util.Base64;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Consumer;

public class AdvancedCryptography {
    
    // Secure key management system
    public static class KeyManagementSystem {
        private final Map<String, SecretKey> keys = new ConcurrentHashMap<>();
        private final String keystorePath;
        private final char[] keystorePassword;
        
        public KeyManagementSystem(String keystorePath, String keystorePassword) {
            this.keystorePath = keystorePath;
            this.keystorePassword = keystorePassword != null ? keystorePassword.toCharArray() : null;
        }
        
        // Generate and store a key
        public void generateAndStoreKey(String keyId, String algorithm, int keySize) throws Exception {
            KeyGenerator keyGen = KeyGenerator.getInstance(algorithm);
            keyGen.init(keySize);
            SecretKey key = keyGen.generateKey();
            keys.put(keyId, key);
        }
        
        // Generate key with specific parameters
        public void generateKeyWithParameters(String keyId, String algorithm, AlgorithmParameterSpec spec) 
                throws Exception {
            KeyGenerator keyGen = KeyGenerator.getInstance(algorithm);
            keyGen.init(spec, SecureRandom.getInstanceStrong());
            SecretKey key = keyGen.generateKey();
            keys.put(keyId, key);
        }
        
        // Retrieve key
        public SecretKey getKey(String keyId) {
            return keys.get(keyId);
        }
        
        // Rotate key
        public void rotateKey(String keyId, String newKeyId, String algorithm, int keySize) 
                throws Exception {
            // Generate new key
            generateAndStoreKey(newKeyId, algorithm, keySize);
            
            // Optionally, keep old key for a period to decrypt existing data
            // This would typically be handled with a key rotation schedule
        }
        
        // Secure key derivation
        public SecretKey deriveKey(String baseKey, String salt, String purpose) throws Exception {
            // Use PBKDF2 for key derivation
            SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
            KeySpec spec = new PBEKeySpec(baseKey.toCharArray(), 
                                        salt.getBytes(), 100000, 256); // 256-bit key
            SecretKey tmp = factory.generateSecret(spec);
            return new SecretKeySpec(tmp.getEncoded(), "AES");
        }
        
        // Key wrapping/unwrapping for secure key transport
        public byte[] wrapKey(SecretKey keyToWrap, SecretKey wrappingKey) throws Exception {
            Cipher cipher = Cipher.getInstance("AES");
            cipher.init(Cipher.WRAP_MODE, wrappingKey);
            return cipher.wrap(keyToWrap);
        }
        
        public SecretKey unwrapKey(byte[] wrappedKey, SecretKey wrappingKey, 
                                 String algorithm) throws Exception {
            Cipher cipher = Cipher.getInstance("AES");
            cipher.init(Cipher.UNWRAP_MODE, wrappingKey);
            return (SecretKey) cipher.unwrap(wrappedKey, algorithm, Cipher.SECRET_KEY);
        }
        
        // Key lifecycle management
        public void scheduleKeyRotation(String keyId, long rotationPeriodMs) {
            // In production, use a proper task scheduler
            new Thread(() -> {
                try {
                    Thread.sleep(rotationPeriodMs);
                    // Trigger key rotation
                    System.out.println("Key rotation scheduled for: " + keyId);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
    
    // Advanced encryption service
    public static class EncryptionService {
        private final KeyManagementSystem keyManager;
        
        public EncryptionService(KeyManagementSystem keyManager) {
            this.keyManager = keyManager;
        }
        
        // AES-GCM encryption with authentication
        public EncryptedData encryptAESGCM(String plaintext, String keyId) throws Exception {
            SecretKey key = keyManager.getKey(keyId);
            if (key == null) {
                throw new IllegalArgumentException("Key not found: " + keyId);
            }
            
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            GCMParameterSpec parameterSpec = new GCMParameterSpec(128, generateNonce(12)); // 12-byte nonce
            
            cipher.init(Cipher.ENCRYPT_MODE, key, parameterSpec);
            byte[] ciphertext = cipher.doFinal(plaintext.getBytes("UTF-8"));
            
            // Combine nonce and ciphertext
            byte[] result = new byte[parameterSpec.getTLen()/8 + ciphertext.length];
            System.arraycopy(parameterSpec.getIV(), 0, result, 0, parameterSpec.getIV().length);
            System.arraycopy(ciphertext, 0, result, parameterSpec.getIV().length, ciphertext.length);
            
            return new EncryptedData(result, parameterSpec.getIV(), 
                                   cipher.getParameters().getParameterSpec(GCMParameterSpec.class));
        }
        
        // AES-GCM decryption
        public String decryptAESGCM(EncryptedData encryptedData, String keyId) throws Exception {
            SecretKey key = keyManager.getKey(keyId);
            if (key == null) {
                throw new IllegalArgumentException("Key not found: " + keyId);
            }
            
            byte[] combined = encryptedData.getEncryptedData();
            byte[] nonce = encryptedData.getNonce();
            byte[] ciphertext = new byte[combined.length - nonce.length];
            System.arraycopy(combined, nonce.length, ciphertext, 0, ciphertext.length);
            
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            GCMParameterSpec parameterSpec = new GCMParameterSpec(128, nonce);
            
            cipher.init(Cipher.DECRYPT_MODE, key, parameterSpec);
            return new String(cipher.doFinal(ciphertext), "UTF-8");
        }
        
        // RSA encryption for key wrapping
        public byte[] encryptRSA(byte[] data, PublicKey publicKey) throws Exception {
            Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
            cipher.init(Cipher.ENCRYPT_MODE, publicKey);
            return cipher.doFinal(data);
        }
        
        public byte[] decryptRSA(byte[] encryptedData, PrivateKey privateKey) throws Exception {
            Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
            cipher.init(Cipher.DECRYPT_MODE, privateKey);
            return cipher.doFinal(encryptedData);
        }
        
        // Hybrid encryption (AES for data, RSA for key)
        public HybridEncryptedData hybridEncrypt(String plaintext, PublicKey rsaPublicKey, 
                                               String aesKeyId) throws Exception {
            // Generate AES key for this operation
            SecretKey aesKey = generateAESKey();
            
            // Encrypt data with AES
            EncryptedData aesEncrypted = encryptAESGCM(plaintext, aesKeyId);
            
            // Encrypt AES key with RSA
            byte[] encryptedAESKey = encryptRSA(aesKey.getEncoded(), rsaPublicKey);
            
            return new HybridEncryptedData(aesEncrypted.getEncryptedData(), encryptedAESKey);
        }
        
        // Secure random generation
        public byte[] generateSecureRandom(int size) {
            byte[] randomBytes = new byte[size];
            new SecureRandom().nextBytes(randomBytes);
            return randomBytes;
        }
        
        private byte[] generateNonce(int size) {
            return generateSecureRandom(size);
        }
        
        private SecretKey generateAESKey() throws Exception {
            KeyGenerator keyGen = KeyGenerator.getInstance("AES");
            keyGen.init(256);
            return keyGen.generateKey();
        }
        
        // Data integrity verification
        public String calculateHMAC(String data, String keyId) throws Exception {
            SecretKey key = keyManager.getKey(keyId);
            if (key == null) {
                throw new IllegalArgumentException("Key not found: " + keyId);
            }
            
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(key);
            byte[] hmac = mac.doFinal(data.getBytes("UTF-8"));
            return Base64.getEncoder().encodeToString(hmac);
        }
        
        public boolean verifyHMAC(String data, String expectedHMAC, String keyId) throws Exception {
            String calculatedHMAC = calculateHMAC(data, keyId);
            return MessageDigest.isEqual(
                expectedHMAC.getBytes(),
                calculatedHMAC.getBytes()
            );
        }
        
        // File encryption
        public void encryptFile(java.io.File inputFile, java.io.File outputFile, String keyId) 
                throws Exception {
            SecretKey key = keyManager.getKey(keyId);
            
            try (java.io.FileInputStream fis = new java.io.FileInputStream(inputFile);
                 java.io.FileOutputStream fos = new java.io.FileOutputStream(outputFile)) {
                
                Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
                GCMParameterSpec parameterSpec = new GCMParameterSpec(128, generateNonce(12));
                cipher.init(Cipher.ENCRYPT_MODE, key, parameterSpec);
                
                // Write IV first
                fos.write(parameterSpec.getIV());
                
                // Encrypt and write data in chunks
                byte[] buffer = new byte[8192];
                int bytesRead;
                while ((bytesRead = fis.read(buffer)) != -1) {
                    byte[] output = cipher.update(buffer, 0, bytesRead);
                    if (output != null) {
                        fos.write(output);
                    }
                }
                
                byte[] finalOutput = cipher.doFinal();
                if (finalOutput != null) {
                    fos.write(finalOutput);
                }
            }
        }
        
        // File decryption
        public void decryptFile(java.io.File inputFile, java.io.File outputFile, String keyId) 
                throws Exception {
            SecretKey key = keyManager.getKey(keyId);
            
            try (java.io.FileInputStream fis = new java.io.FileInputStream(inputFile);
                 java.io.FileOutputStream fos = new java.io.FileOutputStream(outputFile)) {
                
                // Read IV (first 12 bytes)
                byte[] iv = new byte[12];
                int bytesRead = fis.read(iv);
                if (bytesRead != 12) {
                    throw new java.io.IOException("Invalid file format");
                }
                
                Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
                GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
                cipher.init(Cipher.DECRYPT_MODE, key, parameterSpec);
                
                // Decrypt data in chunks
                byte[] buffer = new byte[8192];
                while ((bytesRead = fis.read(buffer)) != -1) {
                    byte[] output = cipher.update(buffer, 0, bytesRead);
                    if (output != null) {
                        fos.write(output);
                    }
                }
                
                byte[] finalOutput = cipher.doFinal();
                if (finalOutput != null) {
                    fos.write(finalOutput);
                }
            }
        }
        
        // Supporting classes
        public static class EncryptedData {
            private final byte[] encryptedData;
            private final byte[] nonce;
            private final GCMParameterSpec parameterSpec;
            
            public EncryptedData(byte[] encryptedData, byte[] nonce, GCMParameterSpec parameterSpec) {
                this.encryptedData = encryptedData;
                this.nonce = nonce;
                this.parameterSpec = parameterSpec;
            }
            
            public byte[] getEncryptedData() { return encryptedData; }
            public byte[] getNonce() { return nonce; }
            public GCMParameterSpec getParameterSpec() { return parameterSpec; }
        }
        
        public static class HybridEncryptedData {
            private final byte[] encryptedData;
            private final byte[] encryptedKey;
            
            public HybridEncryptedData(byte[] encryptedData, byte[] encryptedKey) {
                this.encryptedData = encryptedData;
                this.encryptedKey = encryptedKey;
            }
            
            public byte[] getEncryptedData() { return encryptedData; }
            public byte[] getEncryptedKey() { return encryptedKey; }
        }
    }
    
    // Password hashing service
    public static class PasswordHashingService {
        private final KeyManagementSystem keyManager;
        
        public PasswordHashingService(KeyManagementSystem keyManager) {
            this.keyManager = keyManager;
        }
        
        // Use Argon2 for password hashing (if available) or fall back to BCrypt
        public String hashPassword(String password) throws Exception {
            // In a real implementation, use Argon2 or BCrypt
            // For this example, we'll use a simplified approach with PBKDF2
            byte[] salt = generateSalt();
            byte[] hash = pbkdf2(password.toCharArray(), salt, 100000, 256);
            
            // Combine salt and hash
            byte[] result = new byte[salt.length + hash.length];
            System.arraycopy(salt, 0, result, 0, salt.length);
            System.arraycopy(hash, 0, result, salt.length, hash.length);
            
            return Base64.getEncoder().encodeToString(result);
        }
        
        public boolean verifyPassword(String password, String hashedPassword) throws Exception {
            byte[] combined = Base64.getDecoder().decode(hashedPassword);
            byte[] salt = new byte[16]; // Assuming 16-byte salt
            System.arraycopy(combined, 0, salt, 0, salt.length);
            
            byte[] storedHash = new byte[combined.length - salt.length];
            System.arraycopy(combined, salt.length, storedHash, 0, storedHash.length);
            
            byte[] calculatedHash = pbkdf2(password.toCharArray(), salt, 100000, 256);
            
            return MessageDigest.isEqual(storedHash, calculatedHash);
        }
        
        private byte[] generateSalt() {
            byte[] salt = new byte[16];
            new SecureRandom().nextBytes(salt);
            return salt;
        }
        
        private byte[] pbkdf2(char[] password, byte[] salt, int iterations, int keyLength) 
                throws Exception {
            SecretKeyFactory factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
            KeySpec spec = new PBEKeySpec(password, salt, iterations, keyLength);
            SecretKey tmp = factory.generateSecret(spec);
            return tmp.getEncoded();
        }
    }
}
```

## Secure Coding Practices

### Input Validation and Sanitization

```java
import java.util.*;
import java.util.regex.Pattern;
import java.util.stream.Collectors;

public class SecureCodingPractices {
    
    // Advanced input validation framework
    public static class InputValidationFramework {
        private final List<ValidationRule> globalRules = new ArrayList<>();
        private final Pattern sqlInjectionPattern = Pattern.compile(
            "(?i)(union|select|insert|update|delete|drop|create|alter|exec|execute|sp_|xp_|0x|'|--|;|/\\*|\\*/|\\b(OR|AND)\\b\\s*['\"=])"
        );
        private final Pattern xssPattern = Pattern.compile(
            "(?i)<script[^>]*>.*?</script>|<img[^>]*src=[^>]*>|<a[^>]*href=[^>]*>|javascript:|on\\w+\\s*="
        );
        private final Pattern pathTraversalPattern = Pattern.compile(
            "(\\.\\.\\/|\\.\\.\\\\|%2e%2e%2f|%2e%2e%5c|%252e%252e%252f|%252e%252e%255c)"
        );
        
        public InputValidationFramework() {
            // Add global security rules
            globalRules.add(new SQLInjectionRule());
            globalRules.add(new XSSRule());
            globalRules.add(new PathTraversalRule());
        }
        
        public ValidationResult validate(String input, ValidationType type) {
            return validate(input, Collections.singletonList(type));
        }
        
        public ValidationResult validate(String input, List<ValidationType> types) {
            List<String> errors = new ArrayList<>();
            
            // Apply global rules
            for (ValidationRule rule : globalRules) {
                if (!rule.validate(input)) {
                    errors.add(rule.getErrorMessage());
                }
            }
            
            // Apply specific type validation
            for (ValidationType type : types) {
                switch (type) {
                    case EMAIL:
                        if (!isValidEmail(input)) errors.add("Invalid email format");
                        break;
                    case PHONE:
                        if (!isValidPhone(input)) errors.add("Invalid phone format");
                        break;
                    case URL:
                        if (!isValidUrl(input)) errors.add("Invalid URL format");
                        break;
                    case JSON:
                        if (!isValidJson(input)) errors.add("Invalid JSON format");
                        break;
                    case XML:
                        if (!isValidXml(input)) errors.add("Invalid XML format");
                        break;
                }
            }
            
            return new ValidationResult(errors.isEmpty(), errors);
        }
        
        // Sanitize input
        public String sanitize(String input, SanitizationType type) {
            if (input == null) return null;
            
            switch (type) {
                case HTML:
                    return sanitizeHtml(input);
                case SQL:
                    return sanitizeSql(input);
                case URL:
                    return sanitizeUrl(input);
                case JAVASCRIPT:
                    return sanitizeJavaScript(input);
                default:
                    return input;
            }
        }
        
        // Batch validation
        public Map<String, ValidationResult> validateBatch(Map<String, List<ValidationType>> inputs) {
            return inputs.entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    entry -> validate(entry.getKey(), entry.getValue())
                ));
        }
        
        // Custom validation rules
        public void addCustomRule(ValidationRule rule) {
            globalRules.add(rule);
        }
        
        private boolean isValidEmail(String email) {
            if (email == null || email.isEmpty()) return false;
            return email.matches("^[A-Za-z0-9+_.-]+@([A-Za-z0-9.-]+\\.[A-Za-z]{2,})$");
        }
        
        private boolean isValidPhone(String phone) {
            if (phone == null || phone.isEmpty()) return false;
            return phone.matches("^\\+?[1-9]\\d{1,14}$"); // E.164 format
        }
        
        private boolean isValidUrl(String url) {
            if (url == null || url.isEmpty()) return false;
            try {
                new java.net.URL(url);
                return true;
            } catch (java.net.MalformedURLException e) {
                return false;
            }
        }
        
        private boolean isValidJson(String json) {
            if (json == null || json.isEmpty()) return false;
            try {
                com.fasterxml.jackson.databind.ObjectMapper mapper = 
                    new com.fasterxml.jackson.databind.ObjectMapper();
                mapper.readTree(json);
                return true;
            } catch (Exception e) {
                return false;
            }
        }
        
        private boolean isValidXml(String xml) {
            if (xml == null || xml.isEmpty()) return false;
            try {
                javax.xml.parsers.DocumentBuilderFactory factory = 
                    javax.xml.parsers.DocumentBuilderFactory.newInstance();
                factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
                factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
                factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
                
                javax.xml.parsers.DocumentBuilder builder = factory.newDocumentBuilder();
                builder.parse(new java.io.ByteArrayInputStream(xml.getBytes()));
                return true;
            } catch (Exception e) {
                return false;
            }
        }
        
        private String sanitizeHtml(String input) {
            // Use a proper HTML sanitizer library in production
            // For example: OWASP Java HTML Sanitizer or Jsoup
            return input.replaceAll("<[^>]*>", ""); // Very basic sanitization
        }
        
        private String sanitizeSql(String input) {
            return input.replaceAll("(?i)(union|select|insert|update|delete|drop|create|alter|exec|execute|sp_|xp_)", "");
        }
        
        private String sanitizeUrl(String input) {
            try {
                java.net.URI uri = new java.net.URI(input);
                return uri.toString();
            } catch (Exception e) {
                return "";
            }
        }
        
        private String sanitizeJavaScript(String input) {
            return input.replaceAll("(?i)javascript:", "");
        }
        
        // Validation enums and classes
        public enum ValidationType {
            EMAIL, PHONE, URL, JSON, XML, SQL, HTML
        }
        
        public enum SanitizationType {
            HTML, SQL, URL, JAVASCRIPT
        }
        
        public interface ValidationRule {
            boolean validate(String input);
            String getErrorMessage();
        }
        
        public static class SQLInjectionRule implements ValidationRule {
            @Override
            public boolean validate(String input) {
                return !Pattern.compile("(?i)(union|select|insert|update|delete|drop|create|alter|exec|execute|sp_|xp_|0x|'|--|;|/\\*|\\*/|\\b(OR|AND)\\b\\s*['\"=])")
                    .matcher(input).find();
            }
            
            @Override
            public String getErrorMessage() {
                return "Potential SQL injection detected";
            }
        }
        
        public static class XSSRule implements ValidationRule {
            @Override
            public boolean validate(String input) {
                return !Pattern.compile("(?i)<script[^>]*>.*?</script>|<img[^>]*src=[^>]*>|<a[^>]*href=[^>]*>|javascript:|on\\w+\\s*=")
                    .matcher(input).find();
            }
            
            @Override
            public String getErrorMessage() {
                return "Potential XSS attack detected";
            }
        }
        
        public static class PathTraversalRule implements ValidationRule {
            @Override
            public boolean validate(String input) {
                return !Pattern.compile("(\\.\\.\\/|\\.\\.\\\\|%2e%2e%2f|%2e%2e%5c|%252e%252e%252f|%252e%252e%255c)")
                    .matcher(input).find();
            }
            
            @Override
            public String getErrorMessage() {
                return "Potential path traversal attack detected";
            }
        }
        
        public static class ValidationResult {
            private final boolean valid;
            private final List<String> errors;
            
            public ValidationResult(boolean valid, List<String> errors) {
                this.valid = valid;
                this.errors = errors != null ? new ArrayList<>(errors) : new ArrayList<>();
            }
            
            public boolean isValid() { return valid; }
            public List<String> getErrors() { return new ArrayList<>(errors); }
            public String getErrorString() { return String.join(", ", errors); }
        }
    }
    
    // Output encoding for preventing XSS
    public static class OutputEncoder {
        
        public static String encodeHtml(String input) {
            if (input == null) return null;
            
            return input.replace("&", "&amp;")
                       .replace("<", "&lt;")
                       .replace(">", "&gt;")
                       .replace("\"", "&quot;")
                       .replace("'", "&#x27;")
                       .replace("/", "&#x2F;");
        }
        
        public static String encodeHtmlAttribute(String input) {
            if (input == null) return null;
            
            return input.replace("&", "&amp;")
                       .replace("<", "&lt;")
                       .replace(">", "&gt;")
                       .replace("\"", "&quot;")
                       .replace("'", "&#x27;")
                       .replace(" ", "&#x20;");
        }
        
        public static String encodeJavaScript(String input) {
            if (input == null) return null;
            
            return input.replace("\\", "\\\\")
                       .replace("\"", "\\\"")
                       .replace("'", "\\'")
                       .replace("\n", "\\n")
                       .replace("\r", "\\r")
                       .replace("\t", "\\t")
                       .replace("<", "\\u003C")
                       .replace(">", "\\u003E");
        }
        
        public static String encodeUrl(String input) {
            if (input == null) return null;
            
            try {
                return java.net.URLEncoder.encode(input, "UTF-8")
                    .replace("+", "%20")
                    .replace("*", "%2A")
                    .replace("%7E", "~");
            } catch (java.io.UnsupportedEncodingException e) {
                throw new RuntimeException(e);
            }
        }
        
        public static String encodeCss(String input) {
            if (input == null) return null;
            
            StringBuilder encoded = new StringBuilder();
            for (char c : input.toCharArray()) {
                if (c >= 0x20 && c <= 0x7E && c != '"' && c != '\'' && c != '\\') {
                    encoded.append(c);
                } else {
                    encoded.append("\\").append(String.format("%04x", (int) c));
                }
            }
            return encoded.toString();
        }
    }
    
    // Secure configuration management
    public static class SecureConfiguration {
        private final Properties properties = new Properties();
        private final Set<String> sensitiveKeys = new HashSet<>();
        
        public SecureConfiguration() {
            // Mark sensitive keys
            sensitiveKeys.add("password");
            sensitiveKeys.add("secret");
            sensitiveKeys.add("key");
            sensitiveKeys.add("token");
            sensitiveKeys.add("api.key");
            sensitiveKeys.add("db.password");
        }
        
        public void load(java.io.InputStream inputStream) throws java.io.IOException {
            properties.load(inputStream);
        }
        
        public String getProperty(String key) {
            return properties.getProperty(key);
        }
        
        public String getSecureProperty(String key) {
            String value = properties.getProperty(key);
            if (value != null && isSensitiveKey(key)) {
                // Log access to sensitive properties
                System.out.println("Accessing sensitive property: " + key);
            }
            return value;
        }
        
        public void setProperty(String key, String value) {
            properties.setProperty(key, value);
        }
        
        public void setSecureProperty(String key, String value) {
            if (isSensitiveKey(key)) {
                // Encrypt sensitive values before storing
                String encryptedValue = encrypt(value);
                properties.setProperty(key, encryptedValue);
            } else {
                properties.setProperty(key, value);
            }
        }
        
        public void printSecurely(java.io.PrintStream out) {
            for (String key : properties.stringPropertyNames()) {
                String value = properties.getProperty(key);
                if (isSensitiveKey(key)) {
                    out.println(key + "=" + maskSensitiveValue(value));
                } else {
                    out.println(key + "=" + value);
                }
            }
        }
        
        private boolean isSensitiveKey(String key) {
            return sensitiveKeys.stream()
                .anyMatch(sensitive -> key.toLowerCase().contains(sensitive.toLowerCase()));
        }
        
        private String encrypt(String value) {
            // Implement proper encryption for sensitive values
            // This is a placeholder - use proper encryption in production
            return value; // Don't actually store plain text!
        }
        
        private String maskSensitiveValue(String value) {
            if (value == null) return null;
            return "*".repeat(Math.max(0, value.length()));
        }
    }
    
    // Security logging framework
    public static class SecurityLogger {
        private final java.util.logging.Logger logger = 
            java.util.logging.Logger.getLogger(SecurityLogger.class.getName());
        
        public void logAuthentication(String username, String action, String result, String ip) {
            String logMessage = String.format(
                "AUTHENTICATION_EVENT - User: %s, Action: %s, Result: %s, IP: %s", 
                username, action, result, ip
            );
            logger.info(logMessage);
        }
        
        public void logAuthorization(String username, String resource, String action, boolean allowed) {
            String logMessage = String.format(
                "AUTHORIZATION_EVENT - User: %s, Resource: %s, Action: %s, Allowed: %s", 
                username, resource, action, allowed
            );
            logger.info(logMessage);
        }
        
        public void logSecurityEvent(String eventType, String details, String severity) {
            String logMessage = String.format(
                "SECURITY_EVENT - Type: %s, Details: %s, Severity: %s", 
                eventType, details, severity
            );
            logger.info(logMessage);
        }
        
        public void logInputValidationFailure(String input, String rule, String ip) {
            String logMessage = String.format(
                "INPUT_VALIDATION_FAILURE - Input: %s, Rule: %s, IP: %s", 
                input, rule, ip
            );
            logger.warning(logMessage);
        }
        
        public void logDataAccess(String username, String resource, String action, String result) {
            String logMessage = String.format(
                "DATA_ACCESS_EVENT - User: %s, Resource: %s, Action: %s, Result: %s", 
                username, resource, action, result
            );
            logger.info(logMessage);
        }
    }
}
```

## Conclusion

Advanced Security Best Practices and Implementation provide:
- Comprehensive authentication and authorization frameworks
- Robust cryptographic implementations for data protection
- Secure coding practices for input validation and sanitization
- Security monitoring and logging capabilities
- Protection against common security vulnerabilities

Mastering these concepts is essential for:
- Building secure, production-ready applications
- Implementing enterprise security requirements
- Protecting against common attack vectors
- Ensuring compliance with security standards
- Creating robust security architectures