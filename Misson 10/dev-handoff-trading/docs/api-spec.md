# 🔌 API 명세서 (API Specification)

> **작성자: 이제혁**
> Mission 10 · Dev Handoff Document
> RESTful API + WebSocket 엔드포인트 명세

MVP v1.0의 백엔드 API 명세입니다. 모든 엔드포인트는 `/api/v1` prefix를 사용합니다.

---

## 🔐 공통 규칙

### Base URL

```
Production:  https://api.trading-platform.com/v1
Staging:     https://staging-api.trading-platform.com/v1
Development: http://localhost:3000/api/v1
```

### 인증

- **방식**: JWT Bearer Token
- **Header**: `Authorization: Bearer <token>`
- **만료**: Access Token 1시간 / Refresh Token 30일

### 표준 응답 형식

```typescript
// 성공
{
  "success": true,
  "data": { ... },
  "meta"?: { ... }
}

// 실패
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "사용자에게 표시할 메시지",
    "details"?: { ... }
  }
}
```

### 표준 HTTP 상태 코드

| Code | 의미 |
|---|---|
| 200 | 성공 |
| 201 | 생성됨 |
| 400 | 잘못된 요청 |
| 401 | 인증 실패 |
| 403 | 권한 없음 |
| 404 | 리소스 없음 |
| 429 | 요청 초과 (Rate Limit) |
| 500 | 서버 오류 |

### Rate Limiting

| 엔드포인트 유형 | 제한 |
|---|---|
| 인증 (login, signup) | 5회 / 분 |
| GPT 호출 | 20회 / 분 |
| 일반 조회 | 100회 / 분 |
| WebSocket 메시지 | 무제한 (연결 유지) |

---

## 🚪 1. 인증 API (G1)

### POST /auth/signup

회원가입.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword123",
  "termsAgreed": true
}
```

**Response 201:**
```json
{
  "success": true,
  "data": {
    "userId": "usr_abc123",
    "email": "user@example.com",
    "verificationEmailSent": true
  }
}
```

**Errors:**
- `EMAIL_ALREADY_EXISTS` (400)
- `WEAK_PASSWORD` (400) — 8자 이상, 영문+숫자+특수문자

### POST /auth/verify-email

이메일 인증.

**Request:**
```json
{
  "email": "user@example.com",
  "code": "123456"
}
```

### POST /auth/login

로그인.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword123"
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJ...",
    "refreshToken": "eyJ...",
    "expiresIn": 3600,
    "user": {
      "id": "usr_abc123",
      "email": "user@example.com",
      "nickname": "노경민",
      "hasConnectedExchange": true
    }
  }
}
```

### POST /auth/refresh

토큰 갱신.

### POST /auth/logout

로그아웃 (Refresh Token 무효화).

---

## 🔗 2. 거래소 연동 API (G3)

### GET /exchanges

연동된 거래소 목록.

**Response 200:**
```json
{
  "success": true,
  "data": {
    "exchanges": [
      {
        "id": "exc_abc123",
        "type": "upbit",
        "name": "업비트",
        "connectedAt": "2025-06-01T10:00:00Z",
        "permissions": {
          "query": true,
          "trade": true,
          "withdraw": false
        },
        "status": "active"
      }
    ]
  }
}
```

### POST /exchanges/verify

API 키 검증 (저장 전 확인 단계).

**Request:**
```json
{
  "exchange": "upbit",
  "apiKey": "encrypted_key_here",
  "secretKey": "encrypted_secret_here",
  "withdrawPermissionOff": true
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "verified": true,
    "permissions": {
      "query": true,
      "trade": true,
      "withdraw": false
    }
  }
}
```

**Errors:**
- `INVALID_KEY` — 형식 오류
- `PERMISSION_DENIED` — 필수 권한 부족
- `WITHDRAW_ENABLED` — 출금 권한 활성화됨 (거부)

### POST /exchanges

API 키 등록 (verify 성공 후).

### DELETE /exchanges/:id

거래소 연동 해제 (2FA 재인증 필요).

---

## 📊 3. 대시보드 API (G2)

### GET /dashboard/summary

대시보드 종합 데이터.

**Response 200:** (S-DASH 상세 참조)

### GET /assets/total

총 자산 조회 (비수탁 정보 포함).

**Response 200:**
```json
{
  "success": true,
  "data": {
    "totalKrw": 12450000,
    "change24h": {
      "amount": 640000,
      "percentage": 5.4
    },
    "breakdown": [
      {
        "exchange": "upbit",
        "custodyType": "non-custodial",
        "amount": 12450000
      }
    ]
  }
}
```

---

## 🧠 4. 전략 API (G4) ★ CORE

### POST /gpt/strategy-suggest

GPT에게 전략 요청.

