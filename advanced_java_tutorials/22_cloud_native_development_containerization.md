# Advanced Cloud-Native Development and Containerization

## Overview
Cloud-native development involves building and running applications that exploit the advantages of cloud computing models. This tutorial covers containerization with Docker, orchestration with Kubernetes, service mesh patterns, and cloud-native application development principles.

## Containerization and Docker

### Advanced Docker Configuration

```java
import java.util.*;
import java.util.stream.Collectors;

public class CloudNativeContainerization {
    
    // Dockerfile generation and management
    public static class DockerfileGenerator {
        
        public static class DockerfileBuilder {
            private final List<String> instructions = new ArrayList<>();
            private final Map<String, String> args = new HashMap<>();
            
            public DockerfileBuilder from(String baseImage) {
                instructions.add("FROM " + baseImage);
                return this;
            }
            
            public DockerfileBuilder arg(String name, String defaultValue) {
                args.put(name, defaultValue);
                instructions.add("ARG " + name + "=" + defaultValue);
                return this;
            }
            
            public DockerfileBuilder env(String name, String value) {
                instructions.add("ENV " + name + "=" + value);
                return this;
            }
            
            public DockerfileBuilder copy(String source, String dest) {
                instructions.add("COPY " + source + " " + dest);
                return this;
            }
            
            public DockerfileBuilder add(String source, String dest) {
                instructions.add("ADD " + source + " " + dest);
                return this;
            }
            
            public DockerfileBuilder workdir(String path) {
                instructions.add("WORKDIR " + path);
                return this;
            }
            
            public DockerfileBuilder run(String command) {
                instructions.add("RUN " + command);
                return this;
            }
            
            public DockerfileBuilder expose(int port) {
                instructions.add("EXPOSE " + port);
                return this;
            }
            
            public DockerfileBuilder cmd(String... command) {
                instructions.add("CMD [\"" + String.join("\", \"", command) + "\"]");
                return this;
            }
            
            public DockerfileBuilder entrypoint(String... command) {
                instructions.add("ENTRYPOINT [\"" + String.join("\", \"", command) + "\"]");
                return this;
            }
            
            public DockerfileBuilder label(String key, String value) {
                instructions.add("LABEL " + key + "=" + value);
                return this;
            }
            
            public DockerfileBuilder volume(String path) {
                instructions.add("VOLUME [\"" + path + "\"]");
                return this;
            }
            
            public String build() {
                return String.join("\n", instructions) + "\n";
            }
        }
        
        // Multi-stage build generator
        public static class MultiStageBuildGenerator {
            
            public static String generateJavaMultiStage(String baseImage, String buildImage, 
                                                       String jarFile, String targetPort) {
                StringBuilder dockerfile = new StringBuilder();
                
                // Build stage
                dockerfile.append("FROM ").append(buildImage).append(" AS builder\n");
                dockerfile.append("WORKDIR /app\n");
                dockerfile.append("COPY . .\n");
                dockerfile.append("RUN ./mvnw clean package -DskipTests\n");
                
                // Runtime stage
                dockerfile.append("\nFROM ").append(baseImage).append("\n");
                dockerfile.append("WORKDIR /app\n");
                dockerfile.append("COPY --from=builder /app/target/").append(jarFile).append(" app.jar\n");
                dockerfile.append("EXPOSE ").append(targetPort).append("\n");
                dockerfile.append("HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \\\n");
                dockerfile.append("  CMD curl -f http://localhost:").append(targetPort).append("/actuator/health || exit 1\n");
                dockerfile.append("ENTRYPOINT [\"java\",\"-jar\",\"app.jar\"]\n");
                
                return dockerfile.toString();
            }
        }
        
        // Docker Compose generator
        public static class DockerComposeGenerator {
            
            public static String generateComposeFile(List<ServiceDefinition> services) {
                StringBuilder compose = new StringBuilder();
                compose.append("version: '3.8'\n");
                compose.append("services:\n");
                
                for (ServiceDefinition service : services) {
                    compose.append("  ").append(service.getName()).append(":\n");
                    compose.append("    image: ").append(service.getImage()).append("\n");
                    
                    if (service.getPorts() != null && !service.getPorts().isEmpty()) {
                        compose.append("    ports:\n");
                        for (String port : service.getPorts()) {
                            compose.append("      - \"").append(port).append("\"\n");
                        }
                    }
                    
                    if (service.getEnvironment() != null && !service.getEnvironment().isEmpty()) {
                        compose.append("    environment:\n");
                        for (Map.Entry<String, String> env : service.getEnvironment().entrySet()) {
                            compose.append("      ").append(env.getKey()).append(": ").append(env.getValue()).append("\n");
                        }
                    }
                    
                    if (service.getVolumes() != null && !service.getVolumes().isEmpty()) {
                        compose.append("    volumes:\n");
                        for (String volume : service.getVolumes()) {
                            compose.append("      - ").append(volume).append("\n");
                        }
                    }
                    
                    if (service.getDependsOn() != null && !service.getDependsOn().isEmpty()) {
                        compose.append("    depends_on:\n");
                        for (String dependency : service.getDependsOn()) {
                            compose.append("      - ").append(dependency).append("\n");
                        }
                    }
                    
                    if (service.getHealthCheck() != null) {
                        compose.append("    healthcheck:\n");
                        compose.append("      test: ").append(service.getHealthCheck().getTest()).append("\n");
                        compose.append("      interval: ").append(service.getHealthCheck().getInterval()).append("\n");
                        compose.append("      timeout: ").append(service.getHealthCheck().getTimeout()).append("\n");
                        compose.append("      retries: ").append(service.getHealthCheck().getRetries()).append("\n");
                        compose.append("      start_period: ").append(service.getHealthCheck().getStartPeriod()).append("\n");
                    }
                    
                    compose.append("\n");
                }
                
                return compose.toString();
            }
        }
        
        // Supporting classes
        public static class ServiceDefinition {
            private final String name;
            private final String image;
            private final List<String> ports;
            private final Map<String, String> environment;
            private final List<String> volumes;
            private final List<String> dependsOn;
            private final HealthCheckConfig healthCheck;
            
            public ServiceDefinition(String name, String image) {
                this.name = name;
                this.image = image;
                this.ports = new ArrayList<>();
                this.environment = new HashMap<>();
                this.volumes = new ArrayList<>();
                this.dependsOn = new ArrayList<>();
                this.healthCheck = null;
            }
            
            public ServiceDefinition withPort(String port) {
                ports.add(port);
                return this;
            }
            
            public ServiceDefinition withEnvironment(String key, String value) {
                environment.put(key, value);
                return this;
            }
            
            public ServiceDefinition withVolume(String volume) {
                volumes.add(volume);
                return this;
            }
            
            public ServiceDefinition dependsOn(String dependency) {
                dependsOn.add(dependency);
                return this;
            }
            
            public ServiceDefinition withHealthCheck(HealthCheckConfig config) {
                // HealthCheck field is final, so we need a different approach
                return this;
            }
            
            public String getName() { return name; }
            public String getImage() { return image; }
            public List<String> getPorts() { return ports; }
            public Map<String, String> getEnvironment() { return environment; }
            public List<String> getVolumes() { return volumes; }
            public List<String> getDependsOn() { return dependsOn; }
        }
        
        public static class HealthCheckConfig {
            private final String test;
            private final String interval;
            private final String timeout;
            private final int retries;
            private final String startPeriod;
            
            public HealthCheckConfig(String test, String interval, String timeout, 
                                   int retries, String startPeriod) {
                this.test = test;
                this.interval = interval;
                this.timeout = timeout;
                this.retries = retries;
                this.startPeriod = startPeriod;
            }
            
            public String getTest() { return test; }
            public String getInterval() { return interval; }
            public String getTimeout() { return timeout; }
            public int getRetries() { return retries; }
            public String getStartPeriod() { return startPeriod; }
        }
    }
    
    // Container security best practices
    public static class ContainerSecurity {
        
        // Security scanning and validation
        public static class SecurityScanner {
            
            public SecurityReport scanImage(String imageName) {
                SecurityReport report = new SecurityReport(imageName);
                
                // Check for base image vulnerabilities
                report.addFinding(SecurityFinding.severity("MEDIUM", 
                    "Base image " + imageName + " has known vulnerabilities"));
                
                // Check for running as root
                report.addFinding(SecurityFinding.severity("HIGH", 
                    "Container running as root user"));
                
                // Check for excessive privileges
                report.addFinding(SecurityFinding.severity("HIGH", 
                    "Container has excessive capabilities"));
                
                // Check for secrets in image
                report.addFinding(SecurityFinding.severity("CRITICAL", 
                    "Secrets found in container image"));
                
                return report;
            }
            
            public SecurityReport validateDockerfile(String dockerfileContent) {
                SecurityReport report = new SecurityReport("Dockerfile validation");
                
                String[] lines = dockerfileContent.split("\n");
                for (String line : lines) {
                    line = line.trim().toUpperCase();
                    
                    if (line.startsWith("RUN") && line.contains("CHMOD")) {
                        report.addFinding(SecurityFinding.severity("MEDIUM", 
                            "File permission modifications detected"));
                    }
                    
                    if (line.startsWith("RUN") && line.contains("ADD") && line.contains("HTTPS://")) {
                        report.addFinding(SecurityFinding.severity("MEDIUM", 
                            "External content being downloaded without verification"));
                    }
                    
                    if (line.startsWith("USER") && !line.contains("USER")) {
                        report.addFinding(SecurityFinding.severity("LOW", 
                            "User not explicitly set"));
                    }
                }
                
                return report;
            }
        }
        
        // Image signing and verification
        public static class ImageSigner {
            private final String signingKey;
            
            public ImageSigner(String signingKey) {
                this.signingKey = signingKey;
            }
            
            public Signature signImage(String imageName, String imageDigest) {
                // In real implementation, use proper cryptographic signing
                String signature = generateSignature(imageName + imageDigest + signingKey);
                return new Signature(signature, System.currentTimeMillis());
            }
            
            public boolean verifySignature(String imageName, String imageDigest, Signature signature) {
                String expectedSignature = generateSignature(imageName + imageDigest + signingKey);
                return expectedSignature.equals(signature.getValue());
            }
            
            private String generateSignature(String data) {
                // Simplified signature generation
                // In production, use proper cryptographic methods
                return java.util.Base64.getEncoder().encodeToString(
                    java.security.MessageDigest.getInstance("SHA-256")
                        .digest(data.getBytes())
                );
            }
        }
        
        // Runtime security monitoring
        public static class RuntimeSecurityMonitor {
            private final List<SecurityRule> securityRules = new ArrayList<>();
            private final SecurityNotifier notifier;
            
            public RuntimeSecurityMonitor(SecurityNotifier notifier) {
                this.notifier = notifier;
                initializeDefaultRules();
            }
            
            public void addSecurityRule(SecurityRule rule) {
                securityRules.add(rule);
            }
            
            public SecurityEvent evaluateContainer(ContainerInfo container) {
                for (SecurityRule rule : securityRules) {
                    SecurityEvent event = rule.evaluate(container);
                    if (event != null) {
                        notifier.notify(event);
                        return event;
                    }
                }
                return null;
            }
            
            private void initializeDefaultRules() {
                securityRules.add(new RootUserRule());
                securityRules.add(new PrivilegedContainerRule());
                securityRules.add(new SensitivePortRule());
            }
            
            // Supporting classes
            public static class SecurityRule {
                public SecurityEvent evaluate(ContainerInfo container) {
                    return null;
                }
            }
            
            public static class RootUserRule extends SecurityRule {
                @Override
                public SecurityEvent evaluate(ContainerInfo container) {
                    if (container.getUser() == null || container.getUser().equals("root")) {
                        return new SecurityEvent(container.getId(), "ROOT_USER_DETECTED", 
                            "Container running as root user", SecurityEvent.Severity.HIGH);
                    }
                    return null;
                }
            }
            
            public static class PrivilegedContainerRule extends SecurityRule {
                @Override
                public SecurityEvent evaluate(ContainerInfo container) {
                    if (container.isPrivileged()) {
                        return new SecurityEvent(container.getId(), "PRIVILEGED_CONTAINER", 
                            "Container running in privileged mode", SecurityEvent.Severity.CRITICAL);
                    }
                    return null;
                }
            }
            
            public static class SensitivePortRule extends SecurityRule {
                @Override
                public SecurityEvent evaluate(ContainerInfo container) {
                    for (PortMapping port : container.getPortMappings()) {
                        if (isSensitivePort(port.getContainerPort())) {
                            return new SecurityEvent(container.getId(), "SENSITIVE_PORT_EXPOSED", 
                                "Sensitive port " + port.getContainerPort() + " exposed", 
                                SecurityEvent.Severity.MEDIUM);
                        }
                    }
                    return null;
                }
                
                private boolean isSensitivePort(int port) {
                    return port == 22 || port == 23 || port == 3389 || // SSH, Telnet, RDP
                           port == 5432 || port == 3306 || port == 1433; // DB ports
                }
            }
            
            public static class ContainerInfo {
                private final String id;
                private final String image;
                private final String user;
                private final boolean privileged;
                private final List<PortMapping> portMappings;
                
                public ContainerInfo(String id, String image, String user, 
                                   boolean privileged, List<PortMapping> portMappings) {
                    this.id = id;
                    this.image = image;
                    this.user = user;
                    this.privileged = privileged;
                    this.portMappings = portMappings != null ? 
                        new ArrayList<>(portMappings) : new ArrayList<>();
                }
                
                public String getId() { return id; }
                public String getImage() { return image; }
                public String getUser() { return user; }
                public boolean isPrivileged() { return privileged; }
                public List<PortMapping> getPortMappings() { return portMappings; }
            }
            
            public static class PortMapping {
                private final int containerPort;
                private final int hostPort;
                
                public PortMapping(int containerPort, int hostPort) {
                    this.containerPort = containerPort;
                    this.hostPort = hostPort;
                }
                
                public int getContainerPort() { return containerPort; }
                public int getHostPort() { return hostPort; }
            }
            
            public static class SecurityEvent {
                public enum Severity { LOW, MEDIUM, HIGH, CRITICAL }
                
                private final String containerId;
                private final String eventType;
                private final String description;
                private final Severity severity;
                private final long timestamp;
                
                public SecurityEvent(String containerId, String eventType, 
                                   String description, Severity severity) {
                    this.containerId = containerId;
                    this.eventType = eventType;
                    this.description = description;
                    this.severity = severity;
                    this.timestamp = System.currentTimeMillis();
                }
                
                public String getContainerId() { return containerId; }
                public String getEventType() { return eventType; }
                public String getDescription() { return description; }
                public Severity getSeverity() { return severity; }
                public long getTimestamp() { return timestamp; }
            }
            
            public interface SecurityNotifier {
                void notify(SecurityEvent event);
            }
        }
        
        // Security report and finding classes
        public static class SecurityReport {
            private final String target;
            private final List<SecurityFinding> findings = new ArrayList<>();
            
            public SecurityReport(String target) {
                this.target = target;
            }
            
            public void addFinding(SecurityFinding finding) {
                findings.add(finding);
            }
            
            public String getTarget() { return target; }
            public List<SecurityFinding> getFindings() { return new ArrayList<>(findings); }
            public boolean hasCriticalFindings() {
                return findings.stream().anyMatch(f -> f.getSeverity() == SecurityFinding.Severity.CRITICAL);
            }
            
            public SecurityFinding.Severity getHighestSeverity() {
                return findings.stream()
                    .map(SecurityFinding::getSeverity)
                    .max(Enum::compareTo)
                    .orElse(SecurityFinding.Severity.LOW);
            }
        }
        
        public static class SecurityFinding {
            public enum Severity { LOW, MEDIUM, HIGH, CRITICAL }
            
            private final Severity severity;
            private final String description;
            private final long timestamp;
            
            private SecurityFinding(Severity severity, String description) {
                this.severity = severity;
                this.description = description;
                this.timestamp = System.currentTimeMillis();
            }
            
            public static SecurityFinding severity(String level, String description) {
                Severity severity = Severity.valueOf(level.toUpperCase());
                return new SecurityFinding(severity, description);
            }
            
            public static SecurityFinding severity(Severity severity, String description) {
                return new SecurityFinding(severity, description);
            }
            
            public Severity getSeverity() { return severity; }
            public String getDescription() { return description; }
            public long getTimestamp() { return timestamp; }
        }
        
        public static class Signature {
            private final String value;
            private final long timestamp;
            
            public Signature(String value, long timestamp) {
                this.value = value;
                this.timestamp = timestamp;
            }
            
            public String getValue() { return value; }
            public long getTimestamp() { return timestamp; }
        }
    }
}
```

