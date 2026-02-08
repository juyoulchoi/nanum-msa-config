# Config Server 설정 파일 가이드

이 문서는 Config Server에서 관리하는 설정 파일들의 구조, 로드 순서, 우선순위 규칙을 설명합니다.

## 1. 설정 로드 전략 (Composition)

각 마이크로서비스는 `spring.config.import`를 사용하여 필요한 설정 파일들을 조합(Composition)하여 로드합니다.

**예시 (`auth-service`):**
```properties
spring.config.import=optional:configserver:http://localhost:8888?name=application-service,db-postgres,auth-service
```

- **`application-service`**: 모든 서비스 공통 설정 (인프라, 로깅, Eureka 등)
- **`db-postgres`**: 기능별(Feature) 설정 (DB 접속 정보 등)
- **`auth-service`**: 서비스 고유 설정

## 2. 설정 로드 순서 및 우선순위

Spring Boot는 나중에 로드된 설정이 이전 설정을 덮어씁니다(Override). 따라서 `name` 파라미터의 **오른쪽에 있을수록 우선순위가 높습니다.**

### 상세 로드 단계 (Low -> High Priority)

#### 1단계: 공통 설정 (`application-service`)
가장 기본이 되는 설정입니다. `application-service.properties`에 정의된 **프로필 그룹**에 의해 환경별 인프라 설정이 자동으로 함께 로드됩니다.

- **`application-service.properties`**: 전역 기본값 (JPA, Logging 등)
- **`application-service-{profile}.properties`**: 환경별 기본값 (local, dev, prod)
- **`application-service-{profile}-infra.properties`**: 인프라 설정 (Redis, Mongo, Elastic)
- **`application-service-{profile}-kafka.properties`**: 메시징 설정 (Kafka)

> **참고:** 프로필 그룹 정의 예시 (`application-service.properties`)
> ```properties
> spring.profiles.group.local=local-infra,local-kafka
> ```

#### 2단계: 기능(Feature) 설정 (`db-*`)
공통 설정보다 우선순위가 높으며, 특정 기술 스택(DB 등)에 대한 설정을 적용합니다.

- **`db-postgres.properties`**, **`db-mysql.properties`**, **`db-mariadb.properties`** 등
- 공통 설정에 정의된 DB 값이 있다면 이 단계에서 덮어씌워집니다.

#### 3단계: 서비스 개별 설정 (`{service-name}`)
가장 높은 우선순위를 가집니다.

- **`{service-name}.properties`**: 서비스 고유 설정
- **`{service-name}-{profile}.properties`**: 서비스의 환경별 특화 설정

## 3. 우선순위 요약 (Priority Chart)

아래쪽에 있는 설정이 위쪽 설정을 덮어씁니다.

```text
[Low Priority]
    ↑   1. application-service.properties (Global Base)
    |   2. application-service-{profile}.properties (Env Base)
    |   3. application-service-{profile}-infra.properties (Grouped)
    |   4. application-service-{profile}-kafka.properties (Grouped)
    |   ---------------------------------------------------------
    |   5. db-{type}.properties (Feature Base)
    |   6. db-{type}-{profile}.properties (Feature Profile Config)
    |   ---------------------------------------------------------
    |   7. {service-name}.properties (Service Specific)
    |   8. {service-name}-{profile}.properties (Service Profile Specific)
[High Priority]
```

## 4. 오버라이드 예시

**시나리오:** `batch-service`가 `local` 환경에서 실행되지만, Kafka 포트를 기본값(`19092`)이 아닌 `9095`로 변경해야 하는 경우.

1. **공통 로드**: `application-service-local-kafka.properties`
   - `spring.kafka.bootstrap-servers = localhost:19092`
2. **개별 로드**: `batch-service-local.properties`
   - `spring.kafka.bootstrap-servers = localhost:9095`

**결과:** 최종적으로 `localhost:9095`가 적용됩니다.

# JWT_SECRET: _mk8+951BqW}oMB-@V3/AyG)Z:}n[9x[-)Of0H:%qDOcCUnoSpEoSKq_([$tX,OS#Ob<dDL%@[6F}f_<J%k=@H

Redis나 MongoDB 설정도 별도 파일(db-redis.properties, db-mongo.properties)로 분리하는 이유

필요한 서비스만 선택 로드: Redis를 쓰지 않는 서비스는 Redis 설정을 가져가지 않아 가벼워집니다.
유지보수 용이: 인프라 설정이 application-service.properties에 섞여 있지 않아 관리가 명확해집니다.

# application.properties (User Service 내부)
spring.application.name=user-service
spring.profiles.active=local

# name 파라미터에 필요한 설정 파일들을 나열 (순서 중요: 뒤쪽이 우선순위 높음)
spring.config.import=optional:configserver:http://localhost:8888?name=application-service,db-mariadb,db-redis,{user-service}

# Elasticsearch 공통 설정 (기본값: Dev/Docker 환경)
# db-elastic.properties
spring.elasticsearch.uris=${ELASTICSEARCH_URIS:http://dev-elastic:9200}
spring.elasticsearch.username=${ELASTICSEARCH_USERNAME:}
spring.elasticsearch.password=${ELASTICSEARCH_PASSWORD:}

# Elasticsearch Local 환경 설정 (localhost 사용)
# db-elastic-local.properties
spring.elasticsearch.uris=http://localhost:9200

# Kafka 공통 설정 (기본값: Dev/Docker 환경)
# db-kafka.properties
spring.kafka.bootstrap-servers=${KAFKA_BOOTSTRAP:dev-kafka:9092}
spring.kafka.consumer.auto-offset-reset=latest
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer

# Kafka Local 환경 설정 (외부 노출 포트 사용)
# db-kafka-local.properties
spring.kafka.bootstrap-servers=localhost:19092
