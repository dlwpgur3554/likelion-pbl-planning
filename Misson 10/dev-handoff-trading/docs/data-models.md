# 🗂 데이터 모델 (Data Models)

> **작성자: 이제혁**
> Mission 10 · Dev Handoff Document
> DB 스키마 · Prisma / TypeScript 타입 정의

MVP v1.0의 데이터 모델 명세입니다. PostgreSQL + Prisma ORM 사용.

---

## 📐 ERD 개요

```
User ─┬─ Session (1:N)
      ├─ Exchange (1:N)  ── ApiCredential (1:1)
      ├─ Strategy (1:N)
      │   ├─ Block (1:N)
      │   ├─ Backtest (1:N)
      │   └─ Trade (1:N)
      ├─ Alert (1:N)
      ├─ RlState (1:1)
      └─ TwoFactorAuth (1:1)
```

---

## 1. User

사용자 정보.

```prisma
model User {
  id             String   @id @default(cuid())
  email          String   @unique
  passwordHash   String
  nickname       String?
  emailVerified  Boolean  @default(false)
  plan           Plan     @default(FREE)
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt
  deletedAt      DateTime?

  // Relations
  sessions       Session[]
  exchanges      Exchange[]
  strategies     Strategy[]
  alerts         Alert[]
  rlState        RlState?
  twoFactor      TwoFactorAuth?

  @@index([email])
  @@index([createdAt])
}

enum Plan {
  FREE
  PRO      // v2.0+ 예정
}
```

---

## 2. Session

로그인 세션 (Refresh Token).

```prisma
model Session {
  id              String   @id @default(cuid())
  userId          String
  refreshToken    String   @unique
  userAgent       String?
  ipAddress       String?
  expiresAt       DateTime
  createdAt       DateTime @default(now())
  revokedAt       DateTime?

  user            User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([expiresAt])
}
```

---

## 3. Exchange · 거래소 연동

```prisma
model Exchange {
  id              String   @id @default(cuid())
  userId          String
  type            ExchangeType
  name            String
  connectedAt     DateTime @default(now())
  status          ExchangeStatus @default(ACTIVE)
  deletedAt       DateTime?

  // Non-custodial 명시 (읽기 전용)
  custodyType     String   @default("non-custodial")

  user            User             @relation(fields: [userId], references: [id])
  credential      ApiCredential?
  strategies      Strategy[]
  trades          Trade[]

  @@unique([userId, type])
  @@index([userId])
}

enum ExchangeType {
  UPBIT       // MVP
  BINANCE     // v1.1
  BYBIT       // v1.2
}

enum ExchangeStatus {
  ACTIVE
  ERROR       // API 오류 상태
  DISCONNECTED
}
```

---

## 4. ApiCredential · API 키 (암호화 저장)

**보안 규칙:**
- API/Secret은 **AES-256 암호화** 후 저장
- 별도 KMS(Key Management Service)로 마스터 키 관리
- 애플리케이션에서 조회 시에만 복호화 (로그에 절대 남기지 않음)

```prisma
model ApiCredential {
  id                    String   @id @default(cuid())
  exchangeId            String   @unique
  encryptedApiKey       String   @db.Text  // AES-256 encrypted
  encryptedSecretKey    String   @db.Text  // AES-256 encrypted
  keyVersion            Int      @default(1)

  permissions           Json     // { query: true, trade: true, withdraw: false }

  lastVerifiedAt        DateTime?
  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  exchange              Exchange @relation(fields: [exchangeId], references: [id])
}
```

**Permissions JSON 예시:**
```json
{
  "query": true,
  "trade": true,
  "withdraw": false
}
```

---

## 5. Strategy · 전략

