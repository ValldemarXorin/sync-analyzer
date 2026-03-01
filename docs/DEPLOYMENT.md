# Развёртывание и настройка платформы

## 1. Введение

Данный документ описывает способы развёртывания платформы анализа синхронных взаимодействий для различных целей:
- **Локальная разработка и тестирование** – быстрый запуск на рабочей станции
- **Демонстрационный стенд** – для презентации возможностей
- **Интеграция в CI/CD** – запуск анализа в тестовых пайплайнах

Платформа состоит из нескольких компонентов, которые могут быть развёрнуты как по отдельности, так и в составе единого стека с помощью Docker Compose.

---

## 2. Компоненты и их зависимости

| Компонент       | Зависимости                          | Порт по умолчанию |
|-----------------|--------------------------------------|-------------------|
| **Коллектор**   | Kafka, (опционально) БД для метрик   | 8081              |
| **Kafka**       | Zookeeper                            | 9092              |
| **Zookeeper**   | -                                    | 2181              |
| **TimescaleDB** | -                                    | 5432              |
| **Анализатор**  | Kafka, TimescaleDB                   | - (внутренний)    |
| **Визуализатор**| TimescaleDB                          | 8080              |

---

## 3. Развёртывание с помощью Docker Compose (рекомендуется)

### 3.1. Структура директорий

```
sync-analyzer/
├── docker-compose.yml
├── .env
├── collector/
│   ├── Dockerfile
│   └── target/collector-*.jar
├── analyzer/
│   ├── Dockerfile
│   └── target/analyzer-*.jar
├── visualizer/
│   ├── Dockerfile
│   └── target/visualizer-*.jar
└── config/
    ├── collector.yml
    ├── analyzer.yml
    └── visualizer.yml
```

### 3.2. Файл .env (переменные окружения)

```env
# Версии образов
KAFKA_VERSION=3.6.0
TIMESCALE_VERSION=2.15.0-pg14

# Пароли (изменить в production!)
POSTGRES_PASSWORD=strong_password
POSTGRES_USER=postgres
POSTGRES_DB=postgres

# Настройки Kafka
KAFKA_BROKER_ID=1
KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1

# Окружение
ENVIRONMENT=development
```

### 3.3. docker-compose.yml

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:${KAFKA_VERSION}
    container_name: zookeeper
    restart: unless-stopped
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-log:/var/lib/zookeeper/log
    networks:
      - analyzer-net

  kafka:
    image: confluentinc/cp-kafka:${KAFKA_VERSION}
    container_name: kafka
    restart: unless-stopped
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: ${KAFKA_BROKER_ID}
      KAFKA_ZOOKEEPER_CONNECT: ${KAFKA_ZOOKEEPER_CONNECT}
      KAFKA_ADVERTISED_LISTENERS: ${KAFKA_ADVERTISED_LISTENERS}
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: ${KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR}
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_LOG_RETENTION_HOURS: 24
    volumes:
      - kafka-data:/var/lib/kafka/data
    networks:
      - analyzer-net

  timescaledb:
    image: timescale/timescaledb:${TIMESCALE_VERSION}
    container_name: timescaledb
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - timescaledb-data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    networks:
      - analyzer-net

  collector:
    build: ./collector
    container_name: collector
    restart: unless-stopped
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      ENVIRONMENT: ${ENVIRONMENT}
    volumes:
      - ./config/collector.yml:/app/config/application.yml
      - collector-logs:/app/logs
    networks:
      - analyzer-net

  analyzer:
    build: ./analyzer
    container_name: analyzer
    restart: unless-stopped
    depends_on:
      - kafka
      - timescaledb
    environment:
      SPRING_PROFILES_ACTIVE: docker
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_DATASOURCE_URL: jdbc:postgresql://timescaledb:5432/${POSTGRES_DB}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./config/analyzer.yml:/app/config/application.yml
      - analyzer-logs:/app/logs
    networks:
      - analyzer-net

  visualizer:
    build: ./visualizer
    container_name: visualizer
    restart: unless-stopped
    depends_on:
      - timescaledb
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://timescaledb:5432/${POSTGRES_DB}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./config/visualizer.yml:/app/config/application.yml
      - visualizer-logs:/app/logs
    networks:
      - analyzer-net

