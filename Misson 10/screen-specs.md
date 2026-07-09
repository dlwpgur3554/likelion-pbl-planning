# 🖥 화면별 상세 명세 (Screen Specifications)

> **작성자: 이제혁**
> Mission 10 · Dev Handoff Document
> 개발자용 화면 명세 · 입출력 데이터 · 예외 처리

MVP v1.0의 **18개 화면**에 대한 개발 명세입니다. 각 화면마다 진입 경로, API 호출, 상태 관리, 예외 처리를 명시했습니다.

---

## 📋 화면 인덱스

| ID | 화면명 | 그룹 | 우선순위 | Sprint |
|---|---|---|---|---|
| S-LANDING | 랜딩 페이지 | G1 | P0 | S1 |
| S-SIGNUP | 회원가입 | G1 | P0 | S1 |
| S-LOGIN | 로그인 | G1 | P0 | S1 |
| S-ONBOARD | 온보딩 튜토리얼 | G1 | P1 | S1 |
| S-DASH | 통합 대시보드 | G2 | P0 | S2 |
| S-EXCHANGE-LIST | 거래소 목록 | G3 | P0 | S2 |
| S-APILINK | API 키 입력 | G3 | P0 | S2 |
| S-APILINK-VERIFY | 권한 검증 | G3 | P0 | S2 |
| S-STRAT-LIB | 전략 라이브러리 | G4 | P0 | S3 |
| **S-STRAT** | GPT 전략 편집기 ★ | G4 | P0 | S3 |
| S-BACKTEST | 백테스팅 결과 | G4 | P0 | S3 |
| S-DEPLOY | 전략 배포 (2FA) | G4 | P0 | S3 |
| S-TRADES | 거래 내역 | G5 | P0 | S4 |
| S-RL-MONITOR | RL 엔진 모니터 | G5 | P0 | S4 |
| S-SETTINGS | 설정 | G6 | P1 | S4 |
| E-404 | 404 예외 | E | P0 | S1 |
| E-500 | 500 예외 | E | P0 | S1 |
| **E-API** | API 오류 화면 ★ | E | P0 | S4 |

---

## S-DASH · 통합 대시보드

### 기본 정보

| 항목 | 값 |
|---|---|
| **화면 ID** | `S-DASH` |
| **경로** | `/dashboard` |
| **진입 조건** | 인증 필요 (JWT) |
| **자동 갱신** | WebSocket 5초 |
| **모바일 지원** | Yes (반응형) |

### 입력 데이터 (API 호출)

```typescript
// GET /api/dashboard/summary
interface DashboardSummary {
  totalAssets: {
    amount: number;        // 원화 (KRW)
    currency: 'KRW';
    change24h: {
      amount: number;      // 절대값
      percentage: number;  // 백분율 (예: 5.4)
    };
    custodyLocation: string; // "업비트" | "바이낸스" (비수탁 명시용)
  };
  activeStrategies: {
    count: number;         // 활성 전략 수
    limit: number;         // 최대 전략 수 (플랜별)
    items: Strategy[];     // 최대 3개
  };
  performance30d: PerformancePoint[];  // 30일 성과 데이터
  recentAlerts: Alert[];   // 최근 10개
}
```

### 출력 (UI 상태)

| UI 요소 | 데이터 소스 | 렌더링 규칙 |
|---|---|---|
| 총 자산 위젯 | `totalAssets.amount` | `₩` prefix + 3자리 콤마 |
| 자산 변동률 | `totalAssets.change24h` | 음수: `▼`, 양수: `▲` |
| **비수탁 라벨 ⭐** | `totalAssets.custodyLocation` | `"업비트에 보관 중 (비수탁)"` |
| 활성 전략 카운터 | `activeStrategies.count / limit` | `3 / 5` 형식 |
| 성과 차트 | `performance30d` | Line chart, 30D 기본 |
| 알림 피드 | `recentAlerts` | 최신순, 최대 10개 |

### 예외 처리

| 상황 | UI 대응 |
|---|---|
| 🔴 **API 실패** (500) | 자산 위젯: "데이터 로드 실패" + 재시도 버튼 |
| 🟡 **거래소 미연동** | 자산 위젯 자리에 "거래소 연결하기 →" CTA (중앙 배치) |
| 🟡 **활성 전략 0개** | "운영 중인 전략이 없습니다" + "전략 만들기 →" |
| 🟡 **알림 0개** | "최근 활동이 없습니다" (빈 상태) |
| ⚫ **WebSocket 연결 실패** | 5초/15초/45초 재시도 → 폴링(30초) 폴백 |
| ⚫ **데이터 부족** (< 7일 운영) | "데이터 수집 중입니다" |

