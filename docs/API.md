# API коллектора

## 1. Введение

API коллектора предназначено для приёма спанов (телеметрических данных о синхронных вызовах) от:
- **Java-агентов** (основной источник)
- **Сервисов на других языках** (Python, Go, Node.js и т.д.), которые не могут использовать Java-агент, но хотят отправлять данные в платформу
- **Тестовых скриптов** и инструментов нагрузочного тестирования

API спроектировано максимально простым, чтобы интеграция с любым языком программирования не вызывала затруднений.

**Базовый URL:** `http://<collector-host>:<port>/api`

**Формат данных:** JSON (UTF-8)

---

## 2. Аутентификация (опционально)

В тестовой среде аутентификация может быть отключена. Для production-подобных развёртываний предусмотрена поддержка API-ключей.

**Способ:** передача ключа в заголовке `X-API-Key`

```
X-API-Key: your-secret-api-key
```

При неверном или отсутствующем ключе возвращается `401 Unauthorized`.

---

## 3. Эндпоинты

### 3.1. Отправка спанов

**`POST /api/spans`**

Отправляет один или несколько спанов в коллектор.

#### Запрос

**Заголовки:**
- `Content-Type: application/json` (обязательно)
- `X-API-Key: <key>` (если включена аутентификация)

**Тело запроса:** массив объектов-спанов в формате JSON. Допускается как один объект, так и массив (batch-отправка).

**Максимальный размер одного запроса:** 5 МБ (настраивается).

#### Формат спана

```json
{
  "traceId": "0af7651916cd43dd8448eb211c80319c",
  "spanId": "b7ad6b7169203331",
  "parentSpanId": "b7ad6b7169203330",
  "name": "GET /api/orders/123",
  "serviceName": "order-service",
  "kind": "SERVER",
  "protocol": "REST",
  "method": "GET",
  "path": "/api/orders/123",
  "startTime": "2026-03-01T12:00:00.123Z",
  "endTime": "2026-03-01T12:00:00.456Z",
  "durationMs": 333,
  "statusCode": 200,
  "error": null,
  "targetService": null,
  "targetPath": null,
  "tags": {}
}
```

| Поле | Тип | Обязательное | Описание |
|------|-----|--------------|----------|
| `traceId` | string | ✅ | Глобальный идентификатор трассировки (32 символа hex) |
| `spanId` | string | ✅ | Идентификатор текущего спана (16 символов hex) |
| `parentSpanId` | string | ❌ | Идентификатор родительского спана (для корневого – null/отсутствует) |
| `name` | string | ✅ | Краткое описание операции (например, "GET /api/orders") |
| `serviceName` | string | ✅ | Имя микросервиса |
| `kind` | string | ✅ | Тип спана: `SERVER` (входящий) или `CLIENT` (исходящий) |
| `protocol` | string | ✅ | Протокол: `REST` или `gRPC` |
| `method` | string | ✅ | HTTP-метод (для REST) или имя gRPC-метода |
| `path` | string | ✅ | URI (для REST) или полное имя метода (для gRPC) |
| `startTime` | string | ✅ | Время начала в формате ISO 8601 |
| `endTime` | string | ✅ | Время окончания в формате ISO 8601 |
| `durationMs` | number | ✅ | Длительность в миллисекундах (может вычисляться агентом) |
| `statusCode` | number | ✅ | HTTP-статус (200, 404 и т.д.) или gRPC-код |
| `error` | string | ❌ | Сообщение об ошибке (если есть) |
| `targetService` | string | ❌ | Для CLIENT-спанов – имя вызываемого сервиса |
| `targetPath` | string | ❌ | Для CLIENT-спанов – путь вызываемого эндпоинта |
| `tags` | object | ❌ | Дополнительные метки (ключ-значение) |

#### Пример запроса с одним спаном

```json
[
  {
    "traceId": "0af7651916cd43dd8448eb211c80319c",
    "spanId": "b7ad6b7169203331",
    "parentSpanId": "b7ad6b7169203330",
    "name": "GET /api/orders/123",
    "serviceName": "order-service",
    "kind": "SERVER",
    "protocol": "REST",
    "method": "GET",
    "path": "/api/orders/123",
    "startTime": "2026-03-01T12:00:00.123Z",
    "endTime": "2026-03-01T12:00:00.456Z",
    "durationMs": 333,
    "statusCode": 200,
    "error": null,
    "tags": {
      "userId": "user-456",
      "http.request_size": 512
    }
  }
]
```

#### Пример запроса с несколькими спанами (batch)