volumes:
  zookeeper-data:
  zookeeper-log:
  kafka-data:
  timescaledb-data:
  collector-logs:
  analyzer-logs:
  visualizer-logs:

networks:
  analyzer-net:
    driver: bridge
```

### 3.4. Пример Dockerfile для компонента (коллектор)

```dockerfile
FROM eclipse-temurin:17-jre-alpine

# Создание пользователя (безопасность)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Копирование jar-файла
COPY target/collector-*.jar app.jar

# Копирование конфигурации (опционально, может быть смонтирована)
COPY config/application.yml /app/config/application.yml

# Права на директории
RUN mkdir -p /app/logs && chown -R appuser:appgroup /app

USER appuser

EXPOSE 8081

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8081/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar", "--spring.config.location=file:/app/config/application.yml"]
```

### 3.5. Инициализация базы данных (init-db.sql)

```sql
-- Создание расширения TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Таблица спанов
CREATE TABLE IF NOT EXISTS spans (
    time TIMESTAMPTZ NOT NULL,
    trace_id TEXT NOT NULL,
    span_id TEXT NOT NULL,
    parent_span_id TEXT,
    service_name TEXT NOT NULL,
    span_data JSONB NOT NULL,
    duration_ms BIGINT GENERATED ALWAYS AS ((span_data->>'durationMs')::bigint) STORED,
    PRIMARY KEY (time, trace_id, span_id)
);

SELECT create_hypertable('spans', 'time', chunk_time_interval => INTERVAL '1 day', if_not_exists => TRUE);

-- Индексы
CREATE INDEX IF NOT EXISTS idx_spans_trace_id ON spans (trace_id);
CREATE INDEX IF NOT EXISTS idx_spans_service_name ON spans (service_name);
CREATE INDEX IF NOT EXISTS idx_spans_duration ON spans (duration_ms DESC);
CREATE INDEX IF NOT EXISTS idx_spans_gin ON spans USING gin (span_data jsonb_path_ops);

-- Таблица агрегированных метрик
CREATE TABLE IF NOT EXISTS endpoint_stats (
    bucket TIMESTAMPTZ NOT NULL,
    service_name TEXT NOT NULL,
    path TEXT NOT NULL,
    method TEXT NOT NULL,
    p95_latency DOUBLE PRECISION,
    p99_latency DOUBLE PRECISION,
    error_rate DOUBLE PRECISION,
    throughput INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (bucket, service_name, path, method)
);

SELECT create_hypertable('endpoint_stats', 'bucket', chunk_time_interval => INTERVAL '1 day', if_not_exists => TRUE);

-- Таблица интегральных коэффициентов
CREATE TABLE IF NOT EXISTS integration_scores (
    time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    service_name TEXT NOT NULL,
    endpoint_path TEXT,
    score DOUBLE PRECISION NOT NULL,
    components JSONB NOT NULL,
    calculated_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (time, service_name, endpoint_path)
);

SELECT create_hypertable('integration_scores', 'time', chunk_time_interval => INTERVAL '7 days', if_not_exists => TRUE);

-- Таблица проблем
CREATE TABLE IF NOT EXISTS architectural_problems (
    id BIGSERIAL PRIMARY KEY,
    detected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    trace_id TEXT,
    service_name TEXT NOT NULL,
    endpoint_path TEXT,
    problem_type TEXT NOT NULL,
    description TEXT NOT NULL,
    recommendation TEXT NOT NULL,
    resolved BOOLEAN DEFAULT FALSE,
    resolved_at TIMESTAMPTZ,
    metadata JSONB
);

CREATE INDEX IF NOT EXISTS idx_problems_type ON architectural_problems (problem_type);
CREATE INDEX IF NOT EXISTS idx_problems_service ON architectural_problems (service_name);
CREATE INDEX IF NOT EXISTS idx_problems_resolved ON architectural_problems (resolved);

-- Политика хранения (удалять старые данные)
SELECT add_retention_policy('spans', INTERVAL '7 days', if_not_exists => TRUE);
SELECT add_retention_policy('endpoint_stats', INTERVAL '30 days', if_not_exists => TRUE);
SELECT add_retention_policy('integration_scores', INTERVAL '90 days', if_not_exists => TRUE);
```

### 3.6. Запуск стека

```bash
# Клонировать репозиторий (или создать структуру)
git clone https://github.com/yourname/sync-analyzer.git
cd sync-analyzer