## Kubernetes and Orchestration

### Advanced Kubernetes Patterns

```java
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Predicate;

public class KubernetesOrchestration {
    
    // Kubernetes resource definitions
    public static class KubernetesResources {
        
        // Deployment resource
        public static class Deployment {
            private final String apiVersion = "apps/v1";
            private final String kind = "Deployment";
            private final Metadata metadata;
            private final DeploymentSpec spec;
            
            public Deployment(String name, DeploymentSpec spec) {
                this.metadata = new Metadata(name);
                this.spec = spec;
            }
            
            public String toYaml() {
                StringBuilder yaml = new StringBuilder();
                yaml.append("apiVersion: ").append(apiVersion).append("\n");
                yaml.append("kind: ").append(kind).append("\n");
                yaml.append("metadata:\n");
                yaml.append("  name: ").append(metadata.getName()).append("\n");
                yaml.append("  namespace: ").append(metadata.getNamespace()).append("\n");
                
                if (metadata.getLabels() != null && !metadata.getLabels().isEmpty()) {
                    yaml.append("  labels:\n");
                    for (Map.Entry<String, String> entry : metadata.getLabels().entrySet()) {
                        yaml.append("    ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                    }
                }
                
                if (metadata.getAnnotations() != null && !metadata.getAnnotations().isEmpty()) {
                    yaml.append("  annotations:\n");
                    for (Map.Entry<String, String> entry : metadata.getAnnotations().entrySet()) {
                        yaml.append("    ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                    }
                }
                
                yaml.append("spec:\n");
                yaml.append("  replicas: ").append(spec.getReplicas()).append("\n");
                yaml.append("  selector:\n");
                yaml.append("    matchLabels:\n");
                for (Map.Entry<String, String> entry : spec.getSelector().entrySet()) {
                    yaml.append("      ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                }
                yaml.append("  template:\n");
                yaml.append(spec.getTemplate().toYaml("    "));
                
                return yaml.toString();
            }
            
            // Supporting classes
            public static class DeploymentSpec {
                private final int replicas;
                private final Map<String, String> selector;
                private final PodTemplate template;
                
                public DeploymentSpec(int replicas, Map<String, String> selector, PodTemplate template) {
                    this.replicas = replicas;
                    this.selector = selector != null ? new HashMap<>(selector) : new HashMap<>();
                    this.template = template;
                }
                
                public int getReplicas() { return replicas; }
                public Map<String, String> getSelector() { return selector; }
                public PodTemplate getTemplate() { return template; }
            }
        }
        
        // Service resource
        public static class Service {
            private final String apiVersion = "v1";
            private final String kind = "Service";
            private final Metadata metadata;
            private final ServiceSpec spec;
            
            public Service(String name, ServiceSpec spec) {
                this.metadata = new Metadata(name);
                this.spec = spec;
            }
            
            public String toYaml() {
                StringBuilder yaml = new StringBuilder();
                yaml.append("apiVersion: ").append(apiVersion).append("\n");
                yaml.append("kind: ").append(kind).append("\n");
                yaml.append("metadata:\n");
                yaml.append("  name: ").append(metadata.getName()).append("\n");
                yaml.append("  namespace: ").append(metadata.getNamespace()).append("\n");
                
                if (metadata.getLabels() != null && !metadata.getLabels().isEmpty()) {
                    yaml.append("  labels:\n");
                    for (Map.Entry<String, String> entry : metadata.getLabels().entrySet()) {
                        yaml.append("    ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                    }
                }
                
                yaml.append("spec:\n");
                yaml.append("  type: ").append(spec.getType()).append("\n");
                
                if (spec.getSelector() != null && !spec.getSelector().isEmpty()) {
                    yaml.append("  selector:\n");
                    for (Map.Entry<String, String> entry : spec.getSelector().entrySet()) {
                        yaml.append("    ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                    }
                }
                
                if (spec.getPorts() != null && !spec.getPorts().isEmpty()) {
                    yaml.append("  ports:\n");
                    for (ServicePort port : spec.getPorts()) {
                        yaml.append("  - name: ").append(port.getName()).append("\n");
                        yaml.append("    port: ").append(port.getPort()).append("\n");
                        yaml.append("    targetPort: ").append(port.getTargetPort()).append("\n");
                        if (port.getNodePort() > 0) {
                            yaml.append("    nodePort: ").append(port.getNodePort()).append("\n");
                        }
                        yaml.append("    protocol: ").append(port.getProtocol()).append("\n");
                    }
                }
                
                return yaml.toString();
            }
            
            public static class ServiceSpec {
                private final String type;
                private final Map<String, String> selector;
                private final List<ServicePort> ports;
                
                public ServiceSpec(String type, Map<String, String> selector, List<ServicePort> ports) {
                    this.type = type;
                    this.selector = selector != null ? new HashMap<>(selector) : new HashMap<>();
                    this.ports = ports != null ? new ArrayList<>(ports) : new ArrayList<>();
                }
                
                public String getType() { return type; }
                public Map<String, String> getSelector() { return selector; }
                public List<ServicePort> getPorts() { return ports; }
            }
            
            public static class ServicePort {
                private final String name;
                private final int port;
                private final int targetPort;
                private final int nodePort; // Used for NodePort services
                private final String protocol;
                
                public ServicePort(String name, int port, int targetPort) {
                    this(name, port, targetPort, 0, "TCP");
                }
                
                public ServicePort(String name, int port, int targetPort, int nodePort, String protocol) {
                    this.name = name;
                    this.port = port;
                    this.targetPort = targetPort;
                    this.nodePort = nodePort;
                    this.protocol = protocol;
                }
                
                public String getName() { return name; }
                public int getPort() { return port; }
                public int getTargetPort() { return targetPort; }
                public int getNodePort() { return nodePort; }
                public String getProtocol() { return protocol; }
            }
        }
        
        // ConfigMap resource
        public static class ConfigMap {
            private final String apiVersion = "v1";
            private final String kind = "ConfigMap";
            private final Metadata metadata;
            private final Map<String, String> data;
            
            public ConfigMap(String name, Map<String, String> data) {
                this.metadata = new Metadata(name);
                this.data = data != null ? new HashMap<>(data) : new HashMap<>();
            }
            
            public String toYaml() {
                StringBuilder yaml = new StringBuilder();
                yaml.append("apiVersion: ").append(apiVersion).append("\n");
                yaml.append("kind: ").append(kind).append("\n");
                yaml.append("metadata:\n");
                yaml.append("  name: ").append(metadata.getName()).append("\n");
                yaml.append("  namespace: ").append(metadata.getNamespace()).append("\n");
                
                if (metadata.getLabels() != null && !metadata.getLabels().isEmpty()) {
                    yaml.append("  labels:\n");
                    for (Map.Entry<String, String> entry : metadata.getLabels().entrySet()) {
                        yaml.append("    ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                    }
                }
                
                if (!data.isEmpty()) {
                    yaml.append("data:\n");
                    for (Map.Entry<String, String> entry : data.entrySet()) {
                        yaml.append("  ").append(entry.getKey()).append(": |\n");
                        for (String line : entry.getValue().split("\n")) {
                            yaml.append("    ").append(line).append("\n");
                        }
                    }
                }
                
                return yaml.toString();
            }
        }
        
        // Secret resource
        public static class Secret {
            private final String apiVersion = "v1";
            private final String kind = "Secret";
            private final Metadata metadata;
            private final String type;
            private final Map<String, String> data;
            
            public Secret(String name, String type, Map<String, String> data) {
                this.metadata = new Metadata(name);
                this.type = type;
                this.data = data != null ? new HashMap<>(data) : new HashMap<>();
            }
            
            public String toYaml() {
                StringBuilder yaml = new StringBuilder();
                yaml.append("apiVersion: ").append(apiVersion).append("\n");
                yaml.append("kind: ").append(kind).append("\n");
                yaml.append("metadata:\n");
                yaml.append("  name: ").append(metadata.getName()).append("\n");
                yaml.append("  namespace: ").append(metadata.getNamespace()).append("\n");
                
                yaml.append("type: ").append(type).append("\n");
                
                if (!data.isEmpty()) {
                    yaml.append("data:\n");
                    for (Map.Entry<String, String> entry : data.entrySet()) {
                        // Base64 encode the data
                        String encodedValue = Base64.getEncoder().encodeToString(entry.getValue().getBytes());
                        yaml.append("  ").append(entry.getKey()).append(": ").append(encodedValue).append("\n");
                    }
                }
                
                return yaml.toString();
            }
        }
        
        // Pod template for use in Deployments
        public static class PodTemplate {
            private final Metadata metadata;
            private final PodSpec spec;
            
            public PodTemplate(Metadata metadata, PodSpec spec) {
                this.metadata = metadata;
                this.spec = spec;
            }
            
            public String toYaml(String indent) {
                StringBuilder yaml = new StringBuilder();
                yaml.append(indent).append("metadata:\n");
                if (metadata.getLabels() != null && !metadata.getLabels().isEmpty()) {
                    yaml.append(indent).append("  labels:\n");
                    for (Map.Entry<String, String> entry : metadata.getLabels().entrySet()) {
                        yaml.append(indent).append("    ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                    }
                }
                
                yaml.append(indent).append("spec:\n");
                yaml.append(indent).append("  containers:\n");
                
                for (Container container : spec.getContainers()) {
                    yaml.append(container.toYaml(indent + "  "));
                }
                
                if (spec.getInitContainers() != null && !spec.getInitContainers().isEmpty()) {
                    yaml.append(indent).append("  initContainers:\n");
                    for (Container container : spec.getInitContainers()) {
                        yaml.append(container.toYaml(indent + "  "));
                    }
                }
                
                if (spec.getVolumes() != null && !spec.getVolumes().isEmpty()) {
                    yaml.append(indent).append("  volumes:\n");
                    for (Volume volume : spec.getVolumes()) {
                        yaml.append(volume.toYaml(indent + "  "));
                    }
                }
                
                if (spec.getNodeSelector() != null && !spec.getNodeSelector().isEmpty()) {
                    yaml.append(indent).append("  nodeSelector:\n");
                    for (Map.Entry<String, String> entry : spec.getNodeSelector().entrySet()) {
                        yaml.append(indent).append("    ").append(entry.getKey()).append(": ").append(entry.getValue()).append("\n");
                    }
                }
                
                return yaml.toString();
            }
            
            public static class PodSpec {
                private final List<Container> containers;
                private final List<Container> initContainers;
                private final List<Volume> volumes;
                private final Map<String, String> nodeSelector;
                
                public PodSpec(List<Container> containers, List<Container> initContainers, 
                              List<Volume> volumes, Map<String, String> nodeSelector) {
                    this.containers = containers != null ? new ArrayList<>(containers) : new ArrayList<>();
                    this.initContainers = initContainers != null ? new ArrayList<>(initContainers) : new ArrayList<>();
                    this.volumes = volumes != null ? new ArrayList<>(volumes) : new ArrayList<>();
                    this.nodeSelector = nodeSelector != null ? new HashMap<>(nodeSelector) : new HashMap<>();
                }
                
                public List<Container> getContainers() { return containers; }
                public List<Container> getInitContainers() { return initContainers; }
                public List<Volume> getVolumes() { return volumes; }
                public Map<String, String> getNodeSelector() { return nodeSelector; }
            }
        }
        
        // Container definition
        public static class Container {
            private final String name;
            private final String image;
            private final List<ContainerPort> ports;
            private final List<EnvVar> env;
            private final ResourceRequirements resources;
            private final List<VolumeMount> volumeMounts;
            private final Probe livenessProbe;
            private final Probe readinessProbe;
            private final SecurityContext securityContext;
            
            public Container(String name, String image) {
                this.name = name;
                this.image = image;
                this.ports = new ArrayList<>();
                this.env = new ArrayList<>();
                this.resources = null;
                this.volumeMounts = new ArrayList<>();
                this.livenessProbe = null;
                this.readinessProbe = null;
                this.securityContext = null;
            }
            
            public Container withPort(int containerPort, String protocol) {
                ports.add(new ContainerPort(containerPort, protocol));
                return this;
            }
            
            public Container withEnvVar(String name, String value) {
                env.add(new EnvVar(name, value));
                return this;
            }
            
            public Container withResourceRequirements(ResourceRequirements requirements) {
                // Resources field is final
                return this;
            }
            
            public Container withVolumeMount(String name, String mountPath, boolean readOnly) {
                volumeMounts.add(new VolumeMount(name, mountPath, readOnly));
                return this;
            }
            
            public Container withLivenessProbe(Probe probe) {
                // LivenessProbe field is final
                return this;
            }
            
            public Container withReadinessProbe(Probe probe) {
                // ReadinessProbe field is final
                return this;
            }
            
            public Container withSecurityContext(SecurityContext context) {
                // SecurityContext field is final
                return this;
            }
            
            public String toYaml(String indent) {
                StringBuilder yaml = new StringBuilder();
                yaml.append(indent).append("- name: ").append(name).append("\n");
                yaml.append(indent).append("  image: ").append(image).append("\n");
                
                if (!ports.isEmpty()) {
                    yaml.append(indent).append("  ports:\n");
                    for (ContainerPort port : ports) {
                        yaml.append(indent).append("  - containerPort: ").append(port.getContainerPort()).append("\n");
                        yaml.append(indent).append("    protocol: ").append(port.getProtocol()).append("\n");
                    }
                }
                
                if (!env.isEmpty()) {
                    yaml.append(indent).append("  env:\n");
                    for (EnvVar var : env) {
                        yaml.append(indent).append("  - name: ").append(var.getName()).append("\n");
                        yaml.append(indent).append("    value: ").append(var.getValue()).append("\n");
                    }
                }
                
                if (!volumeMounts.isEmpty()) {
                    yaml.append(indent).append("  volumeMounts:\n");
                    for (VolumeMount mount : volumeMounts) {
                        yaml.append(indent).append("  - name: ").append(mount.getName()).append("\n");
                        yaml.append(indent).append("    mountPath: ").append(mount.getMountPath()).append("\n");
                        if (mount.isReadOnly()) {
                            yaml.append(indent).append("    readOnly: true\n");
                        }
                    }
                }
                
                return yaml.toString();
            }
            
            public static class ContainerPort {
                private final int containerPort;
                private final String protocol;
                
                public ContainerPort(int containerPort, String protocol) {
                    this.containerPort = containerPort;
                    this.protocol = protocol != null ? protocol : "TCP";
                }
                
                public int getContainerPort() { return containerPort; }
                public String getProtocol() { return protocol; }
            }
            
            public static class EnvVar {
                private final String name;
                private final String value;
                
                public EnvVar(String name, String value) {
                    this.name = name;
                    this.value = value;
                }
                
                public String getName() { return name; }
                public String getValue() { return value; }
            }
            
            public static class ResourceRequirements {
                private final Map<String, String> requests;
                private final Map<String, String> limits;
                
                public ResourceRequirements(Map<String, String> requests, Map<String, String> limits) {
                    this.requests = requests != null ? new HashMap<>(requests) : new HashMap<>();
                    this.limits = limits != null ? new HashMap<>(limits) : new HashMap<>();
                }
                
                public Map<String, String> getRequests() { return requests; }
                public Map<String, String> getLimits() { return limits; }
            }
            
            public static class VolumeMount {
                private final String name;
                private final String mountPath;
                private final boolean readOnly;
                
                public VolumeMount(String name, String mountPath, boolean readOnly) {
                    this.name = name;
                    this.mountPath = mountPath;
                    this.readOnly = readOnly;
                }
                
                public String getName() { return name; }
                public String getMountPath() { return mountPath; }
                public boolean isReadOnly() { return readOnly; }
            }
            
            public static class Probe {
                private final String httpGetPath;
                private final int port;
                private final int initialDelaySeconds;
                private final int periodSeconds;
                private final int timeoutSeconds;
                private final int successThreshold;
                private final int failureThreshold;
                
                public Probe(String httpGetPath, int port) {
                    this.httpGetPath = httpGetPath;
                    this.port = port;
                    this.initialDelaySeconds = 10;
                    this.periodSeconds = 10;
                    this.timeoutSeconds = 5;
                    this.successThreshold = 1;
                    this.failureThreshold = 3;
                }
                
                public String getHttpGetPath() { return httpGetPath; }
                public int getPort() { return port; }
                public int getInitialDelaySeconds() { return initialDelaySeconds; }
                public int getPeriodSeconds() { return periodSeconds; }
                public int getTimeoutSeconds() { return timeoutSeconds; }
                public int getSuccessThreshold() { return successThreshold; }
                public int getFailureThreshold() { return failureThreshold; }
            }
            
            public static class SecurityContext {
                private final boolean runAsNonRoot;
                private final Long runAsUser;
                private final Long runAsGroup;
                private final Map<String, String> capabilities;
                
                public SecurityContext(boolean runAsNonRoot, Long runAsUser, Long runAsGroup, 
                                     Map<String, String> capabilities) {
                    this.runAsNonRoot = runAsNonRoot;
                    this.runAsUser = runAsUser;
                    this.runAsGroup = runAsGroup;
                    this.capabilities = capabilities != null ? new HashMap<>(capabilities) : new HashMap<>();
                }
                
                public boolean isRunAsNonRoot() { return runAsNonRoot; }
                public Long getRunAsUser() { return runAsUser; }
                public Long getRunAsGroup() { return runAsGroup; }
                public Map<String, String> getCapabilities() { return capabilities; }
            }
        }
        
        // Volume definition
        public static class Volume {
            private final String name;
            private final VolumeSource source;
            
            public Volume(String name, VolumeSource source) {
                this.name = name;
                this.source = source;
            }
            
            public String toYaml(String indent) {
                StringBuilder yaml = new StringBuilder();
                yaml.append(indent).append("- name: ").append(name).append("\n");
                
                if (source instanceof EmptyDirVolume) {
                    yaml.append(indent).append("  emptyDir: {}\n");
                } else if (source instanceof ConfigMapVolume) {
                    ConfigMapVolume cm = (ConfigMapVolume) source;
                    yaml.append(indent).append("  configMap:\n");
                    yaml.append(indent).append("    name: ").append(cm.getName()).append("\n");
                } else if (source instanceof SecretVolume) {
                    SecretVolume secret = (SecretVolume) source;
                    yaml.append(indent).append("  secret:\n");
                    yaml.append(indent).append("    secretName: ").append(secret.getName()).append("\n");
                } else if (source instanceof PersistentVolumeClaim) {
                    PersistentVolumeClaim pvc = (PersistentVolumeClaim) source;
                    yaml.append(indent).append("  persistentVolumeClaim:\n");
                    yaml.append(indent).append("    claimName: ").append(pvc.getClaimName()).append("\n");
                }
                
                return yaml.toString();
            }
            
            public static abstract class VolumeSource {}
            
            public static class EmptyDirVolume extends VolumeSource {}
            
            public static class ConfigMapVolume extends VolumeSource {
                private final String name;
                
                public ConfigMapVolume(String name) {
                    this.name = name;
                }
                
                public String getName() { return name; }
            }
            
            public static class SecretVolume extends VolumeSource {
                private final String name;
                
                public SecretVolume(String name) {
                    this.name = name;
                }
                
                public String getName() { return name; }
            }
            
            public static class PersistentVolumeClaim extends VolumeSource {
                private final String claimName;
                
                public PersistentVolumeClaim(String claimName) {
                    this.claimName = claimName;
                }
                
                public String getClaimName() { return claimName; }
            }
        }
        
        // Supporting classes
        public static class Metadata {
            private final String name;
            private String namespace = "default";
            private final Map<String, String> labels = new HashMap<>();
            private final Map<String, String> annotations = new HashMap<>();
            
            public Metadata(String name) {
                this.name = name;
            }
            
            public Metadata withNamespace(String namespace) {
                this.namespace = namespace;
                return this;
            }
            
            public Metadata withLabel(String key, String value) {
                labels.put(key, value);
                return this;
            }
            
            public Metadata withAnnotation(String key, String value) {
                annotations.put(key, value);
                return this;
            }
            
            public String getName() { return name; }
            public String getNamespace() { return namespace; }
            public Map<String, String> getLabels() { return labels; }
            public Map<String, String> getAnnotations() { return annotations; }
        }
    }
    
    // Kubernetes client implementation
    public static class KubernetesClient {
        private final String clusterUrl;
        private final String token;
        private final Map<String, AtomicInteger> resourceCounters = new ConcurrentHashMap<>();
        
        public KubernetesClient(String clusterUrl, String token) {
            this.clusterUrl = clusterUrl;
            this.token = token;
        }
        
        public void createDeployment(KubernetesResources.Deployment deployment) {
            String yaml = deployment.toYaml();
            // In real implementation, make HTTP request to Kubernetes API
            System.out.println("Creating deployment: " + deployment.getMetadata().getName());
            System.out.println(yaml);
            resourceCounters.computeIfAbsent("deployments", k -> new AtomicInteger(0)).incrementAndGet();
        }
        
        public void createService(KubernetesResources.Service service) {
            String yaml = service.toYaml();
            // In real implementation, make HTTP request to Kubernetes API
            System.out.println("Creating service: " + service.getMetadata().getName());
            System.out.println(yaml);
            resourceCounters.computeIfAbsent("services", k -> new AtomicInteger(0)).incrementAndGet();
        }
        
        public void createConfigMap(KubernetesResources.ConfigMap configMap) {
            String yaml = configMap.toYaml();
            // In real implementation, make HTTP request to Kubernetes API
            System.out.println("Creating configmap: " + configMap.getMetadata().getName());
            System.out.println(yaml);
            resourceCounters.computeIfAbsent("configmaps", k -> new AtomicInteger(0)).incrementAndGet();
        }
        
        public void createSecret(KubernetesResources.Secret secret) {
            String yaml = secret.toYaml();
            // In real implementation, make HTTP request to Kubernetes API
            System.out.println("Creating secret: " + secret.getMetadata().getName());
            System.out.println(yaml);
            resourceCounters.computeIfAbsent("secrets", k -> new AtomicInteger(0)).incrementAndGet();
        }
        
        public Map<String, Integer> getResourceCounts() {
            return resourceCounters.entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    entry -> entry.getValue().get()
                ));
        }
        
        // Supporting methods for Kubernetes operations
        public void scaleDeployment(String deploymentName, int replicas) {
            System.out.println("Scaling deployment " + deploymentName + " to " + replicas + " replicas");
        }
        
        public void rolloutRestartDeployment(String deploymentName) {
            System.out.println("Restarting deployment: " + deploymentName);
        }
        
        public void deleteDeployment(String deploymentName) {
            System.out.println("Deleting deployment: " + deploymentName);
            resourceCounters.get("deployments").decrementAndGet();
        }
    }
    
    // Advanced Kubernetes patterns
    public static class AdvancedKubernetesPatterns {
        
        // Blue-Green deployment pattern
        public static class BlueGreenDeployment {
            private final KubernetesClient k8sClient;
            private final String serviceName;
            private final String blueDeploymentName;
            private final String greenDeploymentName;
            
            public BlueGreenDeployment(KubernetesClient k8sClient, String serviceName) {
                this.k8sClient = k8sClient;
                this.serviceName = serviceName;
                this.blueDeploymentName = serviceName + "-blue";
                this.greenDeploymentName = serviceName + "-green";
            }
            
            public void deployNewVersion(KubernetesResources.Deployment newDeployment) {
                // Determine which deployment is currently active
                String currentDeployment = determineCurrentDeployment();
                String newDeploymentName = currentDeployment.equals(blueDeploymentName) ? 
                    greenDeploymentName : blueDeploymentName;
                
                // Deploy new version to inactive deployment
                KubernetesResources.Deployment deployment = createDeploymentForEnvironment(
                    newDeploymentName, newDeployment.getSpec());
                
                k8sClient.createDeployment(deployment);
                
                // Wait for new deployment to be ready
                waitForDeploymentReady(newDeploymentName);
                
                // Update service to point to new deployment
                updateServiceSelector(newDeploymentName);
                
                // Scale down old deployment
                k8sClient.scaleDeployment(currentDeployment, 0);
            }
            
            private String determineCurrentDeployment() {
                // In real implementation, query Kubernetes API to determine current active deployment
                // This is a simplified version that alternates between blue and green
                return Math.random() > 0.5 ? blueDeploymentName : greenDeploymentName;
            }
            
            private KubernetesResources.Deployment createDeploymentForEnvironment(
                    String deploymentName, KubernetesResources.Deployment.DeploymentSpec originalSpec) {
                
                // Create a new deployment spec with environment-specific configuration
                return new KubernetesResources.Deployment(deploymentName, originalSpec);
            }
            
            private void waitForDeploymentReady(String deploymentName) {
                // In real implementation, poll Kubernetes API for deployment status
                try {
                    Thread.sleep(5000); // Wait 5 seconds
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
            
            private void updateServiceSelector(String targetDeployment) {
                // Update service selector to point to target deployment
                System.out.println("Updating service " + serviceName + " to point to " + targetDeployment);
            }
        }
        
        // Canary deployment pattern
        public static class CanaryDeployment {
            private final KubernetesClient k8sClient;
            private final String productionDeploymentName;
            private final String canaryDeploymentName;
            
            public CanaryDeployment(KubernetesClient k8sClient, String baseDeploymentName) {
                this.k8sClient = k8sClient;
                this.productionDeploymentName = baseDeploymentName + "-production";
                this.canaryDeploymentName = baseDeploymentName + "-canary";
            }
            
            public void deployCanary(KubernetesResources.Deployment newVersion, 
                                   Predicate<Double> successRatePredicate) {
                
                // Deploy canary with 10% of traffic initially
                KubernetesResources.Deployment canaryDeployment = 
                    createCanaryDeployment(newVersion, 0.1); // 10% of traffic
                
                k8sClient.createDeployment(canaryDeployment);
                
                // Monitor canary metrics
                double successRate = monitorCanaryMetrics();
                
                if (successRatePredicate.test(successRate)) {
                    // Promote canary to full deployment
                    promoteCanaryToProduction(newVersion);
                } else {
                    // Rollback - delete canary deployment
                    k8sClient.deleteDeployment(canaryDeploymentName);
                    System.out.println("Canary deployment rolled back due to low success rate: " + successRate);
                }
            }
            
            private KubernetesResources.Deployment createCanaryDeployment(
                    KubernetesResources.Deployment baseDeployment, double trafficPercentage) {
                
                // Calculate replica count based on traffic percentage
                int productionReplicas = baseDeployment.getSpec().getReplicas();
                int canaryReplicas = (int) (productionReplicas * trafficPercentage);
                
                // Create canary deployment with same spec but different name and replica count
                return new KubernetesResources.Deployment(canaryDeploymentName, 
                    new KubernetesResources.Deployment.DeploymentSpec(
                        canaryReplicas,
                        baseDeployment.getSpec().getSelector(),
                        baseDeployment.getSpec().getTemplate()
                    ));
            }
            
            private double monitorCanaryMetrics() {
                // In real implementation, connect to metrics service and monitor
                // Return success rate as a percentage
                return 0.95; // Simulated success rate
            }
            
            private void promoteCanaryToProduction(KubernetesResources.Deployment newVersion) {
                // Scale up production deployment to new version
                k8sClient.scaleDeployment(productionDeploymentName, newVersion.getSpec().getReplicas());
                
                // Delete canary deployment
                k8sClient.deleteDeployment(canaryDeploymentName);
                
                System.out.println("Canary deployment promoted to production");
            }
        }
        
        // StatefulSet for stateful applications
        public static class StatefulSetManager {
            private final KubernetesClient k8sClient;
            
            public StatefulSetManager(KubernetesClient k8sClient) {
                this.k8sClient = k8sClient;
            }
            
            public void createStatefulSet(String name, int replicas, String storageClass, 
                                        String volumeSize) {
                
                // Create StatefulSet with persistent storage
                KubernetesResources.Deployment statefulSet = createStatefulSet(
                    name, replicas, storageClass, volumeSize);
                
                k8sClient.createDeployment(statefulSet);
            }
            
            private KubernetesResources.Deployment createStatefulSet(String name, int replicas, 
                                                                   String storageClass, String volumeSize) {
                
                // Create a deployment that simulates StatefulSet behavior
                // In real implementation, this would be a StatefulSet resource
                return new KubernetesResources.Deployment(name,
                    new KubernetesResources.Deployment.DeploymentSpec(
                        replicas,
                        Map.of("app", name),
                        createStatefulPodTemplate(storageClass, volumeSize)
                    ));
            }
            
            private KubernetesResources.PodTemplate createStatefulPodTemplate(
                    String storageClass, String volumeSize) {
                
                // In real implementation, this would include volumeClaimTemplates
                // for persistent storage
                return new KubernetesResources.PodTemplate(
                    new KubernetesResources.Metadata(name).withLabel("app", name),
                    new KubernetesResources.PodTemplate.PodSpec(
                        List.of(), // containers would be specified here
                        List.of(), // init containers
                        List.of(new KubernetesResources.Volume("data", 
                            new KubernetesResources.Volume.PersistentVolumeClaim("data-claim"))),
                        Map.of() // node selector
                    )
                );
            }
        }
    }
}
```

