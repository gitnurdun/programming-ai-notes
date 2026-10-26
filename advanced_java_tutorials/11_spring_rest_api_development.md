# Advanced Spring REST and API Development

## Overview
Spring Web and REST support provide powerful tools for building modern RESTful APIs. Understanding advanced features like content negotiation, validation, exception handling, and API documentation is essential for creating robust, maintainable web services.

## REST Controller Patterns

### Advanced Controller Design

```java
@RestController
@RequestMapping("/api/users")
@Slf4j
@Validated
public class UserController {
    
    private final UserService userService;
    private final UserMapper userMapper;
    
    public UserController(UserService userService, UserMapper userMapper) {
        this.userService = userService;
        this.userMapper = userMapper;
    }
    
    // GET with multiple response representations
    @GetMapping(produces = {
        MediaType.APPLICATION_JSON_VALUE,
        MediaType.APPLICATION_XML_VALUE,
        "application/hal+json"
    })
    public ResponseEntity<PagedModel<UserResource>> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id") String sortBy,
            @RequestParam(defaultValue = "asc") String direction,
            @RequestParam(required = false) String search,
            HttpServletRequest request) {
        
        Pageable pageable = PageRequest.of(page, size, 
            Sort.Direction.fromString(direction), sortBy);
        
        Page<User> userPage;
        if (search != null && !search.isEmpty()) {
            userPage = userService.searchUsers(search, pageable);
        } else {
            userPage = userService.getAllUsers(pageable);
        }
        
        // HATEOAS links
        PagedModel<UserResource> pagedModel = PagedModel.of(
            userPage.getContent().stream()
                .map(userMapper::toResource)
                .collect(Collectors.toList()),
            new PagedModel.PageMetadata(
                userPage.getSize(),
                userPage.getNumber(),
                userPage.getTotalElements(),
                userPage.getTotalPages()
            )
        );
        
        // Add links based on content type
        String acceptHeader = request.getHeader("Accept");
        if (acceptHeader != null && acceptHeader.contains("application/hal+json")) {
            addHalLinks(pagedModel, userPage, request);
        }
        
        return ResponseEntity.ok(pagedModel);
    }
    
    // GET with conditional requests
    @GetMapping("/{id}")
    public ResponseEntity<UserResource> getUser(
            @PathVariable Long id,
            @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch,
            @RequestHeader(value = "If-Modified-Since", required = false) 
                @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime ifModifiedSince) {
        
        User user = userService.getUserById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        String etag = generateETag(user);
        LocalDateTime lastModified = user.getLastModified();
        
        // Check ETag
        if (ifNoneMatch != null && ifNoneMatch.equals(etag)) {
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED)
                .eTag(etag)
                .lastModified(lastModified.atZone(ZoneId.systemDefault()).toInstant())
                .build();
        }
        
        // Check last modified
        if (ifModifiedSince != null && 
            lastModified.truncatedTo(ChronoUnit.SECONDS)
                       .isBefore(ifModifiedSince.truncatedTo(ChronoUnit.SECONDS))) {
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED)
                .eTag(etag)
                .lastModified(lastModified.atZone(ZoneId.systemDefault()).toInstant())
                .build();
        }
        
        UserResource resource = userMapper.toResource(user);
        return ResponseEntity.ok()
            .eTag(etag)
            .lastModified(lastModified.atZone(ZoneId.systemDefault()).toInstant())
            .body(resource);
    }
    
    // POST with validation and different response codes
    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<UserResource> createUser(
            @Valid @RequestBody CreateUserRequest request,
            BindingResult bindingResult,
            UriComponentsBuilder uriBuilder) {
        
        // Manual validation
        if (bindingResult.hasErrors()) {
            throw new ValidationException(bindingResult.getFieldErrors());
        }
        
        // Business validation
        validateUserRequest(request);
        
        User user = userMapper.toEntity(request);
        User savedUser = userService.createUser(user);
        
        UserResource resource = userMapper.toResource(savedUser);
        URI location = uriBuilder.path("/api/users/{id}")
            .buildAndExpand(savedUser.getId())
            .toUri();
        
        return ResponseEntity.created(location).body(resource);
    }
    
    // PUT with optimistic locking
    @PutMapping(value = "/{id}", consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<UserResource> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request,
            @RequestHeader(value = "If-Match", required = false) String ifMatch) {
        
        User existingUser = userService.getUserById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        // Optimistic locking check
        if (ifMatch != null) {
            String currentETag = generateETag(existingUser);
            if (!ifMatch.equals(currentETag)) {
                throw new OptimisticLockingException("User has been modified by another request");
            }
        }
        
        User updatedUser = userMapper.updateEntity(existingUser, request);
        User savedUser = userService.updateUser(updatedUser);
        
        UserResource resource = userMapper.toResource(savedUser);
        return ResponseEntity.ok()
            .eTag(generateETag(savedUser))
            .body(resource);
    }
    
    // PATCH with partial updates
    @PatchMapping(value = "/{id}", consumes = "application/merge-patch+json")
    public ResponseEntity<UserResource> patchUser(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        
        User user = userService.getUserById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        // Apply partial updates
        userMapper.applyPatch(user, updates);
        User updatedUser = userService.updateUser(user);
        
        UserResource resource = userMapper.toResource(updatedUser);
        return ResponseEntity.ok()
            .eTag(generateETag(updatedUser))
            .body(resource);
    }
    
    // DELETE with different response codes
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        if (!userService.userExists(id)) {
            return ResponseEntity.notFound().build();
        }
        
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
    
    // Custom media type support
    @GetMapping(value = "/{id}/avatar", produces = MediaType.IMAGE_PNG_VALUE)
    public ResponseEntity<Resource> getUserAvatar(@PathVariable Long id) {
        User user = userService.getUserById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        Resource avatar = userService.getUserAvatar(user);
        if (avatar == null) {
            return ResponseEntity.notFound().build();
        }
        
        return ResponseEntity.ok()
            .contentType(MediaType.IMAGE_PNG)
            .contentLength(avatar.contentLength())
            .body(avatar);
    }
    
    private String generateETag(User user) {
        return "\"" + user.getVersion() + "-" + user.getLastModified().toInstant(ZoneOffset.UTC).toEpochMilli() + "\"";
    }
    
    private void addHalLinks(PagedModel<UserResource> pagedModel, Page<User> userPage, 
                           HttpServletRequest request) {
        String baseUrl = ServletUriComponentsBuilder.fromRequestUri(request)
            .replacePath(request.getContextPath() + "/api/users")
            .build()
            .toUriString();
        
        pagedModel.add(Link.of(baseUrl, "self"));
        
        if (userPage.hasPrevious()) {
            pagedModel.add(Link.of(baseUrl + "?page=" + (userPage.getNumber() - 1), "prev"));
        }
        
        if (userPage.hasNext()) {
            pagedModel.add(Link.of(baseUrl + "?page=" + (userPage.getNumber() + 1), "next"));
        }
        
        pagedModel.add(Link.of(baseUrl + "?page=0", "first"));
        pagedModel.add(Link.of(baseUrl + "?page=" + (userPage.getTotalPages() - 1), "last"));
    }
    
    private void validateUserRequest(CreateUserRequest request) {
        if (userService.existsByUsername(request.getUsername())) {
            throw new BusinessValidationException("Username already exists");
        }
        
        if (userService.existsByEmail(request.getEmail())) {
            throw new BusinessValidationException("Email already exists");
        }
    }
}
```

