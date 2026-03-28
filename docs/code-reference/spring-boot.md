# Spring Boot — MDM Core API

## Project Structure

```
mdm-core-api/
├── pom.xml
└── src/main/java/com/mdm/
    ├── MdmCoreApplication.java
    ├── config/
    │   ├── AemConfig.java
    │   ├── MongoConfig.java
    │   └── RedisConfig.java
    ├── model/
    │   ├── MasterDataObject.java
    │   ├── MdmEvent.java
    │   ├── ChangeRecord.java
    │   └── ConsumerRegistration.java
    ├── service/
    │   ├── MdmCoreService.java
    │   ├── AemEventPublisher.java
    │   ├── FieldChangeTracker.java
    │   └── ReadCacheService.java
    ├── listener/
    │   └── MongoChangeStreamListener.java
    ├── repository/
    │   ├── MasterDataRepository.java
    │   └── ChangeRecordRepository.java
    └── controller/
        └── MdmController.java
```

---

## `pom.xml` — Dependencies

```xml
<project>
  <groupId>com.mdm</groupId>
  <artifactId>mdm-core-api</artifactId>
  <version>1.0.0</version>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
  </parent>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Solace AEM Spring Boot Starter -->
    <dependency>
      <groupId>com.solace.spring.boot</groupId>
      <artifactId>solace-spring-boot-starter</artifactId>
      <version>4.3.0</version>
    </dependency>
    <dependency>
      <groupId>com.solace.spring.cloud</groupId>
      <artifactId>spring-cloud-starter-stream-solace</artifactId>
      <version>4.2.0</version>
    </dependency>

    <!-- Micrometer + OpenTelemetry -->
    <dependency>
      <groupId>io.micrometer</groupId>
      <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
      <groupId>io.opentelemetry.instrumentation</groupId>
      <artifactId>opentelemetry-spring-boot-starter</artifactId>
      <version>2.1.0</version>
    </dependency>

    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>
  </dependencies>
</project>
```

---

## `application.yml`

```yaml
spring:
  application:
    name: mdm-core-api
  data:
    mongodb:
      uri: ${MONGODB_URI}
      database: mdm
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      ssl.enabled: true
  cloud:
    stream:
      solace:
        binder:
          credentials:
            activeKey: default
            keys:
              default:
                host: ${AEM_HOST}
                msgVpn: ${AEM_MSG_VPN}
                clientUsername: ${AEM_CLIENT_USERNAME}
                clientPassword: ${AEM_CLIENT_PASSWORD}
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: ${XSUAA_JWKS_URI}
          issuer-uri: ${XSUAA_ISSUER_URI}

mdm:
  cache.ttl-seconds: 300
  aem.topic-prefix: mdm
```

---

## `MasterDataObject.java` — MongoDB Document

```java
@Data
@Document(collection = "master_data_objects")
public class MasterDataObject {

    @Id
    private String id;                           // e.g. MDM-CUST-001

    private String objectType;                   // Customer, Vendor, Material

    @Version
    private Long version;

    private Map<String, Object> coreAttributes;  // Lead system fields

    // Extension attributes per satellite system
    // { "CRM": { "tier": "Gold" }, "ERP": { "class": "A" } }
    private Map<String, Map<String, Object>> extensions;

    private Metadata metadata;

    private List<String> lastChangedFields;      // Fields changed in last write

    @Data
    public static class Metadata {
        private String leadSystem;
        private String siteId;                   // For locally managed objects
        private String createdBy;
        private Instant createdAt;
        private String updatedBy;
        private Instant updatedAt;
    }
}
```

---