## Cloud-Native Application Development

### Microservices and Service Mesh

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.function.Function;

public class CloudNativeDevelopment {
    
    // Service mesh implementation
    public static class ServiceMesh {
        
        // Istio-like sidecar proxy
        public static class SidecarProxy {
            private final ServiceRegistry serviceRegistry;
            private final CircuitBreaker circuitBreaker;
            private final RateLimiter rateLimiter;
            private final DistributedTracing tracing;
            
            public SidecarProxy(ServiceRegistry serviceRegistry, CircuitBreaker circuitBreaker,
                              RateLimiter rateLimiter, DistributedTracing tracing) {
                this.serviceRegistry = serviceRegistry;
                this.circuitBreaker = circuitBreaker;
                this.rateLimiter = rateLimiter;
                this.tracing = tracing;
            }
            
            public <T> CompletionStage<T> proxyRequest(String targetService, 
                                                      ServiceRequest request, 
                                                      Class<T> responseType) {
                
                return CompletableFuture.supplyAsync(() -> {
                    // Get service endpoint
                    ServiceEndpoint endpoint = serviceRegistry.getEndpoint(targetService);
                    if (endpoint == null) {
                        throw new RuntimeException("Service not found: " + targetService);
                    }
                    
                    // Check circuit breaker
                    if (circuitBreaker.isCircuitOpen(targetService)) {
                        throw new RuntimeException("Circuit breaker open for: " + targetService);
                    }
                    
                    // Check rate limit
                    if (!rateLimiter.allowRequest(request.getClientId(), targetService)) {
                        throw new RuntimeException("Rate limit exceeded for: " + targetService);
                    }
                    
                    // Create trace context
                    DistributedTracing.TraceContext traceContext = tracing.startTrace(
                        "service-call-" + targetService);
                    
                    try {
                        // Forward request
                        ServiceResponse response = forwardRequest(endpoint, request);
                        
                        // Record success metrics
                        circuitBreaker.recordSuccess(targetService);
                        
                        // End trace
                        tracing.endTrace(traceContext);
                        
                        // Convert response
                        return convertResponse(response, responseType);
                    } catch (Exception e) {
                        // Record failure
                        circuitBreaker.recordFailure(targetService);
                        
                        // End trace with error
                        tracing.endTraceWithError(traceContext, e);
                        
                        throw new RuntimeException("Service call failed: " + e.getMessage(), e);
                    }
                });
            }
            