### 로딩 상태

- 초기 진입: **스켈레톤 UI** (각 위젯마다)
- 위젯 개별 갱신: **부드러운 페이드** (250ms)
- 차트 필터 변경: **300ms 애니메이션**

### 상태 관리 (프론트엔드)

```typescript
// Zustand / Redux 스토어
interface DashboardStore {
  summary: DashboardSummary | null;
  isLoading: boolean;
  error: Error | null;
  wsConnected: boolean;
  fetchSummary: () => Promise<void>;
  handleWsMessage: (msg: WsMessage) => void;
}
```

---

## S-STRAT · GPT 전략 편집기 ★ CORE

### 기본 정보

| 항목 | 값 |
|---|---|
| **화면 ID** | `S-STRAT` |
| **경로** | `/strategies/new` 또는 `/strategies/:id/edit` |
| **진입 조건** | 인증 + 거래소 1개 이상 연동 |
| **자동 저장** | ❌ (명시적 저장 필요) |
| **모바일 지원** | Partial (편집은 데스크탑 권장) |

### 입력 데이터

```typescript
// POST /api/gpt/strategy-suggest
interface GptStrategyRequest {
  userMessage: string;    // 최대 500자, 최소 3자
  currentStrategy?: Strategy;  // 기존 전략 컨텍스트
  conversationId?: string;
}

interface GptStrategyResponse {
  message: string;        // GPT 응답 텍스트
  suggestedBlocks: Block[]; // 추천 블록 구조
  conversationId: string;
}

// Block 타입
type BlockType = 'trigger' | 'action' | 'stoploss';
interface Block {
  id: string;
  type: BlockType;
  order: number;
  config: TriggerConfig | ActionConfig | StopLossConfig;
}
```

### UI 요소 명세 (M09 개선안 반영)

| 요소 | 인터랙션 | 예외 처리 |
|---|---|---|
| **채팅 입력창** | onChange 시 카운터 표시 (`0/500`) ⭐ M09 | 500자 초과: 입력 차단 |
| **전송 버튼** | 클릭 또는 Cmd+Enter | 3자 미만/공백: 비활성화 |
| **로딩 인디케이터 ⭐** | GPT 요청 중 표시 (M09 개선) | 30초 타임아웃 → "재시도" |
| **블록 hover 상태 ⭐** | 마우스 오버 시 배경 변화 (M09 개선) | — |
| **블록 편집 아이콘 ⭐** | 각 블록 우측 `✏️` (M09 개선) | — |
| **블록 편집 모달 ⭐** | 블록 클릭 시 열림 (M09 신규) | ESC 키 닫기 |
| **드래그 핸들** | 블록 순서 변경 | 트리거 블록은 최상단 고정 |
| **+ 블록 추가** | 클릭 시 블록 타입 선택 | 최대 10개 도달 시 비활성화 |
| **백테스팅 실행** | 검증 → API 호출 | 블록 검증 실패 시 빨간 테두리 |

### 저장 로직

```typescript
// POST /api/strategies
interface StrategyPayload {
  name: string;           // 자동 생성 or 사용자 입력
  description?: string;
  blocks: Block[];        // 최소 1개 트리거 필수
  status: 'draft';
}

// Response
interface StrategyResponse {
  id: string;
  name: string;
  ...
}
```

### 예외 처리

| 상황 | UI 대응 |
|---|---|
| 🔴 GPT API 실패 | "일시적 오류, 다시 시도해주세요" 토스트 |
| 🔴 30초 타임아웃 | "응답이 지연됩니다" + 재시도 버튼 |
| 🔴 트리거 블록 0개 시 저장 | 저장 버튼 비활성화 + tooltip |
| 🟡 유효 범위 벗어난 값 | 자동 보정 + 경고 표시 |
| 🟡 unsaved changes 이탈 | confirm 모달 |

---

## S-APILINK · API 키 입력

### 기본 정보

| 항목 | 값 |
|---|---|
| **화면 ID** | `S-APILINK` |
| **경로** | `/exchanges/new/keys` |
| **보안 등급** | 🔴 최고 (CRITICAL) |

### 보안 요구사항 ⚠️

| 항목 | 요구사항 |
|---|---|
| API/Secret 저장 | **클라이언트 메모리만** (sessionStorage/localStorage 금지) |
| 서버 전송 | AES-256 암호화 후 전송 |
| 서버 저장 | 암호화된 형태로 DB 저장 (별도 시크릿 벨트) |
| onPaste 즉시 마스킹 ⭐ | 페이스트 즉시 ●로 치환 (M09 개선) |
| 출금 권한 OFF 체크박스 ⭐ | 미체크 시 "권한 검증" 버튼 비활성화 (M09 개선) |
| 표시/숨김 토글 | 3초 후 자동 마스킹 복귀 |