## `MdmCoreService.java` — Business Logic

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MdmCoreService {

    private final MasterDataRepository masterDataRepository;
    private final FieldChangeTracker fieldChangeTracker;
    private final AemEventPublisher aemEventPublisher;
    private final ReadCacheService readCacheService;

    public MasterDataObject createObject(MasterDataObject object) {
        String actor = getCurrentUser();
        object.getMetadata().setCreatedBy(actor);
        object.getMetadata().setCreatedAt(Instant.now());
        object.getMetadata().setUpdatedBy(actor);
        object.getMetadata().setUpdatedAt(Instant.now());

        MasterDataObject saved = masterDataRepository.save(object);
        fieldChangeTracker.recordChange(saved, null, actor, "created");
        aemEventPublisher.publish(saved, "created", null);
        readCacheService.put(saved);
        return saved;
    }

    public MasterDataObject updateObject(String id, MasterDataObject incoming) {
        MasterDataObject existing = masterDataRepository.findById(id)
            .orElseThrow(() -> new ObjectNotFoundException(id));

        List<String> changedFields = fieldChangeTracker.computeChangedFields(existing, incoming);

        if (changedFields.isEmpty()) {
            log.info("No field changes for objectId={}, skipping update", id);
            return existing;
        }

        String actor = getCurrentUser();
        incoming.setId(id);
        incoming.getMetadata().setUpdatedBy(actor);
        incoming.getMetadata().setUpdatedAt(Instant.now());
        incoming.getMetadata().setCreatedAt(existing.getMetadata().getCreatedAt());
        incoming.getMetadata().setCreatedBy(existing.getMetadata().getCreatedBy());
        incoming.setLastChangedFields(changedFields);

        MasterDataObject saved = masterDataRepository.save(incoming);
        fieldChangeTracker.recordChange(saved, changedFields, actor, "changed");
        aemEventPublisher.publish(saved, "changed", changedFields);
        readCacheService.invalidate(id, saved.getObjectType());
        return saved;
    }

    public Optional<MasterDataObject> getObject(String id, String objectType) {
        return readCacheService.get(id, objectType)
            .or(() -> {
                Optional<MasterDataObject> fromDb = masterDataRepository.findById(id);
                fromDb.ifPresent(readCacheService::put);
                return fromDb;
            });
    }

    private String getCurrentUser() {
        return SecurityContextHolder.getContext().getAuthentication().getName();
    }
}
```

---

## `FieldChangeTracker.java` — Field-Level Diff

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class FieldChangeTracker {

    private final ChangeRecordRepository changeRecordRepository;

    /**
     * Returns dot-notation paths of changed fields.
     * e.g. ["coreAttributes.country", "extensions.CRM.tier"]
     */
    public List<String> computeChangedFields(MasterDataObject previous, MasterDataObject current) {
        List<String> changedFields = new ArrayList<>();

        diffMaps("coreAttributes",
            previous.getCoreAttributes(), current.getCoreAttributes(), changedFields);

        Set<String> allSystems = new HashSet<>();
        if (previous.getExtensions() != null) allSystems.addAll(previous.getExtensions().keySet());
        if (current.getExtensions() != null) allSystems.addAll(current.getExtensions().keySet());

        for (String system : allSystems) {
            Map<String, Object> prevExt = previous.getExtensions() != null
                ? previous.getExtensions().getOrDefault(system, Map.of()) : Map.of();
            Map<String, Object> currExt = current.getExtensions() != null
                ? current.getExtensions().getOrDefault(system, Map.of()) : Map.of();
            diffMaps("extensions." + system, prevExt, currExt, changedFields);
        }

        return changedFields;
    }

    private void diffMaps(String prefix, Map<String, Object> prev,
                          Map<String, Object> curr, List<String> result) {
        if (prev == null) prev = Map.of();
        if (curr == null) curr = Map.of();

        Set<String> allKeys = new HashSet<>(prev.keySet());
        allKeys.addAll(curr.keySet());

        for (String key : allKeys) {
            if (!Objects.equals(prev.get(key), curr.get(key))) {
                result.add(prefix + "." + key);
            }
        }
    }

    public void recordChange(MasterDataObject object, List<String> changedFields,
                             String changedBy, String eventType) {
        changeRecordRepository.save(ChangeRecord.builder()
            .objectId(object.getId())
            .objectType(object.getObjectType())
            .version(object.getVersion())
            .eventType(eventType)
            .changedFields(changedFields)
            .changedBy(changedBy)
            .changedAt(Instant.now())
            .build());
    }
}
```