```json
[
  {
    "traceId": "0af7651916cd43dd8448eb211c80319c",
    "spanId": "b7ad6b7169203331",
    "parentSpanId": "b7ad6b7169203330",
    "name": "GET /api/orders/123",
    "serviceName": "order-service",
    "kind": "SERVER",
    "protocol": "REST",
    "method": "GET",
    "path": "/api/orders/123",
    "startTime": "2026-03-01T12:00:00.123Z",
    "endTime": "2026-03-01T12:00:00.456Z",
    "durationMs": 333,
    "statusCode": 200,
    "error": null,
    "tags": {}
  },
  {
    "traceId": "0af7651916cd43dd8448eb211c80319c",
    "spanId": "d8ec7b7169203442",
    "parentSpanId": "b7ad6b7169203331",
    "name": "GET /api/payments/456",
    "serviceName": "payment-service",
    "kind": "SERVER",
    "protocol": "REST",
    "method": "GET",
    "path": "/api/payments/456",
    "startTime": "2026-03-01T12:00:00.200Z",
    "endTime": "2026-03-01T12:00:00.300Z",
    "durationMs": 100,
    "statusCode": 200,
    "error": null,
    "tags": {}
  }
]
```

#### Ответы

**`202 Accepted`**
```json
{
  "status": "accepted",
  "received": 2,
  "message": "Spans accepted for processing"
}
```

**`400 Bad Request`** (неверный формат, отсутствуют обязательные поля)
```json
{
  "status": "error",
  "code": "VALIDATION_FAILED",
  "message": "Invalid span data: missing required field 'traceId' at index 0"
}
```

**`401 Unauthorized`** (неверный или отсутствующий API-ключ)
```json
{
  "status": "error",
  "code": "UNAUTHORIZED",
  "message": "Invalid or missing API key"
}
```

**`413 Payload Too Large`** (превышен максимальный размер запроса)
```json
{
  "status": "error",
  "code": "PAYLOAD_TOO_LARGE",
  "message": "Request size exceeds 5MB limit"
}
```

**`429 Too Many Requests`** (превышен лимит скорости)
```json
{
  "status": "error",
  "code": "RATE_LIMITED",
  "message": "Too many requests, please slow down"
}
```

**`500 Internal Server Error`** (внутренняя ошибка коллектора)
```json
{
  "status": "error",
  "code": "INTERNAL_ERROR",
  "message": "Internal server error"
}
```

---

### 3.2. Проверка здоровья (health check)

**`GET /health`**

Используется для проверки доступности коллектора (например, в Docker Compose или Kubernetes).

#### Ответ

**`200 OK`**
```json
{
  "status": "UP",
  "timestamp": "2026-03-01T12:00:00Z",
  "components": {
    "kafka": {
      "status": "UP"
    }
  }
}
```

**`503 Service Unavailable`** (если коллектор не может подключиться к Kafka)
```json
{
  "status": "DOWN",
  "timestamp": "2026-03-01T12:00:00Z",
  "components": {
    "kafka": {
      "status": "DOWN",
      "reason": "Connection refused"
    }
  }
}
```

---

### 3.3. Метрики (опционально)

**`GET /metrics`**

Возвращает метрики коллектора в формате Prometheus (если включён Actuator).

```
# HELP collector_spans_received_total Total number of spans received
# TYPE collector_spans_received_total counter
collector_spans_received_total{service="order-service"} 1234.0
collector_spans_received_total{service="payment-service"} 5678.0

# HELP collector_spans_invalid_total Total number of invalid spans
# TYPE collector_spans_invalid_total counter
collector_spans_invalid_total 42.0

# HELP collector_request_duration_seconds Request duration histogram
# TYPE collector_request_duration_seconds histogram
collector_request_duration_seconds_bucket{le="0.1"} 100.0
collector_request_duration_seconds_bucket{le="0.5"} 300.0
collector_request_duration_seconds_bucket{le="1.0"} 350.0
collector_request_duration_seconds_bucket{le="+Inf"} 360.0
collector_request_duration_seconds_sum 45.6
collector_request_duration_seconds_count 360.0
```

---

## 4. Интеграция с сервисами на других языках

### 4.1. Python (пример с requests)

```python
import requests
import time
import uuid

def send_span(collector_url, span_data):
    headers = {
        'Content-Type': 'application/json',
        'X-API-Key': 'your-api-key'  # если требуется
    }
    response = requests.post(
        f"{collector_url}/api/spans",
        json=[span_data],  # отправляем как массив
        headers=headers,
        timeout=2
    )
    if response.status_code == 202:
        print("Span accepted")
    else:
        print(f"Error: {response.status_code} - {response.text}")

# Пример использования
span = {
    "traceId": uuid.uuid4().hex,
    "spanId": uuid.uuid4().hex[:16],
    "name": "GET /api/users/123",
    "serviceName": "user-service-python",
    "kind": "SERVER",
    "protocol": "REST",
    "method": "GET",
    "path": "/api/users/123",
    "startTime": datetime.utcnow().isoformat() + "Z",
    "endTime": datetime.utcnow().isoformat() + "Z",
    "durationMs": 150,
    "statusCode": 200,
    "tags": {"language": "python"}
}

send_span("http://collector:8081", span)
```

