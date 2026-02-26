# Azure Monitor Agent (AMA) — Log Doctor 관점 정리

> Application Insights는 **앱 코드 안에 SDK를 심는 방식**이라 VM에서 돌아가는 레거시 앱에는 적용이 어렵습니다.
> Azure Monitor Agent(AMA)는 **VM 자체에 설치하는 에이전트**로, OS 로그와 앱 로그 파일을 LAW로 전송합니다.
>
> 이 문서는 AMA가 뭔지, Log Doctor와 어떤 관계인지, 어떻게 설정하는지 정리합니다.

---

## 1. Application Insights vs AMA — 언제 뭘 쓰나

```
                    Application Insights              Azure Monitor Agent (AMA)
                    ─────────────────────              ───────────────────────────
어디에 붙이나?       앱 코드에 SDK 설치                  VM에 에이전트 설치
뭘 수집하나?        앱의 요청, 에러, 의존성, 로그        OS 로그, 성능 카운터, 로그 파일
대상 리소스         App Service, Container App,         VM, VMSS, Arc-enabled Server
                    Azure Functions (서버리스)           (온프레미스도 가능)
LAW 테이블          AppTraces, AppRequests,             Syslog, Event, Perf,
                    AppExceptions, AppDependencies      SecurityEvent, 커스텀 테이블(_CL)
설치 방법           코드에 SDK 추가 (pip/nuget/maven)   VM Extensions 또는 Azure Portal
```

### 결론

```
서버리스 앱 (App Service, Functions)  →  Application Insights
VM 기반 워크로드                       →  Azure Monitor Agent (AMA)
둘 다 있는 고객사                      →  둘 다 사용 (LAW에 같이 모임)
```

> [!IMPORTANT] Log Doctor는 둘 다 상관없습니다.
> Log Doctor는 **LAW만 조회**합니다. 데이터가 AI에서 왔든 AMA에서 왔든, LAW 테이블에 있으면 진단 가능합니다.

---

## 2. AMA의 위치

```
Azure Monitor (모니터링 통합 플랫폼)
  │
  ├── Application Insights  ← 앱 코드 수준 (SDK)
  │     └ AppTraces, AppRequests, AppExceptions, AppDependencies
  │
  ├── Azure Monitor Agent    ← 🔵 인프라/OS 수준 (에이전트)
  │     └ Syslog, Event, Perf, SecurityEvent, Custom Tables
  │
  └── Log Analytics Workspace (LAW)  ← 둘 다 여기에 저장됨
        Log Doctor는 여기를 KQL로 조회
```

---

## 3. AMA가 수집하는 것들

### 3-1. 기본 데이터 소스

| 데이터 소스 | OS | LAW 테이블 | 설명 |
|:---|:---|:---|:---|
| **Syslog** | Linux | `Syslog` | 시스템 로그 (auth, cron, daemon, kern 등) |
| **Windows Event Log** | Windows | `Event` | System, Application, Security 이벤트 |
| **Performance Counters** | 둘 다 | `Perf` | CPU, 메모리, 디스크, 네트워크 |
| **Security Events** | Windows | `SecurityEvent` | 로그인 시도, 계정 변경, 정책 변경 |

### 3-2. 커스텀 로그 수집 (핵심!)

VM에서 돌아가는 앱이 파일로 로그를 쓸 때 (예: `/var/log/myapp/*.log`),
AMA가 이 파일을 읽어서 LAW의 **Custom Table**으로 전송할 수 있습니다.

```
VM 내부                              LAW
───────                              ───
/var/log/myapp/app.log    ──AMA──→   MyApp_CL (커스텀 테이블)
/var/log/myapp/error.log  ──AMA──→   MyApp_CL
/var/log/nginx/access.log ──AMA──→   NginxAccess_CL
C:\Logs\service.log       ──AMA──→   WinService_CL
```

> [!NOTE] `_CL` 접미사
> Custom Log 테이블은 반드시 `_CL` 접미사가 붙습니다.
> 예: `MyApp_CL`, `NginxAccess_CL`, `JavaBackend_CL`

---

