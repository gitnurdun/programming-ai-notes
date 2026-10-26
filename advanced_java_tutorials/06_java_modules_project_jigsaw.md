# Advanced Java Modules and Project Jigsaw

## Overview
Java Platform Module System (JPMS), introduced in Java 9, provides a powerful mechanism for creating modular applications with strong encapsulation and explicit dependencies. Understanding modules is crucial for modern Java development, especially in large-scale enterprise applications.

## Module System Fundamentals

### Module Declaration and Structure

```java
// module-info.java - Core module declaration
module com.example.core {
    // Export packages for other modules to use
    exports com.example.core.api;
    exports com.example.core.model;
    
    // Open packages for reflection access
    opens com.example.core.internal to com.example.serialization;
    
    // Require other modules
    requires java.base;           // Implicitly required
    requires java.logging;        // Standard module
    requires transitive java.xml; // Transitive dependency
    
    // Require modules with specific access levels
    requires static com.example.optional; // Optional dependency
    requires com.example.other;           // Required dependency
    
    // Services
    uses com.example.core.spi.ServiceProvider;
    provides com.example.core.spi.ServiceProvider 
        with com.example.core.impl.DefaultServiceProvider;
}

// Example package structure
// com.example.core.api
// com.example.core.model
// com.example.core.internal
// com.example.core.spi
// com.example.core.impl
```

### Module Types

1. **System Modules**: Part of the JDK (java.base, java.logging, etc.)
2. **Application Modules**: User-defined modules with module-info.java
3. **Automatic Modules**: JARs on module path without module-info.java
4. **Unnamed Module**: Legacy classpath JARs

## Module Declaration Examples

### Core Application Module

```java
// module-info.java for a core application module
module com.example.application {
    // Exports - make packages accessible to other modules
    exports com.example.application.model;
    exports com.example.application.service;
    exports com.example.application.api;
    
    // Opens - allow reflection access (use sparingly)
    opens com.example.application.config to com.fasterxml.jackson.databind;
    opens com.example.application.internal to com.example.testing;
    
    // Requires - declare dependencies
    requires java.base;                    // Always required implicitly
    requires java.logging;                 // JDK module
    requires transitive java.sql;          // Transitive dependency
    requires static org.apache.commons.lang3; // Optional dependency
    requires com.fasterxml.jackson.databind;   // Third-party module
    
    // Services
    uses com.example.application.spi.DatabaseProvider;  // Service consumer
    provides com.example.application.spi.NotificationService 
        with com.example.application.impl.EmailNotificationService; // Service provider
}
```

### Library Module

```java
// module-info.java for a library module
module com.example.library {
    // Public API exports
    exports com.example.library.api;
    exports com.example.library.model;
    exports com.example.library.exception;
    
    // Limited exports (to specific modules only)
    exports com.example.library.spi to 
        com.example.application,
        com.example.testing;
    
    // Opens for reflection
    opens com.example.library.internal to 
        com.fasterxml.jackson.databind,
        com.google.gson;
    
    // Dependencies
    requires java.base;
    requires transitive java.logging;
    requires static java.management; // Optional
    
    // Service definitions
    uses com.example.library.spi.Plugin;
    provides com.example.library.spi.Serializer 
        with com.example.library.impl.JsonSerializer,
             com.example.library.impl.XmlSerializer;
}
```

## Module Path vs Classpath

### Module Path Operations

```java
// Running with modules
// java --module-path lib --module com.example.application/com.example.application.Main

// Listing available modules
// java --module-path lib --list-modules

// Describing a module
// java --module-path lib --describe-module com.example.library

// Example of checking module dependencies at runtime
public class ModuleInspector {
    public static void inspectModules() {
        ModuleLayer layer = ModuleLayer.boot();
        
        System.out.println("Modules in boot layer:");
        layer.modules().forEach(module -> {
            System.out.println("  " + module.getName());
            
            // Module descriptor information
            ModuleDescriptor descriptor = module.getDescriptor().orElse(null);
            if (descriptor != null) {
                System.out.println("    Exports: " + descriptor.exports());
                System.out.println("    Requires: " + descriptor.requires());
                System.out.println("    Opens: " + descriptor.opens());
            }
        });
    }
    
    public static void inspectCurrentModule() {
        Module currentModule = ModuleInspector.class.getModule();
        System.out.println("Current module: " + currentModule.getName());
        System.out.println("Is open: " + currentModule.isOpen());
        System.out.println("Is named: " + currentModule.isNamed());
        
        // Package information
        currentModule.getPackages().forEach(pkg -> {
            System.out.println("  Package: " + pkg);
            System.out.println("    Can read: " + currentModule.canRead(currentModule));
        });
    }
}
```