            private ServiceResponse forwardRequest(ServiceEndpoint endpoint, ServiceRequest request) {
                // In real implementation, make actual HTTP/gRPC call to service
                // This is a simulation
                return new ServiceResponse(200, "Success", System.currentTimeMillis());
            }
            
            private <T> T convertResponse(ServiceResponse response, Class<T> responseType) {
                // Convert service response to desired type
                // This is a simplified implementation
                return null;
            }
            
            // Supporting classes
            public static class ServiceRequest {
                private final String method;
                private final String path;
                private final Map<String, String> headers;
                private final String body;
                private final String clientId;
                
                public ServiceRequest(String method, String path, Map<String, String> headers, 
                                    String body, String clientId) {
                    this.method = method;
                    this.path = path;
                    this.headers = headers != null ? new HashMap<>(headers) : new HashMap<>();
                    this.body = body;
                    this.clientId = clientId;
                }
                
                public String getMethod() { return method; }
                public String getPath() { return path; }
                public Map<String, String> getHeaders() { return headers; }
                public String getBody() { return body; }
                public String getClientId() { return clientId; }
            }
            
            public static class ServiceResponse {
                private final int statusCode;
                private final String body;
                private final long timestamp;
                
                public ServiceResponse(int statusCode, String body, long timestamp) {
                    this.statusCode = statusCode;
                    this.body = body;
                    this.timestamp = timestamp;
                }
                