### Request and Response Models

```java
// DTOs with validation
@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonIgnoreProperties(ignoreUnknown = true)
public class CreateUserRequest {
    
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username must contain only letters, numbers, and underscores")
    private String username;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).*$", 
             message = "Password must contain at least one lowercase letter, one uppercase letter, and one digit")
    private String password;
    
    @NotNull(message = "Date of birth is required")
    @Past(message = "Date of birth must be in the past")
    private LocalDate dateOfBirth;
    
    @Valid
    private Address address;
    
    // Getters and setters
}

@JsonInclude(JsonInclude.Include.NON_NULL)
public class UserResource {
    
    private Long id;
    private String username;
    private String email;
    private LocalDate dateOfBirth;
    private Address address;
    private LocalDateTime createdDate;
    private LocalDateTime lastModified;
    
    // HATEOAS links
    private final List<Link> links = new ArrayList<>();
    
    // Getters and setters
    
    public UserResource add(Link link) {
        this.links.add(link);
        return this;
    }
    
    @JsonGetter("_links")
    public Map<String, Link> getLinks() {
        Map<String, Link> linkMap = new HashMap<>();
        links.forEach(link -> linkMap.put(link.getRel(), link));
        return linkMap;
    }
}

// Custom serializers and deserializers
@JsonComponent
public class LocalDateSerializer extends JsonSerializer<LocalDate> {
    
    @Override
    public void serialize(LocalDate value, JsonGenerator gen, SerializerProvider serializers) 
            throws IOException {
        gen.writeString(value.format(DateTimeFormatter.ISO_LOCAL_DATE));
    }
}

@JsonComponent
public class LocalDateDeserializer extends JsonDeserializer<LocalDate> {
    
    @Override
    public LocalDate deserialize(JsonParser p, DeserializationContext ctxt) 
            throws IOException, JsonProcessingException {
        return LocalDate.parse(p.getValueAsString(), DateTimeFormatter.ISO_LOCAL_DATE);
    }
}
```