```prisma
model Strategy {
  id              String   @id @default(cuid())
  userId          String
  exchangeId      String
  name            String
  description     String?  @db.Text
  status          StrategyStatus @default(DRAFT)

  capitalAllocation Decimal? @db.Decimal(20, 8)  // 배포 시 자본 배분

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  deployedAt      DateTime?
  archivedAt      DateTime?

  user            User      @relation(fields: [userId], references: [id])
  exchange        Exchange  @relation(fields: [exchangeId], references: [id])
  blocks          Block[]
  backtests       Backtest[]
  trades          Trade[]

  @@index([userId, status])
}

enum StrategyStatus {
  DRAFT           // 초안
  BACKTESTED      // 백테스팅 완료
  ACTIVE          // 실거래 배포 중
  PAUSED          // 일시 중지
  ARCHIVED        // 보관
}
```

---

## 6. Block · No-Code 편집 블록

```prisma
model Block {
  id              String   @id @default(cuid())
  strategyId      String
  type            BlockType
  order           Int
  config          Json     // 타입별 설정 스키마

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  strategy        Strategy @relation(fields: [strategyId], references: [id], onDelete: Cascade)

  @@index([strategyId, order])
}

enum BlockType {
  TRIGGER
  ACTION
  STOPLOSS
  CONDITION       // v1.1+
}
```

**Config JSON 스키마 예시:**

```typescript
// Trigger
interface TriggerConfig {
  condition: 'price_drop' | 'price_rise' | 'volume_spike';
  asset: 'BTC' | 'ETH' | ...;
  threshold: number;         // -100 ~ 100 (percentage)
  timeframe: '1h' | '4h' | '24h' | '7d';
}

// Action
interface ActionConfig {
  action: 'buy' | 'sell' | 'buy_split' | 'sell_split';
  capitalPercentage: number; // 1 ~ 100
  splits?: number;           // 분할 횟수
  interval?: string;         // 분할 간격
}

// StopLoss
interface StopLossConfig {
  triggerLoss: number;       // 0 ~ 50 (percentage)
  action: 'close_position' | 'reduce_position';
}
```

---

## 7. Backtest · 백테스팅 결과

```prisma
model Backtest {
  id              String   @id @default(cuid())
  strategyId      String
  status          BacktestStatus @default(RUNNING)

  // 입력
  period          String   // "90d"
  initialCapital  Decimal  @db.Decimal(20, 8)

  // 결과
  totalReturn     Decimal? @db.Decimal(8, 4)   // %
  winRate         Decimal? @db.Decimal(5, 2)   // %
  maxDrawdown    Decimal? @db.Decimal(8, 4)   // %
  sharpeRatio     Decimal? @db.Decimal(6, 4)
  totalTrades     Int?
  wins            Int?
  losses          Int?

  // 시계열 데이터 (JSON 압축)
  equityCurve     Json?    // [{ timestamp, value }, ...]
  benchmarkCurve  Json?    // BTC 단순 보유

  startedAt       DateTime @default(now())
  completedAt     DateTime?
  error           String?

  strategy        Strategy @relation(fields: [strategyId], references: [id])

  @@index([strategyId])
}

enum BacktestStatus {
  RUNNING
  COMPLETED
  FAILED
}
```

---

## 8. Trade · 실제 거래 내역

```prisma
model Trade {
  id              String   @id @default(cuid())
  userId          String
  exchangeId      String
  strategyId      String?   // null이면 수동 매매

  externalTradeId String?  @unique  // 거래소 발급 ID
  action          TradeAction
  asset           String
  price           Decimal  @db.Decimal(20, 8)
  quantity        Decimal  @db.Decimal(20, 8)
  totalAmount     Decimal  @db.Decimal(20, 8)

  fee             Decimal? @db.Decimal(20, 8)
  status          TradeStatus @default(PENDING)

  executedAt      DateTime?
  createdAt       DateTime @default(now())

  exchange        Exchange  @relation(fields: [exchangeId], references: [id])
  strategy        Strategy? @relation(fields: [strategyId], references: [id])

  @@index([userId, executedAt])
  @@index([strategyId])
  @@index([status])
}

enum TradeAction {
  BUY
  SELL
}

enum TradeStatus {
  PENDING
  EXECUTED
  FAILED
  CANCELLED
}
```

