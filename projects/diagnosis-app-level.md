# Log Doctor: 애플리케이션 레벨 진단 — 상세 구현 가이드

> 고객사 애플리케이션이 **Python, .NET(C#), Spring Boot(Java)** 중 하나로 개발되었다고 가정합니다.
> 세 스택 모두 **Application Insights → LAW**로 로그가 수집되며, LAW에 저장되는 테이블과 필드는 동일합니다.
> 이 문서는 "스택별로 어떤 로그가 어떻게 나오는지"부터 "진단 로직을 어떻게 짜는지"까지 **빈틈없이** 다룹니다.

---

## 1. 전제: Application Insights가 하는 일

```
고객사 앱 (Python/Spring/.NET)
        │
        │  Application Insights SDK / Agent
        │  (뭘 설치하느냐에 따라 방법만 다르고, 결과는 같음)
        │
        ▼
┌─────────────────────────────────────────────────┐
│            Application Insights 리소스            │
│                                                   │
│  자동 수집:                                        │
│  ├── HTTP 요청 (들어오는 것) → AppRequests         │
│  ├── 외부 호출 (나가는 것)   → AppDependencies     │
│  └── 미처리 예외            → AppExceptions        │
│                                                   │
│  개발자가 직접 남긴 로그:                            │
│  └── logger.info/warn/error → AppTraces           │
│                                                   │
│  공통 필드:                                         │
│  ├── OperationId     ← 하나의 요청을 추적하는 ID    │
│  ├── AppRoleName     ← 어떤 서비스인지 (MSA 구분)   │
│  └── CustomDimensions ← 개발자가 추가한 커스텀 속성  │
└────────────────────┬────────────────────────────┘
                     │  Diagnostic Settings
                     ▼
             Log Analytics Workspace (LAW)
             → Log Doctor Agent가 여기를 분석
```

> **핵심**: 스택이 Python이든 .NET이든 Spring이든, LAW에 들어가는 **테이블 이름과 필드 구조는 동일**합니다.
> 차이점은 "어떤 값이 들어오느냐"입니다.

---

## 2. 스택별 Application Insights 연동 방식

### 2-1. Python (FastAPI / Django / Flask)

```python
# ── 설치 ──
# pip install azure-monitor-opentelemetry

# ── 연동 코드 (FastAPI 예시) ──
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
import logging

# Application Insights 연결
configure_azure_monitor(
    connection_string="InstrumentationKey=xxx;IngestionEndpoint=https://..."
)

app = FastAPI()
FastAPIInstrumentor.instrument_app(app)

# ── 개발자가 남기는 로그 ──
logger = logging.getLogger(__name__)

@app.get("/api/orders/{order_id}")
async def get_order(order_id: str):
    logger.info(f"주문 조회 시작: {order_id}")          # → AppTraces (SeverityLevel=1)
    logger.debug(f"DB 쿼리 파라미터: {order_id}")       # → AppTraces (SeverityLevel=0)
    try:
        order = await db.get_order(order_id)            # → AppDependencies (자동)
        logger.info(f"주문 조회 완료: {order.status}")   # → AppTraces (SeverityLevel=1)
        return order                                     # → AppRequests (자동, ResultCode=200)
    except Exception as e:
        logger.error(f"주문 조회 실패: {e}")             # → AppTraces (SeverityLevel=3)
        raise                                            # → AppExceptions (자동)
```

**Python logging → Application Insights SeverityLevel 매핑:**

| Python `logging` 레벨 | AI `SeverityLevel` 값 | AI 표시 이름 |
|:---|:---:|:---|
| `logging.DEBUG` | 0 | Verbose |
| `logging.INFO` | 1 | Information |
| `logging.WARNING` | 2 | Warning |
| `logging.ERROR` | 3 | Error |
| `logging.CRITICAL` | 4 | Critical |

**자동 수집되는 것:**

| 자동 수집 항목 | LAW 테이블 | 어떤 라이브러리가 수집 |
|:---|:---|:---|
| FastAPI 라우터 요청/응답 | `AppRequests` | `opentelemetry-instrumentation-fastapi` |
| `httpx`/`aiohttp` 외부 호출 | `AppDependencies` | `opentelemetry-instrumentation-httpx` |
| DB 쿼리 (SQLAlchemy, psycopg2) | `AppDependencies` | `opentelemetry-instrumentation-sqlalchemy` |
| 미처리 Exception | `AppExceptions` | OpenTelemetry 자동 |

---

### 2-2. .NET (ASP.NET Core / C#)

```csharp
// ── NuGet 패키지 설치 ──
// dotnet add package Microsoft.ApplicationInsights.AspNetCore

// ── Program.cs 연동 ──
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddApplicationInsightsTelemetry(); // ← 이 한 줄로 연동

var app = builder.Build();

// ── Controller에서 개발자가 남기는 로그 ──
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly ILogger<OrdersController> _logger;
    private readonly HttpClient _httpClient;

    public OrdersController(ILogger<OrdersController> logger, HttpClient httpClient)
    {
        _logger = logger;
        _httpClient = httpClient;
    }

    [HttpGet("{orderId}")]
    public async Task<IActionResult> GetOrder(string orderId)
    {
        _logger.LogInformation("주문 조회 시작: {OrderId}", orderId);
        // → AppTraces (SeverityLevel=1)

        _logger.LogDebug("캐시 확인 중: {OrderId}", orderId);
        // → AppTraces (SeverityLevel=0)

        try
        {
            // HttpClient 호출 → AppDependencies (자동, Type="HTTP")
            var response = await _httpClient.GetAsync($"https://payment-api/orders/{orderId}");

            // EF Core DB 쿼리 → AppDependencies (자동, Type="SQL")
            var order = await _dbContext.Orders.FindAsync(orderId);

            _logger.LogInformation("주문 조회 완료: {Status}", order.Status);
            return Ok(order);
            // → AppRequests (자동, ResultCode=200)
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "주문 조회 실패: {OrderId}", orderId);
            // → AppTraces (SeverityLevel=3) + AppExceptions (자동)
            throw;
        }
    }
}
```

**.NET ILogger → Application Insights SeverityLevel 매핑:**

| .NET `LogLevel` | AI `SeverityLevel` 값 | AI 표시 이름 | 비고 |
|:---|:---:|:---|:---|
| `LogLevel.Trace` | 0 | Verbose | Debug와 합쳐짐 |
| `LogLevel.Debug` | 0 | Verbose | Trace와 합쳐짐 |
| `LogLevel.Information` | 1 | Information | |
| `LogLevel.Warning` | 2 | Warning | |
| `LogLevel.Error` | 3 | Error | |
| `LogLevel.Critical` | 4 | Critical | |

> ⚠️ **.NET 특이점**: `Trace`와 `Debug` 모두 SeverityLevel=0으로 매핑됩니다.
> Log Doctor는 둘 다 "불필요한 프로덕션 로그"로 판단합니다.

**자동 수집되는 것:**

| 자동 수집 항목 | LAW 테이블 | 비고 |
|:---|:---|:---|
| ASP.NET Core 요청/응답 | `AppRequests` | Middleware 자동 삽입 |
| `HttpClient` 외부 호출 | `AppDependencies` | `Type="HTTP"` |
| EF Core / SQL 쿼리 | `AppDependencies` | `Type="SQL"`, `Data`에 쿼리문 포함 |
| Redis 호출 | `AppDependencies` | `Type="Redis"` |
| 미처리 Exception | `AppExceptions` | `ExceptionType`에 C# 예외 클래스명 |

---

### 2-3. Spring Boot (Java)

```java
// ── 의존성 (pom.xml) ──
// <dependency>
//   <groupId>com.microsoft.azure</groupId>
//   <artifactId>applicationinsights-runtime-attach</artifactId>
//   <version>3.6.2</version>
// </dependency>

// ── Application 진입점 ──
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        ApplicationInsights.attach();  // ← 이 한 줄로 연동
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// ── Controller에서 개발자가 남기는 로그 ──
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private static final Logger logger = LoggerFactory.getLogger(OrderController.class);

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @GetMapping("/{orderId}")
    public ResponseEntity<Order> getOrder(@PathVariable String orderId) {

        logger.info("주문 조회 시작: {}", orderId);
        // → AppTraces (SeverityLevel=1)

        logger.debug("DB 쿼리 파라미터: {}", orderId);
        // → AppTraces (SeverityLevel=0)

        try {
            // RestTemplate 호출 → AppDependencies (자동, Type="HTTP")
            Payment payment = restTemplate.getForObject(
                "https://payment-api/orders/" + orderId, Payment.class
            );

            // JDBC 쿼리 → AppDependencies (자동, Type="SQL")
            Order order = jdbcTemplate.queryForObject(
                "SELECT * FROM orders WHERE id = ?", orderMapper, orderId
            );

            logger.info("주문 조회 완료: {}", order.getStatus());
            return ResponseEntity.ok(order);
            // → AppRequests (자동, ResultCode=200)

        } catch (Exception e) {
            logger.error("주문 조회 실패: {}", orderId, e);
            // → AppTraces (SeverityLevel=3) + AppExceptions (자동)
            throw e;
        }
    }
}
```

**Spring Boot SLF4J/Logback → Application Insights SeverityLevel 매핑:**

| SLF4J/Logback 레벨 | AI `SeverityLevel` 값 | AI 표시 이름 | 비고 |
|:---|:---:|:---|:---|
| `TRACE` | 0 | Verbose | |
| `DEBUG` | 0 | Verbose | TRACE와 합쳐짐 |
| `INFO` | 1 | Information | |
| `WARN` | 2 | Warning | |
| `ERROR` | 3 | Error | Java에는 FATAL 없음 (ERROR로 통합) |

**자동 수집되는 것:**

| 자동 수집 항목 | LAW 테이블 | 비고 |
|:---|:---|:---|
| Spring MVC 컨트롤러 요청/응답 | `AppRequests` | Java Agent 자동 |
| `RestTemplate` / `WebClient` 호출 | `AppDependencies` | `Type="HTTP"` |
| JDBC / JPA 쿼리 | `AppDependencies` | `Type="SQL"` |
| Kafka Producer/Consumer | `AppDependencies` | `Type="Kafka"` |
| Redis (Jedis/Lettuce) | `AppDependencies` | `Type="Redis"` |
| 미처리 Exception | `AppExceptions` | `ExceptionType`에 Java 예외 클래스명 |

---

## 3. LAW에 저장되는 4개 테이블 상세 스키마

> 스택이 뭐든 LAW에는 동일한 구조로 저장됩니다.

### 3-1. AppTraces (개발자 로그)

```
개발자가 logger.info(), logger.error() 등으로 남긴 모든 로그.
```

| 필드 | 타입 | 설명 | 진단에서의 역할 |
|:---|:---|:---|:---|
| `TimeGenerated` | datetime | 로그 발생 시각 | 시간 범위 필터 |
| **`SeverityLevel`** | int | 0=Verbose, 1=Info, 2=Warning, 3=Error, 4=Critical | **핵심 — 심각도 분류** |
| **`Message`** | string | 로그 메시지 본문 | 패턴 분석, 반복 탐지 |
| `OperationId` | string | 요청 추적 ID (같은 요청의 모든 로그가 공유) | 요청 단위 그룹핑 |
| `OperationName` | string | 어떤 API 엔드포인트에서 발생했는지 | 엔드포인트별 분석 |
| `AppRoleName` | string | 서비스명 (MSA에서 어떤 서비스인지) | 서비스별 분석 |
| `ClientIP` | string | 클라이언트 IP | 보안 연계 |
| `CustomDimensions` | dynamic | 개발자가 추가한 커스텀 속성 (JSON) | 스택별 특화 분석 |

**KQL 예시 — 스택 구분 없이 동작:**

```kql
// 최근 1시간 동안 ERROR 이상 로그
AppTraces
| where TimeGenerated > ago(1h)
| where SeverityLevel >= 3
| summarize ErrorCount=count() by AppRoleName, OperationName
| order by ErrorCount desc
```

```kql
// 프로덕션에 남아있는 Debug/Trace 로그 (= 비용 낭비)
AppTraces
| where TimeGenerated > ago(1h)
| where SeverityLevel <= 0
| summarize DebugCount=count() by AppRoleName
| order by DebugCount desc
```

```kql
// 동일 메시지 반복 로그 탐지 (= 노이즈 후보)
AppTraces
| where TimeGenerated > ago(1h)
| summarize RepeatCount=count() by Message=substring(Message, 0, 100)
| where RepeatCount > 50
| order by RepeatCount desc
```

---

### 3-2. AppExceptions (에러 스택 트레이스)

```
try-catch에서 잡힌 예외 또는 미처리 예외가 자동 수집됨.
```

| 필드 | 타입 | 설명 | 진단에서의 역할 |
|:---|:---|:---|:---|
| `TimeGenerated` | datetime | 예외 발생 시각 | 시간 범위 필터 |
| `SeverityLevel` | int | 보통 3(Error) 또는 4(Critical) | 심각도 분류 |
| **`ExceptionType`** | string | 예외 클래스명 | **스택별 분석의 핵심** |
| **`OuterMessage`** | string | 예외 메시지 | 에러 내용 파악 |
| `ProblemId` | string | 유사 예외를 묶는 해시 코드 | 에러 그룹핑 |
| `Details` | string | 전체 스택 트레이스 (JSON) | 근본 원인 분석 |
| `OperationId` | string | 요청 추적 ID | 어떤 요청에서 발생했는지 |
| `AppRoleName` | string | 서비스명 | 서비스별 에러 분석 |

**스택별 ExceptionType 예시:**

| 스택 | ExceptionType 예시 |
|:---|:---|
| **Python** | `ValueError`, `ConnectionError`, `sqlalchemy.exc.OperationalError` |
| **.NET** | `System.NullReferenceException`, `System.Net.Http.HttpRequestException` |
| **Spring** | `java.lang.NullPointerException`, `org.springframework.web.client.RestClientException` |

**KQL 예시:**

```kql
// 예외 유형별 빈도 - Top 10
AppExceptions
| where TimeGenerated > ago(1h)
| summarize Count=count() by ExceptionType, OuterMessage=substring(OuterMessage, 0, 100)
| order by Count desc
| take 10
```

```kql
// ProblemId로 묶은 에러 그룹 (같은 코드 경로의 에러)
AppExceptions
| where TimeGenerated > ago(1h)
| summarize Count=count(), FirstSeen=min(TimeGenerated), LastSeen=max(TimeGenerated)
    by ProblemId, ExceptionType
| order by Count desc
```

---

### 3-3. AppRequests (들어오는 HTTP 요청)

```
고객사 앱이 받은 모든 HTTP 요청. 자동 수집.
```

| 필드 | 타입 | 설명 | 진단에서의 역할 |
|:---|:---|:---|:---|
| `TimeGenerated` | datetime | 요청 수신 시각 | 시간 범위 필터 |
| `Name` | string | `"GET /api/orders/{id}"` 같은 요청 이름 | 엔드포인트 분석 |
| `Url` | string | 전체 URL | 상세 분석 |
| **`ResultCode`** | string | HTTP 상태 코드 (`"200"`, `"404"`, `"500"`) | **심각도 판단 핵심** |
| **`Success`** | bool | 요청 성공 여부 | 에러율 계산 |
| **`DurationMs`** | real | 응답 시간 (밀리초) | 느린 요청 탐지 |
| `OperationId` | string | 요청 고유 ID | AppTraces/AppDependencies 연결 |
| `AppRoleName` | string | 서비스명 | 서비스별 분석 |

**KQL 예시:**

```kql
// 엔드포인트별 에러율 + 평균 응답시간
AppRequests
| where TimeGenerated > ago(1h)
| summarize
    TotalRequests=count(),
    ErrorCount=countif(toint(ResultCode) >= 500),
    ErrorRate=round(countif(toint(ResultCode) >= 500) * 100.0 / count(), 2),
    AvgDuration=round(avg(DurationMs), 0),
    P95Duration=round(percentile(DurationMs, 95), 0)
    by Name
| order by ErrorRate desc
```

```kql
// 5초 이상 느린 요청 (성능 이슈)
AppRequests
| where TimeGenerated > ago(1h)
| where DurationMs > 5000
| project TimeGenerated, Name, DurationMs, ResultCode, AppRoleName
| order by DurationMs desc
```

---

### 3-4. AppDependencies (나가는 외부 호출)

```
앱에서 외부 서비스(DB, API, Redis 등)를 호출한 기록. 자동 수집.
```

| 필드 | 타입 | 설명 | 진단에서의 역할 |
|:---|:---|:---|:---|
| `TimeGenerated` | datetime | 호출 시각 | 시간 범위 필터 |
| `Name` | string | 호출 이름 (DB 테이블명, API 경로) | 의존성별 분석 |
| **`Type`** | string | `"HTTP"`, `"SQL"`, `"Redis"`, `"Kafka"` 등 | **의존성 유형 분류** |
| **`Target`** | string | 대상 서버 (DB 서버명, API 도메인) | 어디를 호출했는지 |
| `Data` | string | 상세 정보 (SQL 쿼리문, URL 등) | 근본 원인 분석 |
| **`DurationMs`** | real | 호출 시간 (밀리초) | 느린 의존성 탐지 |
| **`ResultCode`** | string | 결과 코드 (HTTP 상태, SQL 에러코드) | 실패 탐지 |
| **`Success`** | bool | 호출 성공 여부 | 실패율 계산 |
| `OperationId` | string | 요청 추적 ID | 어떤 요청에서 호출했는지 |
| `AppRoleName` | string | 서비스명 | 서비스별 분석 |

**스택별 의존성 타입 차이:**

| 의존성 | Python 앱 | .NET 앱 | Spring 앱 |
|:---|:---|:---|:---|
| DB 호출 | `Type="SQL"` (SQLAlchemy) | `Type="SQL"` (EF Core) | `Type="SQL"` (JPA/JDBC) |
| HTTP 호출 | `Type="HTTP"` (httpx/aiohttp) | `Type="HTTP"` (HttpClient) | `Type="HTTP"` (RestTemplate/WebClient) |
| Redis | `Type="Redis"` (redis-py) | `Type="Redis"` (StackExchange.Redis) | `Type="Redis"` (Jedis/Lettuce) |
| 메시지 큐 | `Type="Azure Service Bus"` | `Type="Azure Service Bus"` | `Type="Kafka"` |

**KQL 예시:**

```kql
// 의존성 유형별 실패율 + 평균 응답시간
AppDependencies
| where TimeGenerated > ago(1h)
| summarize
    TotalCalls=count(),
    FailCount=countif(Success == false),
    FailRate=round(countif(Success == false) * 100.0 / count(), 2),
    AvgDuration=round(avg(DurationMs), 0)
    by Type, Target
| order by FailRate desc
```

```kql
// 3초 이상 걸리는 느린 DB 쿼리
AppDependencies
| where TimeGenerated > ago(1h)
| where Type == "SQL" and DurationMs > 3000
| project TimeGenerated, Name, Target, DurationMs, Data
| order by DurationMs desc
```

---

## 4. 진단 로직 상세 설계

### 4-1. 전체 흐름

```
진단 파이프라인:

  ① Collector (수집)
     │  KQL로 4개 테이블에서 최근 1시간 로그 수집
     │  테이블이 없으면 건너뜀 (App Insights 미사용 → 에러 아님)
     ▼
  ② Normalizer (정규화)
     │  테이블마다 다른 심각도 표현을 통일
     │  SeverityLevel 0~4 → TRACE/DEBUG/INFO/WARNING/ERROR/CRITICAL
     │  ResultCode 5xx → ERROR, 4xx → WARNING
     │  Success false → ERROR
     ▼
  ③ Classifier (분류)
     │  정규화된 로그에 "중요도" 점수 부여
     │  점수 = 레이어(1) + 심각도(0~5) + 목적(0~1)
     │  → critical / high / medium / low / noise
     ▼
  ④ Aggregator (집계)
     │  전체 분포 계산 (severity별, criticality별)
     │  엔진별 힌트 생성 (retain/filter/prevent)
     │  인사이트 문장 생성
     ▼
  ⑤ Result (결과)
        Provider에 POST → Teams 대시보드 표시
```

### 4-2. Normalizer 변환 로직 (의사코드)

```
함수 normalize(raw_log):

  테이블 = raw_log._source_table

  ── AppTraces / AppExceptions ──
  if 테이블 in ["AppTraces", "AppExceptions"]:
    severity_value = raw_log.SeverityLevel  (숫자: 0, 1, 2, 3, 4)

    매핑:
      0 → "TRACE"    ← Python=DEBUG, .NET=Trace/Debug, Spring=TRACE/DEBUG
      1 → "INFO"     ← 모든 스택의 INFO
      2 → "WARNING"  ← 모든 스택의 WARNING/WARN
      3 → "ERROR"    ← 모든 스택의 ERROR
      4 → "CRITICAL" ← Python=CRITICAL, .NET=Critical (Spring에는 없음)

    if AppExceptions이고 severity_value가 null:
      severity = "ERROR"  (예외는 기본 ERROR)

  ── AppRequests ──
  if 테이블 == "AppRequests":
    result_code = raw_log.ResultCode (문자열: "200", "404", "500")

    매핑:
      "2xx" → "INFO"     ← 정상 응답
      "3xx" → "INFO"     ← 리다이렉트 (정상)
      "4xx" → "WARNING"  ← 클라이언트 에러 (잘못된 요청)
      "5xx" → "ERROR"    ← 서버 에러 (우리 문제)

    추가 판단:
      DurationMs > 5000 이고 severity가 INFO일 때 → "WARNING" 으로 상향
      (느린 요청은 정상이어도 주의 필요)

  ── AppDependencies ──
  if 테이블 == "AppDependencies":
    success = raw_log.Success (boolean)

    매핑:
      true  → "INFO"    ← 정상 호출
      false → "ERROR"   ← 외부 호출 실패

    추가 판단:
      DurationMs > 3000 이고 severity가 INFO일 때 → "WARNING" 으로 상향
      (느린 외부 호출은 주의 필요)

  반환:
    ld_timestamp     = raw_log.TimeGenerated
    ld_source_table  = 테이블
    ld_layer         = "application"
    ld_severity      = 위에서 결정한 심각도
    ld_message       = (테이블에 따라 Message, OuterMessage, Name 추출)
    ld_context       = {AppRoleName, OperationId, ...}
    raw              = 원본 보존
```

### 4-3. Classifier 점수 계산 로직 (의사코드)

```
함수 classify(normalized_log):

  ── 1. 목적(Purpose) 결정 ──
  if ld_severity in ["DEBUG", "TRACE"]:
    purpose = "diagnostic"     (점수 0)
  else:
    purpose = "operational"    (점수 1)

  ── 2. 점수 계산 ──
  layer_score    = 1  (애플리케이션 레이어 고정)
  severity_score = {TRACE:0, DEBUG:1, INFO:2, WARNING:3, ERROR:4, CRITICAL:5}
  purpose_score  = {diagnostic:0, operational:1}

  total = layer_score + severity_score + purpose_score

  ── 3. 중요도 등급 ──
  if total >= 6:   criticality = "high"     ← ERROR+operational(6), CRITICAL(6~7)
  elif total >= 4: criticality = "medium"   ← WARNING(5), INFO(4)
  elif total >= 2: criticality = "low"      ← DEBUG+operational(3), INFO+diagnostic(3)
  else:            criticality = "noise"    ← TRACE(1), DEBUG+diagnostic(2)

  ── 4. 엔진 힌트 ──
  classification = {
    retain_class:       "A" if high, "B" if medium, "C" if low/noise
    retain_days_hot:    14 if high/medium, 7 if low/noise
    retain_days_archive: 365 if high, 0 otherwise
    filterable:         true if low/noise
    prevent_relevant:   true if severity in [DEBUG, TRACE]
  }

  반환: normalized_log + purpose + criticality + classification
```

### 4-4. 점수 계산 전체 매핑표

| 테이블 | 원본 값 | ld_severity | purpose | 총점 | 중요도 | Retain | Filter |
|:---|:---|:---|:---|:---:|:---|:---:|:---:|
| AppTraces | SeverityLevel=4 | CRITICAL | operational | **7** | high | A | ❌ |
| AppTraces | SeverityLevel=3 | ERROR | operational | **6** | high | A | ❌ |
| AppExceptions | (기본) | ERROR | operational | **6** | high | A | ❌ |
| AppTraces | SeverityLevel=2 | WARNING | operational | **5** | medium | B | ❌ |
| AppRequests | ResultCode=500 | ERROR | operational | **6** | high | A | ❌ |
| AppRequests | ResultCode=404 | WARNING | operational | **5** | medium | B | ❌ |
| AppRequests | ResultCode=200 | INFO | operational | **4** | medium | B | ❌ |
| AppDependencies | Success=false | ERROR | operational | **6** | high | A | ❌ |
| AppDependencies | Success=true | INFO | operational | **4** | medium | B | ❌ |
| AppTraces | SeverityLevel=1 | INFO | operational | **4** | medium | B | ❌ |
| AppTraces | SeverityLevel=1 | DEBUG | diagnostic | **2** | low | C | ✅ |
| AppTraces | SeverityLevel=0 | TRACE | diagnostic | **1** | noise | C | ✅ |

---

## 5. Aggregator — 집계 및 인사이트 생성

```
함수 aggregate(classified_logs):

  ── 분포 계산 ──
  by_severity = count by ld_severity
  by_criticality = count by ld_criticality
  by_table = count by ld_source_table
  by_service = count by ld_context.AppRoleName

  ── 에러율 계산 ──
  total = len(classified_logs)
  error_count = count where ld_severity in [ERROR, CRITICAL]
  error_rate = error_count / total * 100

  ── 엔진 힌트 ──
  retain_a = count where retain_class == "A"
  retain_b = count where retain_class == "B"
  retain_c = count where retain_class == "C"
  noise_count = count where filterable == true
  noise_rate = noise_count / total * 100
  debug_count = count where prevent_relevant == true

  ── 인사이트 자동 생성 ──
  insights = []
  if noise_rate > 20%:
    insights.add("⚠️ 노이즈 로그가 {noise_rate}% — 필터링으로 비용 절감 가능")
  if debug_count > 0:
    insights.add("⚠️ 프로덕션에 Debug/Trace 로그 {debug_count}건 — 레벨 상향 권고")
  if error_rate > 5%:
    insights.add("🔴 에러율 {error_rate}% — 즉시 확인 필요")

  ── 느린 요청/의존성 Top 5 ──
  slow_requests = AppRequests에서 DurationMs 기준 Top 5
  slow_dependencies = AppDependencies에서 DurationMs 기준 Top 5

  반환: {summary, distribution, engine_hints, insights, slow_requests, slow_dependencies}
```

---

## 6. 진단 결과 예시 (풀 JSON)

```json
{
  "tenant_id": "tenant-abc",
  "agent_id": "agent-001",
  "diagnosed_at": "2026-02-26T00:00:00+09:00",
  "diagnosis_type": "manual",

  "summary": {
    "total_logs_analyzed": 12500,
    "tables_scanned": ["AppTraces", "AppExceptions", "AppRequests", "AppDependencies"],
    "time_range_hours": 1,
    "services_found": ["order-service", "payment-service", "user-service"]
  },

  "distribution": {
    "by_table": {
      "AppTraces": 8200, "AppRequests": 2100,
      "AppDependencies": 1900, "AppExceptions": 300
    },
    "by_severity": {
      "CRITICAL": 15, "ERROR": 450, "WARNING": 680,
      "INFO": 7155, "DEBUG": 3200, "TRACE": 1000
    },
    "by_criticality": {
      "high": 465, "medium": 7835, "low": 3200, "noise": 1000
    },
    "by_service": {
      "order-service": 5200, "payment-service": 4100, "user-service": 3200
    }
  },

  "engine_hints": {
    "retain": { "class_a": 465, "class_b": 7835, "class_c": 4200 },
    "filter": {
      "noise_count": 1000,
      "noise_rate_percent": 8.0,
      "estimated_monthly_savings_gb": 2.4
    },
    "prevent": {
      "debug_in_prod_count": 4200,
      "debug_rate_percent": 33.6,
      "services_with_debug": ["order-service", "user-service"]
    }
  },

  "insights": [
    "⚠️ 프로덕션에 Debug/Trace 로그 4200건(33.6%) — order-service, user-service 로그 레벨 상향 필요",
    "🔴 AppExceptions 300건 — payment-service에서 집중 발생 (NullPointerException 45%)",
    "🐢 payment-service → SQL 평균 응답시간 2.3초 (P95: 8.1초) — DB 쿼리 최적화 필요",
    "💰 노이즈 필터링 시 월 ~2.4GB 절감 가능"
  ],

  "slow_requests_top5": [
    {"name": "POST /api/orders", "avg_ms": 4200, "p95_ms": 12000, "service": "order-service"},
    {"name": "GET /api/payments/{id}", "avg_ms": 3100, "p95_ms": 8500, "service": "payment-service"}
  ],

  "slow_dependencies_top5": [
    {"type": "SQL", "target": "orders-db.database.windows.net", "avg_ms": 2300, "fail_rate": 2.1},
    {"type": "HTTP", "target": "external-payment-api.com", "avg_ms": 1800, "fail_rate": 5.3}
  ]
}
```

---

## 7. log-doctor-client-back 현재 구조와 진단 모듈 위치

### 7-1. 현재 코드베이스 (실제)

```
log-doctor-client-back/
├── function_app.py              ← Azure Functions 진입점 (현재: queue + timer 2개)
├── host.json
├── pyproject.toml               ← 의존성 (azure-monitor-query 추가 필요!)
├── requirements.txt
│
├── agent/
│   ├── __init__.py
│   ├── handshake.py             ← ✅ 이미 구현됨 — 멱등 핸드셰이크
│   ├── pipeline.py              ← ⚠️ 기존 AnalysisPipeline (엔진 4개 모두 실행)
│   │
│   ├── core/
│   │   └── config.py            ← ✅ 이미 구현됨 — Settings (tenant_id, provider_url 등)
│   │
│   ├── engines/                 ← 기존 엔진들 (진단과 분리)
│   │   ├── base.py              ← ✅ BaseEngine ABC
│   │   ├── detect.py            ← 스텁 (미구현)
│   │   ├── filter.py            ← 스텁 (미구현)
│   │   ├── prevent.py           ← 스텁 (미구현)
│   │   └── retain.py            ← 스텁 (미구현)
│   │
│   ├── infra/
│   │   ├── auth.py              ← ✅ 이미 구현됨 — Managed Identity 토큰
│   │   ├── azure.py             ← ✅ AzureClient (query_logs 메서드 있음 — 확장 필요)
│   │   └── provider.py          ← ✅ ProviderClient (should_i_run, report_result, handshake)
│   │
│   └── diagnosis/               ← 🔵 새로 만들 폴더 ──────────────────────────
│       ├── __init__.py
│       ├── runner.py            ← 진찰 오케스트레이터
│       ├── collector.py         ← LAW에서 KQL로 수집 (AzureClient.query_logs 활용)
│       ├── normalizer.py        ← 원본 → ld_ 스키마 변환
│       ├── classifier.py        ← 목적/중요도 점수 계산
│       ├── aggregator.py        ← 집계 + 인사이트 생성
│       └── mapping/
│           ├── __init__.py
│           └── table_registry.py  ← 4개 테이블 매핑 정의
│
└── tests/
    └── (기존 테스트)
```

### 7-2. 이미 있어서 재사용할 것

| 모듈 | 위치 | 재사용 방법 |
|:---|:---|:---|
| `Settings` | `agent/core/config.py` | `settings.tenant_id`, `settings.subscription_id` 그대로 사용 |
| `AzureClient` | `agent/infra/azure.py` | `query_logs(workspace_id, kql)` 확장하여 LAW 쿼리 |
| `ProviderClient` | `agent/infra/provider.py` | 진단 결과를 `report_result()`로 Provider에 전송 |
| `get_bearer_token` | `agent/infra/auth.py` | Managed Identity 인증 그대로 사용 |
| `perform_idempotent_handshake` | `agent/handshake.py` | 진단 트리거 시작 시 호출 |

### 7-3. 추가/수정할 것

**① `pyproject.toml` — 의존성 추가**

```toml
# azure-monitor-query 추가 (LAW KQL 쿼리용)
"azure-monitor-query>=1.4.0",
```

**② `agent/infra/azure.py` — `query_logs` 실제 구현**

```python
# 현재 (스텁):
async def query_logs(self, workspace_id: str, kql_query: str):
    return []

# 변경 후:
from azure.monitor.query.aio import LogsQueryClient
from azure.identity.aio import DefaultAzureCredential

async def query_logs(self, workspace_id: str, kql_query: str):
    credential = DefaultAzureCredential()
    client = LogsQueryClient(credential)
    response = await client.query_workspace(workspace_id, kql_query, timespan=timedelta(hours=1))
    await credential.close()
    return response
```

**③ `agent/core/config.py` — Settings에 workspace_id 추가**

```python
# 추가:
workspace_id: Optional[str] = Field(None, validation_alias="WORKSPACE_ID")
```

**④ `function_app.py` — 진단 트리거 추가**

```python
# 기존 트리거(queue_trigger, timer_trigger)는 유지
# 진단 전용 QueueTrigger 추가:

from agent.diagnosis.runner import DiagnosisRunner

@app.queue_trigger(arg_name="msg", queue_name="diagnosis-requests",
                   connection="AzureWebJobsStorage")
async def diagnosis_trigger(msg: func.QueueMessage):
    """진단 전용 트리거 — 초진(자동) 또는 재진(Teams 버튼 클릭)"""
    request = json.loads(msg.get_body().decode("utf-8"))
    logging.info(f"진단 요청 수신: {request.get('type', 'unknown')}")

    await perform_idempotent_handshake()

    runner = DiagnosisRunner(azure_client)
    result = await runner.run(
        workspace_id=settings.workspace_id,
        tenant_id=settings.tenant_id,
        agent_id=settings.agent_id,
        options=request.get("options"),
    )
    await provider_client.report_result({"trigger": "diagnosis", "results": result})
```

---

## 8. 구현 순서 (체크리스트)

### Phase 1: 순수 Python (LAW 없이) — 지금 시작 가능

- [ ] `agent/diagnosis/` 폴더 구조 생성
- [ ] `agent/diagnosis/mapping/table_registry.py` — `TableMapping` 데이터클래스 + 4개 테이블 레지스트리
- [ ] `agent/diagnosis/normalizer.py` — `SEVERITY_MAP` + 심각도 정규화 + 메시지 추출
- [ ] `agent/diagnosis/classifier.py` — 점수 계산 + 중요도 판정 + 엔진 힌트
- [ ] `agent/diagnosis/aggregator.py` — 분포 집계 + 인사이트 자동 생성
- [ ] `tests/test_normalizer.py` — 샘플 JSON으로 정규화 검증
- [ ] `tests/test_classifier.py` — 점수 경계값 검증
- [ ] `tests/test_aggregator.py` — 집계/인사이트 검증

### Phase 2: LAW 연동

- [ ] `pyproject.toml`에 `azure-monitor-query` 의존성 추가
- [ ] `agent/infra/azure.py` — `query_logs()` 실제 구현 (현재 스텁 → LogsQueryClient)
- [ ] `agent/core/config.py` — `workspace_id` 설정 추가
- [ ] `agent/diagnosis/collector.py` — `AzureClient.query_logs()` 호출 + 테이블 없음 핸들링
- [ ] `agent/diagnosis/runner.py` — 수집→정규화→분류→집계 통합 오케스트레이션
- [ ] 실제 LAW 통합 테스트

### Phase 3: Provider + function_app.py 연동

- [ ] `function_app.py`에 `diagnosis_trigger` (QueueTrigger) 추가
- [ ] `ProviderClient`에 `submit_diagnosis()` 메서드 추가 (또는 기존 `report_result` 활용)
- [ ] Provider Backend에 `POST /diagnosis-results` API 추가
- [ ] Cosmos DB에 진단 결과 저장 스키마 설계
- [ ] Teams 대시보드에 진단 결과 시각화

---

## 9. 확장 계획

```
현재: 애플리케이션 레이어만 (AppTraces/AppExceptions/AppRequests/AppDependencies)
      대상 스택: Python(FastAPI), .NET(ASP.NET Core), Spring Boot

이후:
  Phase 2: + 런타임 레이어 (ContainerLog, FunctionAppLogs)
  Phase 3: + 인프라 레이어 (AzureActivity, AzureDiagnostics)
  Phase 4: + 보안 레이어 (SigninLogs, AuditLogs)

→ table_registry.py에 테이블만 추가하면 파이프라인은 동일하게 동작
→ normalizer.py에 매핑 규칙만 추가하면 정규화도 확장됨
```