## Exception Handling and Error Responses

### Global Exception Handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        log.warn("User not found: {}", ex.getMessage());
        
        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error("User Not Found")
            .message(ex.getMessage())
            .path(getCurrentPath())
            .build();
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(ValidationException ex) {
        log.warn("Validation failed: {}", ex.getMessage());
        
        ValidationErrorResponse error = ValidationErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Input validation failed")
            .path(getCurrentPath())
            .errors(ex.getFieldErrors().stream()
                .map(fieldError -> ValidationError.builder()
                    .field(fieldError.getField())
                    .rejectedValue(fieldError.getRejectedValue())
                    .message(fieldError.getDefaultMessage())
                    .build())
                .collect(Collectors.toList()))
            .build();
        
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(OptimisticLockingException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleOptimisticLocking(OptimisticLockingException ex) {
        log.warn("Optimistic locking conflict: {}", ex.getMessage());
        
        return ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.CONFLICT.value())
            .error("Conflict")
            .message("Resource has been modified by another user")
            .path(getCurrentPath())
            .build();
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        log.warn("Access denied: {}", ex.getMessage());
        
        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.FORBIDDEN.value())
            .error("Forbidden")
            .message("Access denied to this resource")
            .path(getCurrentPath())
            .build();
        
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex) {
        
        log.warn("Method argument validation failed: {}", ex.getMessage());
        
        ValidationErrorResponse error = ValidationErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Input validation failed")
            .path(getCurrentPath())
            .errors(ex.getBindingResult().getFieldErrors().stream()
                .map(fieldError -> ValidationError.builder()
                    .field(fieldError.getField())
                    .rejectedValue(fieldError.getRejectedValue())
                    .message(fieldError.getDefaultMessage())
                    .build())
                .collect(Collectors.toList()))
            .build();
        
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ErrorResponse> handleHttpMessageNotReadable(
            HttpMessageNotReadableException ex) {
        
        log.warn("Malformed request: {}", ex.getMessage());
        
        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Bad Request")
            .message("Malformed request body")
            .path(getCurrentPath())
            .build();
        
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error occurred", ex);
        
        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Internal Server Error")
            .message("An unexpected error occurred")
            .path(getCurrentPath())
            .build();
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
    
    private String getCurrentPath() {
        ServletWebRequest requestAttributes = (ServletWebRequest) RequestContextHolder.getRequestAttributes();
        if (requestAttributes != null) {
            HttpServletRequest request = requestAttributes.getRequest();
            return request.getRequestURI();
        }
        return "unknown";
    }
}

// Error response models
@Builder
@Data
@AllArgsConstructor
@NoArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private String exception;
}

@Builder
@Data
@AllArgsConstructor
@NoArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ValidationErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private List<ValidationError> errors;
}

@Builder
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ValidationError {
    private String field;
    private Object rejectedValue;
    private String message;
}
```

## Content Negotiation and Media Types

### Advanced Content Negotiation

```java
@RestController
@RequestMapping("/api/documents")
public class DocumentController {
    
    private final DocumentService documentService;
    