### 입출력

```typescript
// POST /api/exchanges/verify
interface ApiKeyPayload {
  exchange: 'upbit' | 'binance';  // MVP는 upbit만
  apiKey: string;         // 클라이언트 암호화
  secretKey: string;      // 클라이언트 암호화
  withdrawPermissionOff: boolean;  // 필수 true
}

interface VerifyResponse {
  success: boolean;
  permissions: {
    query: boolean;       // 필수 true
    trade: boolean;       // 필수 true
    withdraw: boolean;    // 반드시 false여야 함
  };
  errorCode?: 'INVALID_KEY' | 'PERMISSION_DENIED' | 'WITHDRAW_ENABLED';
}
```

### 예외 처리

| 상황 | UI 대응 |
|---|---|
| 🔴 잘못된 API 키 형식 | 빨간 테두리 + "올바른 형식이 아닙니다" |
| 🔴 권한 검증 실패 | 실패 원인별 안내 (INVALID_KEY / PERMISSION_DENIED) |
| 🔴 **출금 권한 활성화 감지** | Critical 경고: "출금 권한을 반드시 비활성화해주세요" |
| ⚫ 30초 응답 없음 | "거래소 응답이 지연됩니다" |
| 🟡 중복 등록 시도 | "이미 연동된 거래소입니다. 키를 교체하시겠어요?" |

---

## E-API · API 오류 예외 화면 ★ (차별화 핵심)

### 기본 정보

| 항목 | 값 |
|---|---|
| **화면 ID** | `E-API` |
| **트리거** | 시스템 자동 (API 호출 실패 감지) |
| **폴링 주기** | 5초 |

### 상태 관리 (4-Step 트래커)

```typescript
type RecoveryStep = 
  | 'detected'      // 오류 감지 완료 ✓
  | 'reconnecting'  // 자동 재연결 시도 중 ↻
  | 'rl_response'   // RL 엔진 대응 중 ↻
  | 'normalized';   // 매매 정상화 ✓

interface RecoveryState {
  currentStep: RecoveryStep;
  errorType: 'NETWORK' | 'AUTH' | 'MAINTENANCE';
  detectedAt: string;    // ISO timestamp
  attempts: number;      // 재연결 시도 횟수
  positionRatio: {
    current: number;     // 현재 포지션 %
    target: number;      // RL 목표 %
  };
  estimatedCompletion?: string;  // 예상 완료 시각
}
```

### 자동 폴링

```typescript
// GET /api/system/recovery-status
// 5초 간격 폴링 (E-010)
// 30초 응답 없음: "연결 복구 시도 중" 폴백
```

### 예외 처리

| 상황 | UI 대응 |
|---|---|
| 🔴 재연결 3회 실패 | Critical 알림 (푸시 + 이메일 + 카톡) |
| 🔴 AUTH 에러 | 별도 화면 분기 (사용자 액션 필요) |
| 🟡 MAINTENANCE | 예상 종료 시각 표시 |
| 🟡 5분 초과 | "예상보다 오래 걸리고 있습니다" 추가 |

### 카피 원칙 ⭐

> **"사용자 개입 불필요"** — Mission 02 CUT 5 의도 반영
> UX 테스트에서 3명 중 3명이 정확히 인지 (100%, 평균 11초)

---

## 공통 예외 화면

### E-404 · 페이지 없음

- **경로**: `*` (모든 미매칭 경로)
- **UI**: "찾으시는 페이지가 없습니다" + "대시보드로 →" 버튼
- **로깅**: `analytics.track('404', { path: window.location.pathname })`

### E-500 · 서버 오류

- **UI**: "일시적인 서버 오류가 발생했습니다" + "다시 시도" + "고객센터 문의"
- **자동 복구**: 3회 재시도 후 사용자 액션 유도
- **로깅**: Sentry로 자동 전송

### 로딩 상태 (공통)

| 유형 | UI |
|---|---|
| **초기 페이지 로드** | 전체 스켈레톤 |
| **위젯 개별 로드** | 위젯 스켈레톤 |
| **버튼 액션** | 버튼 내 스피너 + 비활성화 |
| **인라인 액션** | 인라인 스피너 |

### 빈 상태 (공통)

모든 리스트/그리드 화면에 빈 상태 정의:
- 아이콘 (표준 회색 무채색)
- 상태 설명 ("아직 X가 없습니다")
- 다음 액션 CTA ("만들기 →")

---

**작성자: 이제혁**