### Migration from Classpath to Module Path

```java
// Example of handling automatic modules
module com.example.migration {
    // Automatic modules are available but not strongly encapsulated
    requires com.google.gson;        // Becomes automatic module
    requires org.apache.commons.io;  // Becomes automatic module
    
    // Exports for your own code
    exports com.example.migration.api;
    exports com.example.migration.service;
    
    // Opens for reflection with automatic modules
    opens com.example.migration.config to com.google.gson;
}
```

## Advanced Module Features

### Services and ServiceLoader

```java
// Service provider interface
package com.example.services.spi;

public interface MessageService {
    String formatMessage(String message);
    String getServiceName();
}

// Service implementations
package com.example.services.impl;

public class JsonMessageService implements MessageService {
    @Override
    public String formatMessage(String message) {
        return "{\"message\": \"" + message + "\"}";
    }
    
    @Override
    public String getServiceName() {
        return "JSON";
    }
}

public class XmlMessageService implements MessageService {
    @Override
    public String formatMessage(String message) {
        return "<message>" + message + "</message>";
    }
    
    @Override
    public String getServiceName() {
        return "XML";
    }
}

// Module declaration with service
module com.example.services {
    exports com.example.services.spi;
    exports com.example.services.impl;
    
    provides com.example.services.spi.MessageService 
        with com.example.services.impl.JsonMessageService,
             com.example.services.impl.XmlMessageService;
}

// Service consumer
module com.example.consumer {
    requires com.example.services;
    uses com.example.services.spi.MessageService;
}

// Service consumer implementation
package com.example.consumer;

import com.example.services.spi.MessageService;
import java.util.ServiceLoader;
import java.util.List;
import java.util.stream.Collectors;

public class MessageProcessor {
    private final List<MessageService> services;
    
    public MessageProcessor() {
        // Load services using ServiceLoader
        ServiceLoader<MessageService> loader = ServiceLoader.load(MessageService.class);
        this.services = loader.stream()
            .map(ServiceLoader.Provider::get)
            .collect(Collectors.toList());
    }
    
    public void processMessage(String message) {
        services.forEach(service -> {
            System.out.println(service.getServiceName() + ": " + 
                             service.formatMessage(message));
        });
    }
    
    public MessageService getServiceByName(String name) {
        return services.stream()
            .filter(service -> service.getServiceName().equals(name))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Service not found: " + name));
    }
}
```

### Layered Module System

```java
import java.lang.module.ModuleDescriptor;
import java.lang.module.ModuleFinder;
import java.lang.module.ModuleReference;
import java.nio.file.Path;
import java.util.Set;

public class ModuleLayerExample {
    
    public static void createCustomLayer() throws Exception {
        // Create module finder for custom modules
        ModuleFinder finder = ModuleFinder.of(
            Path.of("lib/custom-module.jar"),
            Path.of("modules/")
        );
        
        // Define module names to include
        Set<String> moduleNames = Set.of("com.example.custom");
        
        // Get parent layer (usually boot layer)
        ModuleLayer parent = ModuleLayer.boot();
        
        // Create configuration
        ModuleLayer.Controller controller = ModuleLayer.defineModulesWithOneLoader(
            parent.configuration().resolve(finder, ModuleFinder.of(), moduleNames),
            List.of(parent),
            ClassLoader.getSystemClassLoader()
        );
        
        // Get the new layer
        ModuleLayer layer = controller.layer();
        
        // Load and use classes from the new layer
        Optional<Module> customModule = layer.findModule("com.example.custom");
        if (customModule.isPresent()) {
            Class<?> customClass = layer.findLoader("com.example.custom")
                .loadClass("com.example.custom.CustomService");
            
            Object instance = customClass.getDeclaredConstructor().newInstance();
            System.out.println("Loaded from custom layer: " + instance);
        }
    }
    
    public static void inspectModuleLayers() {
        // Current layer
        ModuleLayer currentLayer = ModuleLayerExample.class.getModule().getLayer();
        if (currentLayer != null) {
            System.out.println("Current layer: " + currentLayer);
            currentLayer.modules().forEach(m -> 
                System.out.println("  Module: " + m.getName()));
        }
        
        // Parent layers
        ModuleLayer parent = ModuleLayer.boot();
        System.out.println("Boot layer modules:");
        parent.modules().forEach(m -> 
            System.out.println("  Module: " + m.getName()));
    }
}
```