    public DocumentController(DocumentService documentService) {
        this.documentService = documentService;
    }
    
    // Multiple representations based on Accept header
    @GetMapping(value = "/{id}", produces = {
        MediaType.APPLICATION_JSON_VALUE,
        MediaType.APPLICATION_XML_VALUE,
        "application/hal+json",
        "text/csv"
    })
    public ResponseEntity<?> getDocument(@PathVariable Long id, HttpServletRequest request) {
        
        Document document = documentService.findById(id)
            .orElseThrow(() -> new DocumentNotFoundException(id));
        
        String acceptHeader = request.getHeader("Accept");
        
        if (acceptHeader != null) {
            if (acceptHeader.contains("application/xml")) {
                return ResponseEntity.ok().contentType(MediaType.APPLICATION_XML)
                    .body(toXmlDocument(document));
            } else if (acceptHeader.contains("text/csv")) {
                return ResponseEntity.ok().contentType(MediaType.parseMediaType("text/csv"))
                    .body(toCsvDocument(document));
            } else if (acceptHeader.contains("application/hal+json")) {
                return ResponseEntity.ok().contentType(MediaType.parseMediaType("application/hal+json"))
                    .body(toHalDocument(document));
            }
        }
        
        // Default to JSON
        return ResponseEntity.ok().contentType(MediaType.APPLICATION_JSON)
            .body(toJsonDocument(document));
    }
    
    // Custom media type with versioning
    @GetMapping(value = "/{id}", 
                produces = {"application/vnd.company.document.v1+json"})
    public ResponseEntity<DocumentV1> getDocumentV1(@PathVariable Long id) {
        Document document = documentService.findById(id)
            .orElseThrow(() -> new DocumentNotFoundException(id));
        return ResponseEntity.ok(toDocumentV1(document));
    }
    
    @GetMapping(value = "/{id}", 
                produces = {"application/vnd.company.document.v2+json"})
    public ResponseEntity<DocumentV2> getDocumentV2(@PathVariable Long id) {
        Document document = documentService.findById(id)
            .orElseThrow(() -> new DocumentNotFoundException(id));
        return ResponseEntity.ok(toDocumentV2(document));
    }
    
    // File upload with content type validation
    @PostMapping(value = "/{id}/content", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<DocumentContentInfo> uploadDocumentContent(
            @PathVariable Long id,
            @RequestParam("file") MultipartFile file) {
        
        // Validate file type
        String contentType = file.getContentType();
        if (!isValidContentType(contentType)) {
            throw new BusinessValidationException("Unsupported file type: " + contentType);
        }
        
        // Validate file size
        if (file.getSize() > getMaxFileSize()) {
            throw new BusinessValidationException("File too large");
        }
        
        DocumentContentInfo contentInfo = documentService.saveDocumentContent(id, file);
        return ResponseEntity.ok(contentInfo);
    }
    
    // File download with streaming
    @GetMapping(value = "/{id}/content", produces = MediaType.APPLICATION_OCTET_STREAM_VALUE)
    public ResponseEntity<Resource> downloadDocumentContent(
            @PathVariable Long id,
            HttpServletResponse response) {
        
        DocumentContent content = documentService.getDocumentContent(id);
        
        Resource resource = new ByteArrayResource(content.getData());
        
        return ResponseEntity.ok()
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .contentLength(content.getSize())
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                   "attachment; filename=\"" + content.getFileName() + "\"")
            .eTag("\"" + content.getChecksum() + "\"")
            .lastModified(content.getLastModified().atZone(ZoneId.systemDefault()).toInstant())
            .body(resource);
    }
    
    private boolean isValidContentType(String contentType) {
        return contentType != null && (
            contentType.startsWith("application/pdf") ||
            contentType.startsWith("application/msword") ||
            contentType.startsWith("application/vnd.openxmlformats-officedocument.wordprocessingml.document") ||
            contentType.startsWith("text/")
        );
    }
    
    private long getMaxFileSize() {
        return 10 * 1024 * 1024; // 10MB
    }
    
    // Conversion methods
    private XmlDocument toXmlDocument(Document document) {
        // XML conversion logic
        return new XmlDocument();
    }
    