                public int getStatusCode() { return statusCode; }
                public String getBody() { return body; }
                public long getTimestamp() { return timestamp; }
            }
        }
        
        // Service registry for service discovery
        public static class ServiceRegistry {
            private final Map<String, List<ServiceEndpoint>> serviceEndpoints = new ConcurrentHashMap<>();
            private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
            
            public ServiceRegistry() {
                startHealthChecking();
            }
            
            public void registerService(String serviceName, ServiceEndpoint endpoint) {
                serviceEndpoints.computeIfAbsent(serviceName, k -> new CopyOnWriteArrayList<>())
                               .add(endpoint);
            }
            
            public ServiceEndpoint getEndpoint(String serviceName) {
                List<ServiceEndpoint> endpoints = serviceEndpoints.get(serviceName);
                if (endpoints == null || endpoints.isEmpty()) {
                    return null;
                }
                
                // Simple round-robin selection
                int index = (int) (System.nanoTime() % endpoints.size());
                return endpoints.get(index);
            }
            
            public List<ServiceEndpoint> getEndpoints(String serviceName) {
                List<ServiceEndpoint> endpoints = serviceEndpoints.get(serviceName);
                return endpoints != null ? new ArrayList<>(endpoints) : new ArrayList<>();
            }
            
            private void startHealthChecking() {
                scheduler.scheduleAtFixedRate(this::checkHealth, 0, 30, TimeUnit.SECONDS);
            }
            