**Request:**
```json
{
  "userMessage": "BTC 5% 떨어지면 분할매수",
  "conversationId": null
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "conversationId": "conv_xyz",
    "message": "좋아요! 다음 전략을 제안합니다...",
    "suggestedBlocks": [
      {
        "id": "blk_1",
        "type": "trigger",
        "order": 1,
        "config": {
          "condition": "price_drop",
          "asset": "BTC",
          "threshold": -5,
          "timeframe": "24h"
        }
      },
      {
        "id": "blk_2",
        "type": "action",
        "order": 2,
        "config": {
          "action": "buy_split",
          "capitalPercentage": 30,
          "splits": 3,
          "interval": "2h"
        }
      }
    ]
  }
}
```

**Rate Limit:** 20회 / 분

**Timeout:** 30초 (프론트엔드도 동일하게)

### GET /strategies

내 전략 목록.

### POST /strategies

전략 저장.

**Request:**
```json
{
  "name": "BTC 분할매수 v1",
  "description": "...",
  "blocks": [ ... ]
}
```

### PATCH /strategies/:id

전략 수정.

### DELETE /strategies/:id

전략 삭제 (soft delete).

### POST /strategies/:id/toggle

전략 ON/OFF (2FA 재인증 필요 for OFF→ON).

### POST /backtest/run

백테스팅 실행.

**Request:**
```json
{
  "strategyId": "str_abc",
  "period": "90d",
  "initialCapital": 1000000
}
```

**Response 200:**
```json
{
  "success": true,
  "data": {
    "backtestId": "bt_xyz",
    "result": {
      "totalReturn": 24.7,
      "winRate": 68,
      "maxDrawdown": -8.2,
      "sharpeRatio": 1.84,
      "totalTrades": 62,
      "wins": 42,
      "losses": 20,
      "equityCurve": [ ... ],
      "benchmarkCurve": [ ... ]
    }
  }
}
```

**Timeout:** 60초

### POST /strategies/:id/deploy

실거래 배포.

**Request:** 2FA 코드 필수
```json
{
  "twoFactorCode": "123456",
  "capitalAllocation": 1000000
}
```

---

## 📈 5. 거래·리포트 API (G5)

### GET /trades

거래 내역 조회.

**Query:**
- `strategyId?` — 필터
- `from?`, `to?` — 기간
- `limit?` (기본 50)
- `cursor?` — 페이지네이션

### GET /rl-engine/status

RL 엔진 상태.

**Response:**
```json
{
  "success": true,
  "data": {
    "status": "active",
    "volatilityDetected": false,
    "currentPositionRatio": 30,
    "targetPositionRatio": 30,
    "lastLearningTime": "2025-07-01T09:15:00Z"
  }
}
```

---

## ⚠️ 6. 시스템 API (예외 대응)

### GET /system/recovery-status ★

API 오류 복구 상태 (E-API 화면 폴링용, 5초 간격).

**Response 200:**
```json
{
  "success": true,
  "data": {
    "hasIssue": true,
    "currentStep": "rl_response",
    "errorType": "NETWORK",
    "detectedAt": "2025-07-01T12:34:21Z",
    "attempts": 2,
    "positionRatio": {
      "current": 22,
      "target": 15
    },
    "estimatedCompletion": "2025-07-01T12:36:00Z"
  }
}
```

### GET /system/health

시스템 상태 (헬스체크).

---

## 🔌 7. WebSocket API

### 연결

```
wss://api.trading-platform.com/v1/ws
Authorization: Bearer <token>
```

### 재연결 정책

프론트엔드 구현:
- 초기 연결 실패: 즉시 재시도
- 이후 실패: 5초 → 15초 → 45초 백오프
- 3회 연속 실패: 폴링(30초)으로 폴백

### 메시지 형식

```typescript
interface WsMessage {
  type: WsMessageType;
  timestamp: string;
  data: any;
}

type WsMessageType =
  | 'trade_executed'      // 거래 체결
  | 'strategy_triggered'  // 전략 발동
  | 'price_update'        // 가격 갱신
  | 'alert'               // 알림
  | 'api_error'           // API 오류 감지 (E-API 트리거)
  | 'recovery_update'     // 복구 진행 업데이트
  | 'position_change';    // 포지션 변경
```

### 예시: 거래 체결 알림

```json
{
  "type": "trade_executed",
  "timestamp": "2025-07-01T12:34:56Z",
  "data": {
    "tradeId": "trd_abc",
    "strategyId": "str_xyz",
    "action": "buy",
    "asset": "BTC",
    "price": 45200000,
    "quantity": 0.01,
    "totalAmount": 452000
  }
}
```

---

## 📊 API 요약

| 그룹 | 엔드포인트 수 | 우선순위 |
|---|---|---|
| G1 인증 | 6 | P0 |
| G2 대시보드 | 2 | P0 |
| G3 거래소 | 4 | P0 |
| G4 전략 | 9 | P0 |
| G5 거래·리포트 | 3 | P0 |
| G6 설정 | 4 | P1 |
| 시스템 | 2 | P0 |
| WebSocket | 1 | P0 |
| **합계** | **31** | — |

---

**작성자: 이제혁**