## 4. AMA 설정 방법

### 4-1. AMA 설치

```
방법 1: Azure Portal
  VM 블레이드 → Settings → Extensions + applications → Add → Azure Monitor Agent

방법 2: Azure CLI
  az vm extension set \
    --publisher Microsoft.Azure.Monitor \
    --name AzureMonitorLinuxAgent \          # Windows: AzureMonitorWindowsAgent
    --resource-group myRG \
    --vm-name myVM

방법 3: Bicep (온보딩 자동화 시)
  resource ama 'Microsoft.Compute/virtualMachines/extensions@2023-09-01' = {
    parent: vm
    name: 'AzureMonitorLinuxAgent'
    properties: {
      publisher: 'Microsoft.Azure.Monitor'
      type: 'AzureMonitorLinuxAgent'
      autoUpgradeMinorVersion: true
    }
  }
```

### 4-2. DCR (Data Collection Rule) 설정

AMA는 **DCR(Data Collection Rule)**에 따라 "무엇을 수집할지"를 결정합니다.

```
DCR = "AMA에게 내리는 지시서"

  DCR 예시:
  ┌─────────────────────────────────────────┐
  │  이 VM에서 수집할 것:                      │
  │                                           │
  │  1. Syslog (auth, daemon 시설만)           │
  │     → LAW의 Syslog 테이블로 전송            │
  │                                           │
  │  2. /var/log/myapp/*.log 파일               │
  │     → LAW의 MyApp_CL 테이블로 전송          │
  │     → 파싱 규칙: 타임스탬프 + 레벨 + 메시지  │
  │                                           │
  │  3. Performance (CPU, Memory만)            │
  │     → LAW의 Perf 테이블로 전송              │
  └─────────────────────────────────────────┘
```

### 4-3. 커스텀 로그 DCR JSON 예시

```json
{
  "properties": {
    "dataSources": {
      "logFiles": [
        {
          "streams": ["Custom-MyApp_CL"],
          "filePatterns": ["/var/log/myapp/*.log"],
          "format": "text",
          "settings": {
            "text": {
              "recordStartTimestampFormat": "ISO 8601"
            }
          },
          "name": "myapp-logs"
        }
      ],
      "syslog": [
        {
          "streams": ["Microsoft-Syslog"],
          "facilityNames": ["auth", "authpriv", "daemon", "kern"],
          "logLevels": ["Warning", "Error", "Critical"],
          "name": "syslog-important"
        }
      ]
    },
    "destinations": {
      "logAnalytics": [
        {
          "workspaceResourceId": "/subscriptions/.../workspaces/myLAW",
          "name": "myLAW"
        }
      ]
    },
    "dataFlows": [
      {
        "streams": ["Custom-MyApp_CL"],
        "destinations": ["myLAW"],
        "transformKql": "source | extend Level = extract('\\[(\\w+)\\]', 1, RawData)",
        "outputStream": "Custom-MyApp_CL"
      },
      {
        "streams": ["Microsoft-Syslog"],
        "destinations": ["myLAW"]
      }
    ]
  }
}
```

> [!TIP] `transformKql`
> DCR에서 KQL 변환을 넣으면, **LAW에 저장되기 전에** 데이터를 파싱/필터링할 수 있습니다.
> 예: 로그 파일에서 `[ERROR]`, `[INFO]` 같은 레벨을 추출하여 별도 칼럼으로 저장.
> 이게 바로 Log Doctor의 **Filter 엔진**이 활용하는 DCR 제어 기능입니다.

---

## 5. Log Doctor와의 관계

### 5-1. 데이터 흐름

```
고객사 VM                        Azure                         Log Doctor
──────────                     ───────                        ──────────
앱이 로그 파일에 기록              AMA가 파일을 읽음
  /var/log/myapp/app.log           │
                                   ▼
                              DCR 규칙 적용
                              (필터링, 파싱)
                                   │
                                   ▼
                              LAW에 저장
                              (MyApp_CL 테이블)
                                                              KQL로 조회
                                                              수집 → 정규화 → 분류
                                                              진단 결과 리포트
```