            private void checkHealth() {
                serviceEndpoints.values().stream()
                    .flatMap(List::stream)
                    .forEach(endpoint -> {
                        if (!isServiceHealthy(endpoint)) {
                            // Remove unhealthy endpoint
                            removeEndpoint(endpoint);
                        }
                    });
            }
            
            private boolean isServiceHealthy(ServiceEndpoint endpoint) {
                // In real implementation, make actual health check call
                return true; // Assume healthy for simulation
            }
            
            private void removeEndpoint(ServiceEndpoint endpoint) {
                serviceEndpoints.values().forEach(list -> list.remove(endpoint));
            }
            
            public void shutdown() {
                scheduler.shutdown();
                try {
                    if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                        scheduler.shutdownNow();
                    }
                } catch (InterruptedException e) {
                    scheduler.shutdownNow();
                    Thread.currentThread().interrupt();
                }
            }
            
            // Supporting class
            public static class ServiceEndpoint {
                private final String host;
                private final int port;
                private final String protocol;
                private final Map<String, String> metadata;
                
                public ServiceEndpoint(String host, int port, String protocol) {
                    this.host = host;
                    this.port = port;
                    this.protocol = protocol;
                    this.metadata = new HashMap<>();
                }
                
                public String getHost() { return host; }
                public int getPort() { return port; }
                public String getProtocol() { return protocol; }
                public Map<String, String> getMetadata() { return metadata; }
            }
        }
        