# Собрать jar-файлы (если не собраны)
./mvnw clean package

# Запустить все сервисы
docker-compose up -d

# Проверить статус
docker-compose ps

# Просмотр логов
docker-compose logs -f collector
docker-compose logs -f analyzer

# Остановка
docker-compose down
```

После запуска:
- Визуализатор доступен по адресу: http://localhost:8080
- Коллектор: http://localhost:8081 (для отправки спанов)
- Kafka: localhost:9092
- TimescaleDB: localhost:5432

---

## 4. Локальный запуск для разработки (без Docker)

### 4.1. Требования
- Java 17+
- Maven 3.8+
- Apache Kafka (установленная локально или через Docker)
- PostgreSQL 14+ с расширением TimescaleDB

### 4.2. Запуск зависимостей через Docker (рекомендуется)

```bash
# Запустить только Kafka и TimescaleDB
docker-compose up -d kafka timescaledb zookeeper
```

### 4.3. Сборка и запуск компонентов

```bash
# Сборка всех модулей
mvn clean package

# Запуск коллектора
java -jar collector/target/collector-*.jar \
  --spring.config.location=file:config/collector-dev.yml

# Запуск анализатора (в отдельном терминале)
java -jar analyzer/target/analyzer-*.jar \
  --spring.config.location=file:config/analyzer-dev.yml

# Запуск визуализатора (в отдельном терминале)
java -jar visualizer/target/visualizer-*.jar \
  --spring.config.location=file:config/visualizer-dev.yml
```

### 4.4. Пример конфигурации для разработки (collector-dev.yml)

```yaml
server:
  port: 8081

collector:
  validation:
    strict: false  # в разработке можно быть мягче
  enrichment:
    environment: development
  kafka:
    topic: raw-spans
    bootstrap-servers: localhost:9092
    producer:
      acks: all
      retries: 5
      batch-size: 16384
      linger-ms: 10
      enable-idempotence: true

logging:
  level:
    com.yourproject.collector: DEBUG
```

---

## 5. Интеграция в CI/CD

### 5.1. GitLab CI пример (.gitlab-ci.yml)

```yaml
stages:
  - test
  - analyze

variables:
  DOCKER_DRIVER: overlay2
  COLLECTOR_IMAGE: $CI_REGISTRY_IMAGE/collector:$CI_COMMIT_SHA
  ANALYZER_IMAGE: $CI_REGISTRY_IMAGE/analyzer:$CI_COMMIT_SHA
  VISUALIZER_IMAGE: $CI_REGISTRY_IMAGE/visualizer:$CI_COMMIT_SHA

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

test:
  stage: test
  script:
    - mvn clean test
  artifacts:
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
      coverage:
        - target/site/jacoco/index.html

analyze:
  stage: analyze
  services:
    - docker:dind
  script:
    # Сборка образов
    - mvn clean package -DskipTests
    - docker build -t $COLLECTOR_IMAGE ./collector
    - docker build -t $ANALYZER_IMAGE ./analyzer
    - docker build -t $VISUALIZER_IMAGE ./visualizer
    
    # Запуск стека для анализа
    - docker-compose -f docker-compose.ci.yml up -d
    
    # Ожидание готовности
    - sleep 30
    
    # Запуск тестовых сценариев (с генерацией нагрузки)
    - ./run-load-tests.sh
    
    # Сбор отчёта анализатора
    - curl http://localhost:8080/api/reports/latest > report.json
    
    # Проверка интегрального коэффициента (не выше порога)
    - jq '.avg_score' report.json | awk '{if ($1 > 0.6) exit 1}'
    
    # Остановка
    - docker-compose -f docker-compose.ci.yml down
  artifacts:
    paths:
      - report.json
```

### 5.2. docker-compose.ci.yml (упрощённый для CI)

```yaml
version: '3.8'

