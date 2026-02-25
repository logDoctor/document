# Application Insights — Log Doctor 관점 정리

> Application Insights가 뭔지, Log Doctor와 어떤 관계인지, 고객사 앱에 어떻게 붙어 있는지 정리합니다.

---

## 1. Application Insights의 위치

```
Azure Monitor (모니터링 통합 플랫폼)
  │
  ├── Metrics        ← CPU, 메모리 같은 숫자 지표
  ├── Alerts         ← 조건 충족 시 알림
  ├── Workbooks      ← 시각화 대시보드
  │
  ├── Application Insights  ← 🔵 애플리케이션 모니터링 (APM)
  │     앱에 SDK를 붙이면 요청, 에러, 외부 호출, 개발자 로그를 자동 수집
  │
  └── Log Analytics Workspace (LAW)  ← 로그 저장소
        Application Insights가 수집한 데이터가 여기에 테이블로 저장됨
        Log Doctor는 여기를 KQL로 조회
```

---

## 2. Application Insights가 수집하는 4가지

| 텔레메트리 | LAW 테이블 | 무엇을 수집하나 | 수집 방식 |
|:---|:---|:---|:---|
| **Traces** | `AppTraces` | 개발자가 `logger.info()` 등으로 남긴 로그 | SDK가 자동 전달 |
| **Exceptions** | `AppExceptions` | try-catch 예외, 미처리 예외 스택 트레이스 | 자동 수집 |
| **Requests** | `AppRequests` | 앱이 받은 HTTP 요청 (URL, 상태코드, 응답시간) | 자동 수집 |
| **Dependencies** | `AppDependencies` | 앱이 호출한 외부 서비스 (DB, API, Redis) | 자동 수집 |

### 수집 흐름

```
사용자 → HTTP 요청 → 고객사 앱
                       │
                       ├── [자동] 요청 정보 기록 → AppRequests
                       │
                       ├── logger.info("처리 시작") → AppTraces
                       │
                       ├── DB 쿼리 실행 ──────────→ AppDependencies
                       │
                       ├── 외부 API 호출 ─────────→ AppDependencies
                       │
                       ├── 예외 발생! ────────────→ AppExceptions
                       │
                       └── 응답 반환 (상태코드 기록) → AppRequests (업데이트)
                       
                       전부 Application Insights → LAW로 자동 전송
```

---

## 3. 스택별 Application Insights 연동

### Python (FastAPI)

```python
# 설치: pip install azure-monitor-opentelemetry
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor(connection_string="InstrumentationKey=...")
# → 이후 logging.info(), logging.error() 등이 자동으로 AppTraces에 기록
```

### .NET (ASP.NET Core)

```csharp
// NuGet: Microsoft.ApplicationInsights.AspNetCore
builder.Services.AddApplicationInsightsTelemetry();
// → ILogger<T>의 모든 로그가 자동으로 AppTraces에 기록
// → HttpClient, EF Core 호출이 자동으로 AppDependencies에 기록
```

### Spring Boot (Java)

```java
// Maven: applicationinsights-runtime-attach
ApplicationInsights.attach();
// → SLF4J/Logback 로그가 자동으로 AppTraces에 기록
// → RestTemplate, JDBC 호출이 자동으로 AppDependencies에 기록
```

---

## 4. Log Doctor와의 관계

```
고객사 역할:                    Log Doctor 역할:
앱에 AI SDK 설치               LAW에서 KQL로 읽기만 함
   ↓                              ↓
앱 실행 → 로그 발생             KQL 쿼리로 4개 테이블 조회
   ↓                              ↓
AI가 자동으로 LAW에 저장         수집 → 정규화 → 분류 → 점수
                                   ↓
                                진단 결과 리포트 생성
```

**핵심:**
- Log Doctor는 Application Insights API를 직접 호출하지 **않음**
- Log Doctor는 LAW만 조회 (`azure-monitor-query` → `LogsQueryClient`)
- Application Insights가 없는 고객사 → `AppTraces` 테이블 자체가 없음 → Collector에서 건너뜀

---

## 5. Application Insights 리소스 구조

```
고객사 Azure 구독
  │
  ├── Resource Group
  │     ├── 고객사 앱 (App Service, Container App, Functions 등)
  │     ├── Application Insights 리소스  ← Connection String 가짐
  │     └── Log Analytics Workspace      ← AI 데이터가 여기에 저장됨
  │
  └── Log Doctor Agent (Azure Functions)
        → LAW의 Workspace ID로 KQL 쿼리
```

### 필요한 정보

| 설정 | 용도 | 누가 제공 |
|:---|:---|:---|
| **Workspace ID** | LAW에 KQL 쿼리할 때 필요 | 고객사 (온보딩 시) |
| **Connection String** | AI SDK가 데이터를 보낼 때 필요 | 고객사가 자기 앱에 설정 (Log Doctor와 무관) |
| **Managed Identity 권한** | Agent가 LAW를 읽을 수 있도록 | 온보딩 시 RBAC 설정 |

> Log Doctor Agent는 **Workspace ID만 있으면** 진단을 실행할 수 있다.
> Connection String은 고객사 앱의 설정이지, Log Doctor가 건드리는 것이 아님.

---

## 6. Application Insights 미사용 고객사 대응

```
경우 1: AI를 쓰고 있는 고객사
  → LAW에 AppTraces/AppRequests/AppExceptions/AppDependencies 테이블 있음
  → 정상 진단 가능

경우 2: AI를 안 쓰는 고객사
  → LAW에 위 테이블이 없음
  → Collector가 KQL 실행 시 BadArgumentError 발생
  → 건너뛰고 다음 테이블 시도
  → 4개 다 없으면 "수집된 로그 없음" 결과 반환
  → Teams에서 "Application Insights 설정이 필요합니다" 안내

경우 3: AI를 일부만 쓰는 고객사
  → 예: Python 앱에 SDK는 설치했지만 DB Instrumentation은 안 한 경우
  → AppTraces/AppRequests는 있지만 AppDependencies가 비어있음
  → 있는 테이블만으로 부분 진단 수행
  → 리포트에 "의존성 모니터링 미설정" 안내 포함
```