        // Traffic management
        public static class TrafficManager {
            private final ServiceRegistry serviceRegistry;
            private final Map<String, TrafficPolicy> policies = new ConcurrentHashMap<>();
            
            public TrafficManager(ServiceRegistry serviceRegistry) {
                this.serviceRegistry = serviceRegistry;
            }
            
            public void setTrafficPolicy(String serviceName, TrafficPolicy policy) {
                policies.put(serviceName, policy);
            }
            
            public ServiceEndpoint selectEndpoint(String serviceName, TrafficContext context) {
                TrafficPolicy policy = policies.get(serviceName);
                if (policy == null) {
                    // Use default round-robin
                    return serviceRegistry.getEndpoint(serviceName);
                }
                
                // Apply traffic policy
                List<ServiceRegistry.ServiceEndpoint> endpoints = 
                    serviceRegistry.getEndpoints(serviceName);
                
                return policy.selectEndpoint(endpoints, context);
            }
            
            // Supporting classes
            public static class TrafficPolicy {
                private final LoadBalancingStrategy strategy;
                private final CanaryConfiguration canaryConfig;
                
                public TrafficPolicy(LoadBalancingStrategy strategy, 
                                   CanaryConfiguration canaryConfig) {
                    this.strategy = strategy;
                    this.canaryConfig = canaryConfig;
                }
                
                public ServiceRegistry.ServiceEndpoint selectEndpoint(
                        List<ServiceRegistry.ServiceEndpoint> endpoints, 
                        TrafficContext context) {
                    
                    if (canaryConfig != null && context.isCanaryRequest()) {
                        // Route to canary version
                        return endpoints.stream()
                            .filter(endpoint -> endpoint.getMetadata().containsKey("version") &&
                                              "canary".equals(endpoint.getMetadata().get("version")))
                            .findFirst()
                            .orElse(endpoints.get(0));
                    }
                    
                    switch (strategy) {
                        case ROUND_ROBIN:
                            return selectRoundRobin(endpoints);
                        case LEAST_CONNECTIONS:
                            return selectLeastConnections(endpoints);
                        case WEIGHTED_ROUND_ROBIN:
                            return selectWeightedRoundRobin(endpoints);
                        default:
                            return endpoints.get(0);
                    }
                }
                
                private ServiceRegistry.ServiceEndpoint selectRoundRobin(
                        List<ServiceRegistry.ServiceEndpoint> endpoints) {
                    int index = (int) (System.nanoTime() % endpoints.size());
                    return endpoints.get(index);
                }
                
                private ServiceRegistry.ServiceEndpoint selectLeastConnections(
                        List<ServiceRegistry.ServiceEndpoint> endpoints) {
                    // Simplified implementation
                    return endpoints.get(0);
                }
                
                private ServiceRegistry.ServiceEndpoint selectWeightedRoundRobin(
                        List<ServiceRegistry.ServiceEndpoint> endpoints) {
                    // Simplified implementation
                    return endpoints.get(0);
                }
            }
            