---

## 9. RlState · RL 엔진 상태 (사용자별)

```prisma
model RlState {
  id              String   @id @default(cuid())
  userId          String   @unique

  status          RlStatus @default(ACTIVE)
  volatilityLevel Decimal  @db.Decimal(5, 2)  // 0 ~ 100

  currentPositionRatio  Decimal @db.Decimal(5, 2)
  targetPositionRatio   Decimal @db.Decimal(5, 2)

  lastLearningAt        DateTime @default(now())
  lastAdjustedAt        DateTime?

  // 감지된 이슈
  activeIssue     Json?   // { errorType, detectedAt, attempts, ... }

  user            User    @relation(fields: [userId], references: [id])

  @@index([status])
}

enum RlStatus {
  ACTIVE          // 정상
  RESPONDING      // 이벤트 대응 중
  RECOVERING      // API 복구 중
  DISABLED
}
```

**ActiveIssue JSON 예시 (E-API 화면 데이터):**
```json
{
  "errorType": "NETWORK",
  "detectedAt": "2025-07-01T12:34:21Z",
  "currentStep": "rl_response",
  "attempts": 2,
  "positionAdjustment": {
    "from": 30,
    "to": 15
  }
}
```

---

## 10. Alert · 알림

```prisma
model Alert {
  id              String   @id @default(cuid())
  userId          String
  type            AlertType
  title           String
  message         String   @db.Text

  linkTo          String?  // 클릭 시 이동할 경로
  metadata        Json?

  readAt          DateTime?
  createdAt       DateTime @default(now())

  user            User     @relation(fields: [userId], references: [id])

  @@index([userId, createdAt(sort: Desc)])
  @@index([userId, readAt])
}

enum AlertType {
  TRADE_EXECUTED
  STRATEGY_TRIGGERED
  API_ERROR
  RL_ADJUSTMENT
  SYSTEM
}
```

---

## 11. TwoFactorAuth · 2FA 설정

```prisma
model TwoFactorAuth {
  id              String   @id @default(cuid())
  userId          String   @unique

  method          TwoFactorMethod @default(TOTP)
  secretEncrypted String   @db.Text
  backupCodes     Json     // 암호화된 백업 코드 배열
  enabled         Boolean  @default(false)

  lastUsedAt      DateTime?
  createdAt       DateTime @default(now())

  user            User     @relation(fields: [userId], references: [id])
}

enum TwoFactorMethod {
  TOTP            // Google Authenticator 등
  SMS             // v1.1+
}
```

---

## 📊 DB 규모 예측 (MVP 6개월 운영 기준)

| 테이블 | 예상 레코드 수 | 저장 용량 |
|---|---|---|
| User | 10,000 | 5 MB |
| Exchange | 10,000 | 3 MB |
| ApiCredential | 10,000 | 8 MB (암호화 오버헤드) |
| Strategy | 40,000 | 20 MB |
| Block | 200,000 | 80 MB |
| Backtest | 100,000 | 500 MB (시계열 JSON) |
| Trade | 5,000,000 | 2 GB |
| Alert | 10,000,000 | 3 GB |
| **합계** | — | **~ 6 GB** |

**성능 고려:**
- Trade / Alert 대량 쓰기 → 파티셔닝 (월별)
- Backtest 시계열 → 대안: TimescaleDB 이관 검토

---

## 🔒 데이터 보안 원칙

1. **모든 API/Secret 키는 AES-256 암호화** (KMS 관리)
2. **패스워드는 bcrypt (cost=12) 해싱**
3. **PII (이메일 등) DB 레벨 암호화 검토** (GDPR 대비)
4. **모든 삭제는 Soft Delete** (deletedAt 컬럼)
5. **감사 로그(Audit Log) 별도 테이블 관리**

---

**작성자: 이제혁**