### 4.2. Node.js (пример с axios)

```javascript
const axios = require('axios');
const { v4: uuidv4 } = require('uuid');

async function sendSpan(collectorUrl, spanData) {
    try {
        const response = await axios.post(
            `${collectorUrl}/api/spans`,
            [spanData],  // массив
            {
                headers: {
                    'Content-Type': 'application/json',
                    'X-API-Key': 'your-api-key'
                },
                timeout: 2000
            }
        );
        if (response.status === 202) {
            console.log('Span accepted');
        }
    } catch (error) {
        console.error('Error sending span:', error.message);
    }
}

// Пример
const span = {
    traceId: uuidv4().replace(/-/g, ''),
    spanId: uuidv4().replace(/-/g, '').substring(0, 16),
    name: 'GET /api/users/123',
    serviceName: 'user-service-node',
    kind: 'SERVER',
    protocol: 'REST',
    method: 'GET',
    path: '/api/users/123',
    startTime: new Date().toISOString(),
    endTime: new Date().toISOString(),
    durationMs: 150,
    statusCode: 200,
    tags: { language: 'nodejs' }
};

sendSpan('http://collector:8081', span);
```

### 4.3. Go (пример с net/http)

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type Span struct {
    TraceId     string            `json:"traceId"`
    SpanId      string            `json:"spanId"`
    Name        string            `json:"name"`
    ServiceName string            `json:"serviceName"`
    Kind        string            `json:"kind"`
    Protocol    string            `json:"protocol"`
    Method      string            `json:"method"`
    Path        string            `json:"path"`
    StartTime   string            `json:"startTime"`
    EndTime     string            `json:"endTime"`
    DurationMs  int64             `json:"durationMs"`
    StatusCode  int               `json:"statusCode"`
    Tags        map[string]string `json:"tags"`
}

func sendSpan(collectorURL string, span Span) error {
    spans := []Span{span}
    data, err := json.Marshal(spans)
    if err != nil {
        return err
    }

    req, err := http.NewRequest("POST", collectorURL+"/api/spans", bytes.NewBuffer(data))
    if err != nil {
        return err
    }
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-API-Key", "your-api-key")

    client := &http.Client{Timeout: 2 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusAccepted {
        fmt.Println("Span accepted")
        return nil
    }
    return fmt.Errorf("unexpected status: %s", resp.Status)
}

func main() {
    span := Span{
        TraceId:     "0af7651916cd43dd8448eb211c80319c",
        SpanId:      "b7ad6b7169203331",
        Name:        "GET /api/users/123",
        ServiceName: "user-service-go",
        Kind:        "SERVER",
        Protocol:    "REST",
        Method:      "GET",
        Path:        "/api/users/123",
        StartTime:   time.Now().UTC().Format(time.RFC3339),
        EndTime:     time.Now().UTC().Format(time.RFC3339),
        DurationMs:  150,
        StatusCode:  200,
        Tags:        map[string]string{"language": "go"},
    }

    sendSpan("http://collector:8081", span)
}
```

---

## 5. Ограничения и рекомендации

### 5.1. Размер батча
- Рекомендуемый размер батча: 10–100 спанов
- Максимальный размер запроса: 5 МБ (настраивается)
- При превышении лимита вернётся ошибка 413

### 5.2. Частота отправки
- Для агентов рекомендуется отправлять спаны батчами раз в 1–5 секунд или при накоплении N спанов (например, 50)
- Для сервисов с низкой нагрузкой можно отправлять сразу по одному спану

### 5.3. Таймауты
- Клиентам следует устанавливать таймаут на отправку (рекомендуется 2–5 секунд)
- При таймауте рекомендуется сохранять спаны локально и повторить позже

### 5.4. Обработка ошибок
- **400 Bad Request** – исправить формат спана и повторить
- **429 Too Many Requests** – замедлить отправку (exponential backoff)
- **5xx** – повторить с задержкой (коллектор временно недоступен)

---

## 6. Версионирование API

В текущей версии API не версионируется (v1 подразумевается по умолчанию). При необходимости добавления обратно несовместимых изменений будет введён префикс `/v2/`.

---

## 7. Заключение

API коллектора спроектировано максимально простым и понятным, чтобы обеспечить лёгкую интеграцию с сервисами на любых языках программирования. Формат спана соответствует общей модели данных платформы и содержит всю необходимую информацию для последующего анализа.

При возникновении вопросов или проблем обращайтесь к документации или создавайте issue в репозитории проекта.