### 5-2. Table Registry에 커스텀 테이블 추가

`log-diagnosis-implementation.md`의 `table_registry.py`에 커스텀 테이블을 추가하면 됩니다:

```python
# agent/diagnosis/mapping/table_registry.py 에 추가

# ── Custom Tables (고객사 VM 앱 로그) ──
"MyApp_CL": TableMapping(
    table_name="MyApp_CL",
    layer="application",
    severity_field="Level",         # DCR에서 파싱한 칼럼
    message_field="RawData",        # 또는 DCR에서 파싱한 Message 칼럼
    default_severity="INFO",
    context_fields=["Computer", "TimeGenerated"],
),
```

> [!IMPORTANT] 고객사마다 커스텀 테이블이 다름
> 커스텀 테이블명은 고객사마다 다릅니다 (예: `MyApp_CL`, `JavaBackend_CL`).
> 온보딩 시 고객사의 LAW에 어떤 `_CL` 테이블이 있는지 API로 조회하여 자동 등록하는 것이 이상적입니다.

---

## 6. AMA vs 기존 에이전트 비교

```
                    MMA (레거시)              AMA (신규 — 우리가 사용)
                    ────────────             ─────────────────────────
상태               2024년 8월 사용 중단      현재 표준
DCR 지원           ❌                        ✅ (핵심 기능)
멀티 호밍          제한적                    ✅ 여러 LAW로 동시 전송 가능
Managed Identity   ❌                        ✅
커스텀 로그        MMA Custom Logs (v1)      DCR 기반 Custom Logs (v2)
권장 사항          마이그레이션 필요           ← 이걸 사용
```

> [!CAUTION] MMA (Microsoft Monitoring Agent)는 2024년 8월부로 공식 사용 중단되었습니다.
> 고객사가 아직 MMA를 사용 중이면, 온보딩 시 AMA로의 마이그레이션을 안내해야 합니다.

---

## 7. Log Doctor 온보딩 시 AMA 관련 체크리스트

| 단계 | 확인 사항 | 자동화 가능 |
|:---|:---|:---|
| 1 | 고객사 VM에 AMA가 설치되어 있는가? | ✅ ARM API로 확인 가능 |
| 2 | DCR이 LAW와 연결되어 있는가? | ✅ ARM API로 확인 가능 |
| 3 | 어떤 데이터 소스가 수집 중인가? | ✅ DCR 내용 조회 |
| 4 | 커스텀 로그 테이블(`_CL`)이 있는가? | ✅ LAW 테이블 목록 조회 |
| 5 | Agent에 LAW 읽기 권한이 있는가? | ✅ RBAC 확인 |
| 6 | MMA → AMA 마이그레이션 필요한가? | ✅ VM Extensions 조회 |

### 자동 테이블 발견 KQL

```kql
// LAW에 있는 모든 테이블 목록 조회 (커스텀 테이블 포함)
search *
| distinct $table
| where $table endswith "_CL" or $table in (
    "Syslog", "Event", "Perf", "SecurityEvent",
    "AppTraces", "AppExceptions", "AppRequests", "AppDependencies"
)
| sort by $table asc
```

---

## 8. 정리: 고객사 유형별 로그 수집 흐름

```
케이스 1: 서버리스만 쓰는 고객사
  App Service + Functions
  → Application Insights SDK
  → AppTraces, AppRequests 등
  → LAW → Log Doctor 진단 ✅

케이스 2: VM만 쓰는 고객사
  VM (Linux/Windows)
  → AMA 설치
  → Syslog, Event, 커스텀 _CL 테이블
  → LAW → Log Doctor 진단 ✅

케이스 3: 하이브리드 고객사
  App Service (AI) + VM (AMA)
  → 두 데이터 소스 모두 같은 LAW에 저장
  → AppTraces + Syslog + MyApp_CL 모두 진단 가능 ✅

케이스 4: 아무것도 안 하는 고객사
  → LAW에 아무 데이터 없음
  → Collector가 모든 테이블에서 "없음"
  → 리포트: "모니터링 설정이 필요합니다" 안내
```