    private String toCsvDocument(Document document) {
        // CSV conversion logic
        return "";
    }
    
    private HalDocument toHalDocument(Document document) {
        // HAL conversion logic with links
        return new HalDocument();
    }
    
    private JsonDocument toJsonDocument(Document document) {
        // JSON conversion logic
        return new JsonDocument();
    }
    
    private DocumentV1 toDocumentV1(Document document) {
        // V1 conversion logic
        return new DocumentV1();
    }
    
    private DocumentV2 toDocumentV2(Document document) {
        // V2 conversion logic with additional fields
        return new DocumentV2();
    }
}
```

## API Versioning

### Versioning Strategies

```java
// 1. URI Versioning
@RestController
@RequestMapping("/api/v1/users")
public class UserV1Controller {
    
    @GetMapping("/{id}")
    public ResponseEntity<UserV1> getUser(@PathVariable Long id) {
        // V1 implementation
        return ResponseEntity.ok(new UserV1());
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserV2Controller {
    
    @GetMapping("/{id}")
    public ResponseEntity<UserV2> getUser(@PathVariable Long id) {
        // V2 implementation with additional fields
        return ResponseEntity.ok(new UserV2());
    }
}

// 2. Media Type Versioning
@RestController
@RequestMapping("/api/users")
public class UserMediaTypeVersionController {
    
    @GetMapping(value = "/{id}", 
                produces = {"application/vnd.company.user.v1+json"})
    public ResponseEntity<UserV1> getUserV1(@PathVariable Long id) {
        return ResponseEntity.ok(new UserV1());
    }
    
    @GetMapping(value = "/{id}", 
                produces = {"application/vnd.company.user.v2+json"})
    public ResponseEntity<UserV2> getUserV2(@PathVariable Long id) {
        return ResponseEntity.ok(new UserV2());
    }
}

// 3. Header Versioning
@RestController
@RequestMapping("/api/users")
public class UserHeaderVersionController {
    
    @GetMapping("/{id}")
    public ResponseEntity<?> getUser(@PathVariable Long id, 
                                   @RequestHeader(value = "API-Version", defaultValue = "1") 
                                   String apiVersion) {
        
        switch (apiVersion) {
            case "1":
                return ResponseEntity.ok(new UserV1());
            case "2":
                return ResponseEntity.ok(new UserV2());
            default:
                throw new UnsupportedApiVersionException(apiVersion);
        }
    }
}

// 4. Parameter Versioning
@RestController
@RequestMapping("/api/users")
public class UserParameterVersionController {
    
    @GetMapping("/{id}")
    public ResponseEntity<?> getUser(@PathVariable Long id, 
                                   @RequestParam(value = "version", defaultValue = "1") 
                                   String version) {
        
        switch (version) {
            case "1":
                return ResponseEntity.ok(new UserV1());
            case "2":
                return ResponseEntity.ok(new UserV2());
            default:
                throw new UnsupportedApiVersionException(version);
        }
    }
}

// Version-aware service
@Service
public class VersionedUserService {
    
    public Object getUser(Long id, String version) {
        switch (version) {
            case "1":
                return getUserV1(id);
            case "2":
                return getUserV2(id);
            default:
                throw new UnsupportedApiVersionException(version);
        }
    }
    
    private UserV1 getUserV1(Long id) {
        // V1 implementation
        return new UserV1();
    }
    
    private UserV2 getUserV2(Long id) {
        // V2 implementation
        return new UserV2();
    }
}
```

## Testing REST APIs

### Comprehensive API Testing

```java
@WebMvcTest(UserController.class)
@Import({UserMapperImpl.class, GlobalExceptionHandler.class})
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void testGetUser_Success() throws Exception {
        // Given
        User user = new User(1L, "john_doe", "john@example.com");
        when(userService.getUserById(1L)).thenReturn(Optional.of(user));
        
        // When & Then
        mockMvc.perform(get("/api/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1L))
            .andExpect(jsonPath("$.username").value("john_doe"))
            .andExpect(jsonPath("$.email").value("john@example.com"))
            .andExpect(header().string("ETag", notNullValue()));
    }
    
    @Test
    void testGetUser_NotFound() throws Exception {
        // Given
        when(userService.getUserById(999L)).thenReturn(Optional.empty());
        
        // When & Then
        mockMvc.perform(get("/api/users/999")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("User Not Found"));
    }
    
    @Test
    void testCreateUser_Success() throws Exception {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername("jane_doe");
        request.setEmail("jane@example.com");
        request.setPassword("Password123");
        request.setDateOfBirth(LocalDate.of(1990, 1, 1));
        
        User user = new User(2L, "jane_doe", "jane@example.com");
        when(userService.createUser(any(User.class))).thenReturn(user);
        
        // When & Then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(2L))
            .andExpect(jsonPath("$.username").value("jane_doe"))
            .andExpect(header().string("Location", containsString("/api/users/2")));
    }
    
    @Test
    void testCreateUser_ValidationFailure() throws Exception {
        // Given
        CreateUserRequest request = new CreateUserRequest();
        request.setUsername(""); // Invalid
        request.setEmail("invalid-email"); // Invalid
        
        // When & Then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors", hasSize(2)))
            .andExpect(jsonPath("$.errors[0].field").value("username"))
            .andExpect(jsonPath("$.errors[1].field").value("email"));
    }
    
    @Test
    void testUpdateUser_Conflict() throws Exception {
        // Given
        UpdateUserRequest request = new UpdateUserRequest();
        request.setEmail("new@example.com");
        
        User user = new User(1L, "john_doe", "john@example.com");
        when(userService.getUserById(1L)).thenReturn(Optional.of(user));
        
        // When & Then
        mockMvc.perform(put("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON)
                .header("If-Match", "\"wrong-etag\"")
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.error").value("Conflict"));
    }
    
    @Test
    void testConditionalGet_NotModified() throws Exception {
        // Given
        User user = new User(1L, "john_doe", "john@example.com");
        user.setLastModified(LocalDateTime.of(2023, 1, 1, 12, 0));
        user.setVersion(1L);
        
        when(userService.getUserById(1L)).thenReturn(Optional.of(user));
        
        // When & Then
        mockMvc.perform(get("/api/users/1")
                .header("If-None-Match", "\"1-1672574400000\"")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isNotModified())
            .andExpect(header().string("ETag", "\"1-1672574400000\""));
    }
}

// Integration testing with Testcontainers
@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserApiIntegrationTest {
    
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
    void testUserLifecycle() {
        // 1. Create user
        CreateUserRequest createRequest = new CreateUserRequest();
        createRequest.setUsername("testuser");
        createRequest.setEmail("test@example.com");
        createRequest.setPassword("Password123");
        createRequest.setDateOfBirth(LocalDate.of(1990, 1, 1));
        
        ResponseEntity<UserResource> createResponse = restTemplate.postForEntity(
            "/api/users", createRequest, UserResource.class);
        
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(createResponse.getBody()).isNotNull();
        assertThat(createResponse.getBody().getUsername()).isEqualTo("testuser");
        
        Long userId = createResponse.getBody().getId();
        URI location = createResponse.getHeaders().getLocation();
        
        // 2. Get user
        ResponseEntity<UserResource> getResponse = restTemplate.getForEntity(
            location, UserResource.class);
        
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().getId()).isEqualTo(userId);
        assertThat(getResponse.getHeaders()).containsKey("ETag");
        
        String etag = getResponse.getHeaders().getETag();
        
        // 3. Update user
        UpdateUserRequest updateRequest = new UpdateUserRequest();
        updateRequest.setEmail("updated@example.com");
        
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setIfMatch(etag);
        
        HttpEntity<UpdateUserRequest> updateEntity = new HttpEntity<>(updateRequest, headers);
        
        ResponseEntity<UserResource> updateResponse = restTemplate.exchange(
            location, HttpMethod.PUT, updateEntity, UserResource.class);
        
        assertThat(updateResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(updateResponse.getBody().getEmail()).isEqualTo("updated@example.com");
        
        // 4. Delete user
        restTemplate.delete(location);
        
        // 5. Verify deletion
        ResponseEntity<UserResource> deletedResponse = restTemplate.getForEntity(
            location, UserResource.class);
        
        assertThat(deletedResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}
```

## API Documentation with SpringDoc OpenAPI

### OpenAPI Configuration

```java
@Configuration
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("User Management API")
                .version("1.0.0")
                .description("API for managing users in the system")
                .termsOfService("https://example.com/terms")
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0.html"))
                .contact(new Contact()
                    .name("API Support")
                    .url("https://example.com/support")
                    .email("support@example.com")))
            .servers(List.of(
                new Server().url("https://api.example.com").description("Production server"),
                new Server().url("https://staging-api.example.com").description("Staging server"),
                new Server().url("http://localhost:8080").description("Development server")
            ))
            .components(new Components()
                .addSecuritySchemes("bearerAuth", new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
                    .description("JWT Authorization header using the Bearer scheme"))
                .addSchemas("User", createUserSchema())
                .addSchemas("CreateUserRequest", createCreateUserRequestSchema()))
            .security(List.of(new SecurityRequirement().addList("bearerAuth")));
    }
    
    private Schema createUserSchema() {
        return new Schema<User>()
            .addProperties("id", new IntegerSchema().format("int64"))
            .addProperties("username", new StringSchema())
            .addProperties("email", new StringSchema().format("email"))
            .addProperties("createdDate", new DateTimeSchema())
            .example(new User(1L, "john_doe", "john@example.com"));
    }
    
    private Schema createCreateUserRequestSchema() {
        return new Schema<CreateUserRequest>()
            .addProperties("username", new StringSchema()
                .minLength(3)
                .maxLength(50)
                .pattern("^[a-zA-Z0-9_]+$"))
            .addProperties("email", new StringSchema().format("email"))
            .addProperties("password", new StringSchema().minLength(8))
            .required(List.of("username", "email", "password"));
    }
}

// Controller documentation
@RestController
@RequestMapping("/api/users")
@Tag(name = "User Management", description = "Operations for managing users")
public class UserController {
    
    @Operation(
        summary = "Get all users",
        description = "Returns a paginated list of users with optional search functionality"
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Successfully retrieved users",
            content = @Content(mediaType = "application/json",
                schema = @Schema(implementation = PagedModel.class))),
        @ApiResponse(responseCode = "400", description = "Invalid request parameters",
            content = @Content(mediaType = "application/json",
                schema = @Schema(implementation = ValidationErrorResponse.class))),
        @ApiResponse(responseCode = "401", description = "Unauthorized",
            content = @Content),
        @ApiResponse(responseCode = "403", description = "Forbidden",
            content = @Content)
    })
    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<PagedModel<UserResource>> getUsers(
            @Parameter(description = "Page number (0-based)", example = "0") 
            @RequestParam(defaultValue = "0") int page,
            @Parameter(description = "Page size", example = "20") 
            @RequestParam(defaultValue = "20") int size,
            @Parameter(description = "Search term") 
            @RequestParam(required = false) String search) {
        // Implementation
        return ResponseEntity.ok(new PagedModel<>());
    }
    
    @Operation(summary = "Get user by ID")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "User found"),
        @ApiResponse(responseCode = "304", description = "Not modified"),
        @ApiResponse(responseCode = "404", description = "User not found")
    })
    @GetMapping("/{id}")
    public ResponseEntity<UserResource> getUser(
            @Parameter(description = "User ID", required = true) @PathVariable Long id,
            @Parameter(description = "ETag for conditional request") 
            @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {
        // Implementation
        return ResponseEntity.ok(new UserResource());
    }
    
    @Operation(summary = "Create new user")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "User created"),
        @ApiResponse(responseCode = "400", description = "Validation error"),
        @ApiResponse(responseCode = "409", description = "User already exists")
    })
    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<UserResource> createUser(
            @Parameter(description = "User data", required = true) 
            @Valid @RequestBody CreateUserRequest request) {
        // Implementation
        return ResponseEntity.created(null).body(new UserResource());
    }
}
```

## Conclusion

Advanced Spring REST and API Development provides:
- Comprehensive RESTful API design patterns
- Robust error handling and validation
- Content negotiation and media type support
- API versioning strategies
- Comprehensive testing approaches
- API documentation with OpenAPI

Mastering these features is essential for:
- Senior Java developer roles
- Enterprise API development
- Microservices architecture
- API design and documentation
- Performance optimization and security