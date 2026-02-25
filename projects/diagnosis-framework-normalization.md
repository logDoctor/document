# Log Doctor: 프레임워크별 로그 차이와 정규화 전략

> Python(FastAPI), .NET(ASP.NET Core), Spring Boot(Java)의 로그가 왜 동일하게 처리되는지,
> 그리고 어디에서 차이가 발생하고 어떻게 대응하는지 정리합니다.

---

## 핵심 결론

```
Application Insights가 "번역기" 역할을 이미 하고 있기 때문에,
Log Doctor 진단 로직은 프레임워크를 구분할 필요가 없다.
```

---

## 1. Application Insights의 정규화 구조

```
Python (FastAPI)             .NET (ASP.NET Core)         Spring Boot (Java)
  logging.error("실패")       _logger.LogError("실패")     logger.error("실패")
       │                           │                           │
       │  OpenTelemetry SDK         │  AI NuGet 패키지          │  AI Java Agent
       ▼                           ▼                           ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │              Application Insights (번역 완료)                     │
  │                                                                  │
  │  전부 동일한 형태로 LAW에 저장:                                     │
  │  AppTraces    → SeverityLevel: 3,  Message: "실패"               │
  │  AppRequests  → ResultCode: "500", DurationMs: 1200              │
  │  AppExceptions → ExceptionType: "...", OuterMessage: "실패"      │
  │                                                                  │
  │  ⇒ 프레임워크가 뭐였는지는 이미 상관없음                             │
  └──────────────────────────────────────────────────────────────────┘
       │
       ▼
      LAW  ←── KQL은 여기만 바라봄. 통일된 스키마.
```

---

## 2. 진단에 영향 없는 이유: 공통 필드 비교

| LAW 필드 | Python 앱 | .NET 앱 | Spring 앱 | **동일?** |
|:---|:---|:---|:---|:---:|
| `SeverityLevel` | 0~4 (int) | 0~4 (int) | 0~4 (int) | ✅ |
| `ResultCode` | `"200"`, `"500"` | `"200"`, `"500"` | `"200"`, `"500"` | ✅ |
| `Success` | true/false | true/false | true/false | ✅ |
| `DurationMs` | 밀리초 | 밀리초 | 밀리초 | ✅ |
| `Message` | 문자열 | 문자열 | 문자열 | ✅ |
| `OperationId` | 요청 추적 ID | 요청 추적 ID | 요청 추적 ID | ✅ |
| `AppRoleName` | 서비스명 | 서비스명 | 서비스명 | ✅ |

**→ 진단 로직(정규화, 분류, 점수 계산)에 사용하는 모든 필드가 프레임워크와 무관**

---

## 3. 실제로 달라지는 것 (진단 로직에 영향 없음)

### 3-1. 기본 로그 레벨 차이

| 프레임워크 | 프로덕션 기본 레벨 | 결과 |
|:---|:---|:---|
| Python (FastAPI) | `WARNING` | DEBUG 로그 적게 나옴 |
| .NET (ASP.NET Core) | `Information` (but EF Core = `Debug`) | DEBUG 많이 나올 수 있음 |
| Spring Boot | `INFO` | 보통 수준 |

> **진단 영향**: 없음. 어차피 SeverityLevel 숫자로 판단하므로 양이 다를 뿐 분류 로직은 동일.

### 3-2. ExceptionType 형태 차이

| 같은 에러 | Python | .NET | Java |
|:---|:---|:---|:---|
| Null 참조 | `AttributeError` | `System.NullReferenceException` | `java.lang.NullPointerException` |
| HTTP 실패 | `httpx.ConnectError` | `System.Net.Http.HttpRequestException` | `org.springframework.web.client.RestClientException` |
| DB 실패 | `sqlalchemy.exc.OperationalError` | `Microsoft.Data.SqlClient.SqlException` | `java.sql.SQLException` |

> **진단 영향**: 없음. 분류 점수는 SeverityLevel로 계산하지 ExceptionType으로 하지 않음.
> 다만, **인사이트 문장 생성 시** 예외명을 보여주면 더 유용함.

### 3-3. 자동 수집되는 의존성 Type 차이

| 의존성 | Python | .NET | Spring |
|:---|:---|:---|:---|
| DB | `Type="SQL"` | `Type="SQL"` | `Type="SQL"` |
| HTTP | `Type="HTTP"` | `Type="HTTP"` | `Type="HTTP"` |
| Redis | `Type="Redis"` | `Type="Redis"` | `Type="Redis"` |
| 메시지 큐 | `Type="Azure Service Bus"` | `Type="Azure Service Bus"` | `Type="Kafka"` |

> **진단 영향**: 거의 없음. `Success`와 `DurationMs`로 판단하므로 `Type`은 리포트용.

---

## 4. KQL은 프레임워크를 몰라도 된다

모든 진단 KQL 쿼리는 프레임워크에 상관없이 동일:

```kql
// ✅ 에러율 계산 — 프레임워크 무관
AppRequests
| where TimeGenerated > ago(1h)
| summarize Total=count(), Errors=countif(toint(ResultCode) >= 500)
| extend ErrorRate = round(Errors * 100.0 / Total, 2)

// ✅ Debug 비율 — 프레임워크 무관
AppTraces
| where TimeGenerated > ago(1h)
| summarize Total=count(), DebugCount=countif(SeverityLevel <= 0)
| extend DebugRate = round(DebugCount * 100.0 / Total, 2)

// ✅ 느린 의존성 — 프레임워크 무관
AppDependencies
| where TimeGenerated > ago(1h)
| summarize P95=percentile(DurationMs, 95) by Type, Target
| where P95 > 3000
```

---

## 5. 프레임워크 감지가 필요한 유일한 경우

**인사이트를 더 구체적으로 보여줄 때만** 필요:

```kql
// ExceptionType에서 프레임워크 자동 감지
AppExceptions
| extend Stack = case(
    ExceptionType startswith "System.",             ".NET",
    ExceptionType startswith "java.",               "Java/Spring",
    ExceptionType startswith "Microsoft.",           ".NET",
    ExceptionType startswith "org.springframework.", "Spring",
    "Python"
  )
| summarize Count=count() by Stack, ExceptionType
| order by Count desc
```

**용도**: "이 고객사는 .NET 앱이고, NullReferenceException이 가장 많습니다" 같은 문장 생성.
**진단 점수에는 영향 없음** — 순수히 리포트 가독성을 위한 것.

---

## 정리

```
Q: 프레임워크마다 로그가 달라지면 어떻게 하지?
A: Application Insights가 이미 통일해놨다. 우리가 할 일 없음.

Q: KQL로 해결할 수 있나?
A: KQL이 해결하는 게 아니라, KQL이 보는 데이터가 이미 통일되어 있다.

Q: 그러면 프레임워크 차이는 어디서 다루나?
A: 인사이트 문장 생성할 때만. 점수/분류 로직에서는 안 다룸.
```