---

## `AemEventPublisher.java` — AEM Publishing

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class AemEventPublisher {

    private final JCSMPSession jcsmpSession;
    private final ObjectMapper objectMapper;

    @Value("${mdm.aem.topic-prefix}")
    private String topicPrefix;

    public void publish(MasterDataObject object, String eventType, List<String> changedFields) {
        try {
            XMLMessageProducer producer = jcsmpSession.getMessageProducer(
                new JCSMPStreamingPublishEventHandler() {
                    @Override public void responseReceived(String messageID) {}
                    @Override public void handleError(String messageID, JCSMPException e, long ts) {
                        log.error("AEM publish error messageId={}", messageID, e);
                    }
                });

            MdmEvent event = MdmEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .eventType("mdm.object." + eventType)
                .objectType(object.getObjectType())
                .objectId(object.getId())
                .version(object.getVersion())
                .leadSystem(object.getMetadata().getLeadSystem())
                .changedFields(changedFields)
                .occurredAt(Instant.now())
                .correlationId(UUID.randomUUID().toString())
                .build();

            TextMessage message = JCSMPFactory.onlyInstance().createMessage(TextMessage.class);
            message.setText(objectMapper.writeValueAsString(event));
            message.setDeliveryMode(DeliveryMode.PERSISTENT);

            SDTMap userProps = JCSMPFactory.onlyInstance().createMap();
            userProps.putString("objectType", object.getObjectType());
            userProps.putString("eventType", eventType);
            if (changedFields != null) {
                userProps.putString("changedFields", String.join(",", changedFields));
            }
            message.setProperties(userProps);

            // Topic: mdm/customer/changed/v1/S4HANA
            String topicName = String.format("%s/%s/%s/v1/%s",
                topicPrefix,
                object.getObjectType().toLowerCase(),
                eventType,
                object.getMetadata().getLeadSystem());

            producer.send(message, JCSMPFactory.onlyInstance().createTopic(topicName));
            log.info("Published eventType={} objectId={} topic={}", eventType, object.getId(), topicName);

        } catch (Exception e) {
            log.error("Failed to publish AEM event objectId={}", object.getId(), e);
            throw new AemPublishException("AEM publish failed", e);
        }
    }
}
```

---

## `MongoChangeStreamListener.java` — CDC Fallback

```java
/**
 * Fallback CDC listener via MongoDB Change Streams.
 * Catches writes that bypass the service layer (direct DB writes, migrations).
 * Primary fanout is done in MdmCoreService — this is the safety net.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class MongoChangeStreamListener {

    private final MongoTemplate mongoTemplate;
    private final AemEventPublisher aemEventPublisher;

    @EventListener(ApplicationReadyEvent.class)
    public void startChangeStreamListener() {
        MessageListenerContainer container = new DefaultMessageListenerContainer(mongoTemplate);
        container.start();

        ChangeStreamRequest<MasterDataObject> request = ChangeStreamRequest
            .builder(this::handleChangeEvent)
            .collection("master_data_objects")
            .build();

        container.register(request, MasterDataObject.class);
        log.info("MongoDB Change Stream listener started");
    }

    private void handleChangeEvent(
            Message<ChangeStreamDocument<Document>, MasterDataObject> message) {

        ChangeStreamDocument<Document> raw = message.getRaw();
        MasterDataObject body = message.getBody();
        if (raw == null || body == null) return;

        log.warn("CDC fallback triggered opType={} objectId={}", raw.getOperationType(), body.getId());

        switch (raw.getOperationType()) {
            case INSERT  -> aemEventPublisher.publish(body, "created", null);
            case UPDATE,
                 REPLACE -> aemEventPublisher.publish(body, "changed",
                                body.getLastChangedFields() != null
                                    ? body.getLastChangedFields() : List.of());
            default      -> {}
        }
    }
}
```