## Module Testing

### Testing Modular Applications

```java
// Test module declaration
open module com.example.application.test {
    // Open all packages for reflection (testing)
    requires com.example.application;
    requires org.junit.jupiter.api;
    requires org.mockito;
    
    // Allow access to internal packages for testing
    exports com.example.application.test.util;
}
```

```java
// Example test with module access
package com.example.application.test;

import com.example.application.service.UserService;
import com.example.application.model.User;
import org.junit.jupiter.api.Test;
import org.mockito.MockedStatic;
import static org.mockito.Mockito.*;

class UserServiceTest {
    
    @Test
    void testCreateUser() {
        // Test with proper module access
        UserService service = new UserService();
        User user = service.createUser("John", "john@example.com");
        
        assert user != null;
        assert "John".equals(user.getName());
    }
    
    // Testing with reflection in modular environment
    @Test
    void testInternalAccess() throws Exception {
        UserService service = new UserService();
        
        // Access internal field using reflection (requires opens directive)
        java.lang.reflect.Field internalField = 
            UserService.class.getDeclaredField("internalCounter");
        internalField.setAccessible(true);
        
        // This works because of the opens directive in module-info.java
        int value = (int) internalField.get(service);
        System.out.println("Internal counter: " + value);
    }
}
```

## Migration Strategies

### Step-by-Step Migration

1. **Make JARs Modular-Ready**:
```java
// Add module-info.java to existing projects gradually
// Start with automatic modules
```

2. **Strong Encapsulation**:
```java
// Before migration - everything is accessible
public class LegacyService {
    public void internalMethod() { } // Public but internal
}

// After migration - proper encapsulation
module com.example.migrated {
    exports com.example.migrated.api;  // Public API only
    // internal classes remain encapsulated
}
```

3. **Dependency Management**:
```java
// Update build tools for module support
// Maven example:
/*
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <release>17</release>
    </configuration>
</plugin>
*/
```

## Performance and Security Benefits

### Performance Improvements

1. **Faster Startup**: JVM only loads required modules
2. **Reduced Memory Footprint**: Smaller runtime image
3. **Optimized Compilation**: Better JIT compilation with module boundaries

### Security Enhancements

1. **Strong Encapsulation**: Prevents unauthorized access to internal APIs
2. **Explicit Dependencies**: Clear security boundaries
3. **Reduced Attack Surface**: Smaller, more controlled runtime

## Best Practices

1. **Design for Modularity from Start**:
   - Keep modules cohesive and loosely coupled
   - Minimize exports to essential APIs only
   - Use services for loose coupling

2. **Migration Approach**:
   - Start with automatic modules
   - Gradually add module-info.java
   - Refactor for strong encapsulation

3. **Testing**:
   - Create open test modules for reflection access
   - Test module boundaries explicitly
   - Verify service loading works correctly

4. **Build Tools**:
   - Configure Maven/Gradle for module support
   - Use jlink for custom runtime images
   - Consider multi-release JARs for compatibility

## Conclusion

Java Modules provide:
- Strong encapsulation and explicit dependencies
- Improved performance and security
- Better maintainability for large applications
- Foundation for modern Java development

Understanding modules is essential for:
- Enterprise application architecture
- Library development
- Performance optimization
- Security hardening
- Migration of legacy applications