services:
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  timescaledb:
    image: timescale/timescaledb:latest-pg14
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_USER: postgres
      POSTGRES_DB: postgres

  collector:
    image: ${COLLECTOR_IMAGE}
    environment:
      SPRING_PROFILES_ACTIVE: ci
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092

  analyzer:
    image: ${ANALYZER_IMAGE}
    depends_on:
      - kafka
      - timescaledb
    environment:
      SPRING_PROFILES_ACTIVE: ci
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_DATASOURCE_URL: jdbc:postgresql://timescaledb:5432/postgres
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password

  visualizer:
    image: ${VISUALIZER_IMAGE}
    ports:
      - "8080:8080"
    depends_on:
      - timescaledb
    environment:
      SPRING_PROFILES_ACTIVE: ci
      SPRING_DATASOURCE_URL: jdbc:postgresql://timescaledb:5432/postgres
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password
```

---

## 6. Настройка под разные окружения

### 6.1. Профили Spring Boot

| Профиль      | Назначение                                  |
|--------------|---------------------------------------------|
| `default`    | Локальная разработка (in-memory H2)         |
| `dev`        | Разработка с внешними зависимостями         |
| `test`       | Тестирование (интеграционные тесты)         |
| `ci`         | CI/CD пайплайн (упрощённая конфигурация)    |
| `demo`       | Демонстрационный стенд (с тестовыми данными)|
| `prod`       | Production (строгие настройки, безопасность)|

### 6.2. Пример application-demo.yml для демо-стенда

```yaml
server:
  port: 8080

spring:
  thymeleaf:
    cache: false  # для быстрого обновления страниц
  datasource:
    url: jdbc:postgresql://timescaledb:5432/postgres
    username: postgres
    password: password
    hikari:
      read-only: true

visualizer:
  refresh-interval: 10000  # автообновление каждые 10 сек
  default-time-range: 1h

logging:
  level:
    com.yourproject.visualizer: INFO
    org.springframework.web: INFO
```

---

## 7. Мониторинг и обслуживание

### 7.1. Проверка здоровья компонентов

```bash
# Коллектор
curl http://localhost:8081/health

# Визуализатор
curl http://localhost:8080/actuator/health

# Kafka
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --list

# TimescaleDB
docker exec timescaledb psql -U postgres -c "SELECT * FROM timescaledb_information.hypertables;"
```

### 7.2. Просмотр логов

```bash
# Все логи
docker-compose logs -f

# Только анализатор
docker-compose logs -f analyzer

# Только коллектор с уровнем DEBUG
docker-compose logs -f collector | grep DEBUG
```

### 7.3. Резервное копирование БД

```bash
# Резервное копирование
docker exec timescaledb pg_dump -U postgres -d postgres > backup.sql

# Восстановление
cat backup.sql | docker exec -i timescaledb psql -U postgres -d postgres
```

---

## 8. Устранение неполадок

### 8.1. Коллектор не принимает спаны

**Проверка:**
```bash
# Доступность Kafka
docker exec kafka kafka-topics --bootstrap-server localhost:9092 --list

# Логи коллектора
docker-compose logs collector | grep ERROR
```

**Решение:** убедиться, что Kafka доступна по адресу, указанному в `KAFKA_BOOTSTRAP_SERVERS`.

### 8.2. Анализатор не обрабатывает данные

**Проверка:**
```bash
# Лаг в Kafka
docker exec kafka kafka-consumer-groups --bootstrap-server localhost:9092 --group analyzer-group --describe

# Логи анализатора
docker-compose logs analyzer | grep ERROR
```

**Решение:** проверить подключение к БД и права на запись.

### 8.3. Визуализатор не показывает данные

**Проверка:**
```bash
# Есть ли данные в БД?
docker exec timescaledb psql -U postgres -d postgres -c "SELECT COUNT(*) FROM spans;"
```

**Решение:** убедиться, что анализатор успешно сохраняет спаны.

---

## 9. Заключение

Данный документ покрывает основные сценарии развёртывания платформы:
- Локальная разработка с Docker Compose
- Ручной запуск компонентов для разработки
- Интеграция в CI/CD пайплайн
- Настройка под различные окружения

Для большинства целей рекомендуется использовать **Docker Compose** как самый простой и воспроизводимый способ запуска всего стека.

При возникновении проблем обращайтесь к логам компонентов и проверяйте доступность зависимостей (Kafka, БД).