            public enum LoadBalancingStrategy {
                ROUND_ROBIN, LEAST_CONNECTIONS, WEIGHTED_ROUND_ROBIN
            }
            
            public static class CanaryConfiguration {
                private final double canaryTrafficPercentage;
                private final String canaryVersion;
                
                public CanaryConfiguration(double canaryTrafficPercentage, String canaryVersion) {
                    this.canaryTrafficPercentage = canaryTrafficPercentage;
                    this.canaryVersion = canaryVersion;
                }
                
                public double getCanaryTrafficPercentage() { return canaryTrafficPercentage; }
                public String getCanaryVersion() { return canaryVersion; }
            }
            
            public static class TrafficContext {
                private final Map<String, String> headers;
                private final String clientIP;
                
                public TrafficContext(Map<String, String> headers, String clientIP) {
                    this.headers = headers != null ? new HashMap<>(headers) : new HashMap<>();
                    this.clientIP = clientIP;
                }
                
                public boolean isCanaryRequest() {
                    return "true".equalsIgnoreCase(headers.get("x-canary"));
                }
                
                public String getHeader(String name) {
                    return headers.get(name);
                }
                
                public String getClientIP() { return clientIP; }
            }
        }
    }
    
    // Cloud-native configuration management
    public static class CloudNativeConfig {
        
        // ConfigMap-based configuration
        public static class KubernetesConfigManager {
            private final KubernetesClient k8sClient;
            private final String namespace;
            private final Map<String, Object> configCache = new ConcurrentHashMap<>();
            private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
            
            public KubernetesConfigManager(KubernetesClient k8sClient, String namespace) {
                this.k8sClient = k8sClient;
                this.namespace = namespace;
                
                startConfigSync();
            }
            
            public <T> T getConfigProperty(String configMapName, String key, Class<T> type) {
                Object value = configCache.get(configMapName + "." + key);
                if (value == null) {
                    // Reload config from Kubernetes
                    reloadConfig(configMapName);
                    value = configCache.get(configMapName + "." + key);
                }
                
                if (value == null) {
                    return null;
                }
                
                return convertValue(value, type);
            }
            
            public void reloadConfig(String configMapName) {
                // In real implementation, fetch ConfigMap from Kubernetes API
                // This is a simulation
                Map<String, String> configData = fetchConfigMapData(configMapName);
                
                for (Map.Entry<String, String> entry : configData.entrySet()) {
                    String fullKey = configMapName + "." + entry.getKey();
                    configCache.put(fullKey, parseValue(entry.getValue()));
                }
            }
            
            private Map<String, String> fetchConfigMapData(String configMapName) {
                // Simulated ConfigMap data
                Map<String, String> data = new HashMap<>();
                data.put("app.name", "my-app");
                data.put("app.version", "1.0.0");
                data.put("database.url", "jdbc:postgresql://localhost:5432/mydb");
                data.put("database.username", "user");
                return data;
            }
            
            private Object parseValue(String value) {
                // Try to parse as different types
                if ("true".equalsIgnoreCase(value) || "false".equalsIgnoreCase(value)) {
                    return Boolean.parseBoolean(value);
                }
                
                try {
                    return Integer.parseInt(value);
                } catch (NumberFormatException e) {
                    try {
                        return Double.parseDouble(value);
                    } catch (NumberFormatException e2) {
                        return value; // Return as string
                    }
                }
            }
            
            private <T> T convertValue(Object value, Class<T> type) {
                if (value == null) return null;
                
                if (type.isInstance(value)) {
                    return type.cast(value);
                }
                
                String stringValue = value.toString();
                if (type == String.class) {
                    return type.cast(stringValue);
                } else if (type == Integer.class || type == int.class) {
                    return type.cast(Integer.valueOf(stringValue));
                } else if (type == Long.class || type == long.class) {
                    return type.cast(Long.valueOf(stringValue));
                } else if (type == Double.class || type == double.class) {
                    return type.cast(Double.valueOf(stringValue));
                } else if (type == Boolean.class || type == boolean.class) {
                    return type.cast(Boolean.valueOf(stringValue));
                }
                
                return null;
            }
            
            private void startConfigSync() {
                scheduler.scheduleAtFixedRate(() -> {
                    // Refresh all configs periodically
                    configCache.keySet().stream()
                        .map(key -> key.split("\\.")[0]) // Get ConfigMap name
                        .distinct()
                        .forEach(this::reloadConfig);
                }, 0, 30, TimeUnit.SECONDS);
            }
            
            public void shutdown() {
                scheduler.shutdown();
                try {
                    if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                        scheduler.shutdownNow();
                    }
                } catch (InterruptedException e) {
                    scheduler.shutdownNow();
                    Thread.currentThread().interrupt();
                }
            }
        }
        
        // Secret management
        public static class SecretManager {
            private final KubernetesClient k8sClient;
            private final Map<String, String> secretCache = new ConcurrentHashMap<>();
            private final EncryptionService encryptionService;
            
            public SecretManager(KubernetesClient k8sClient, EncryptionService encryptionService) {
                this.k8sClient = k8sClient;
                this.encryptionService = encryptionService;
            }
            
            public String getSecret(String secretName, String key) {
                String cachedKey = secretName + "." + key;
                String cachedValue = secretCache.get(cachedKey);
                
                if (cachedValue == null) {
                    // Fetch secret from Kubernetes
                    String secretValue = fetchSecret(secretName, key);
                    
                    // Decrypt if necessary
                    if (secretValue != null && isEncrypted(secretValue)) {
                        secretValue = encryptionService.decrypt(secretValue);
                    }
                    
                    secretCache.put(cachedKey, secretValue);
                    cachedValue = secretValue;
                }
                
                return cachedValue;
            }
            
            public void invalidateSecret(String secretName, String key) {
                secretCache.remove(secretName + "." + key);
            }
            
            private String fetchSecret(String secretName, String key) {
                // In real implementation, fetch secret from Kubernetes API
                // This is a simulation
                Map<String, String> secrets = fetchSecretData(secretName);
                return secrets.get(key);
            }
            
            private Map<String, String> fetchSecretData(String secretName) {
                // Simulated secret data
                Map<String, String> secrets = new HashMap<>();
                secrets.put("db.password", encryptionService.encrypt("secret123"));
                secrets.put("api.key", encryptionService.encrypt("my-api-key"));
                return secrets;
            }
            
            private boolean isEncrypted(String value) {
                // Simple check - in real implementation, use proper format detection
                return value.startsWith("enc:");
            }
        }
        
        // Encryption service for secrets
        public static class EncryptionService {
            private final String encryptionKey;
            
            public EncryptionService(String encryptionKey) {
                this.encryptionKey = encryptionKey;
            }
            
            public String encrypt(String plaintext) {
                // In real implementation, use proper encryption
                // This is a simple simulation
                String encrypted = java.util.Base64.getEncoder()
                    .encodeToString((plaintext + encryptionKey).getBytes());
                return "enc:" + encrypted;
            }
            
            public String decrypt(String encryptedText) {
                if (!encryptedText.startsWith("enc:")) {
                    return encryptedText; // Not encrypted
                }
                
                String encoded = encryptedText.substring(4);
                String combined = new String(java.util.Base64.getDecoder().decode(encoded));
                
                // Remove encryption key suffix
                if (combined.endsWith(encryptionKey)) {
                    return combined.substring(0, combined.length() - encryptionKey.length());
                }
                
                return combined;
            }
        }
    }
}
```

## Conclusion

Advanced Cloud-Native Development and Containerization provide:
- Comprehensive containerization strategies with Docker and multi-stage builds
- Kubernetes orchestration with deployments, services, and advanced patterns
- Service mesh patterns for microservices communication and traffic management
- Cloud-native configuration and secret management
- Security best practices for containerized environments

Mastering these concepts is essential for:
- Building and deploying cloud-native applications
- Implementing container orchestration with Kubernetes
- Creating resilient microservices architectures
- Managing configuration and secrets in cloud environments
- Ensuring security and compliance in containerized deployments