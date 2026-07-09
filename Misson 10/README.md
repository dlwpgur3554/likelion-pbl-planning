# 🚀 OpenAI 연동형 자동매매 시스템 — 개발 협업 문서

> **작성자: 이제혁**
> Mission 10 · Final Dev Handoff Package
> Last updated: 2025-07

**Mission 01 ~ 09**의 모든 산출물을 개발자가 바로 사용할 수 있도록 정리한 **최종 개발 협업 패키지**입니다. MVP 스코프 확정, API/DB 명세, 프레젠테이션 자료가 모두 포함되어 있습니다.

---

## 📌 프로젝트 요약

| 항목 | 내용 |
|---|---|
| 서비스명 | OpenAI 연동형 자동매매 시스템 |
| 카테고리 | 핀테크 · AI · 트레이딩 |
| **차별화 3-Pillar** | ① No-Code 자연어 전략 ② 비수탁형 자산 ③ 강화학습 적응 |
| MVP 범위 | 18 화면 · 15 P0 기능 · 4 예외 화면 |
| 개발 기간 | 8주 · 4 스프린트 |
| 팀 규모 | 3-4명 (FE 2 + BE 1 + FS 1) |
| 기술 스택 | React + TypeScript + Node.js + PostgreSQL |

---

## 🗂 문서 인덱스

### 개발 전달용 문서 (`docs/`)

| 문서 | 내용 |
|---|---|
| [`screen-specs.md`](./docs/screen-specs.md) | 18개 화면 상세 명세 · 입출력 · 예외 처리 |
| [`api-spec.md`](./docs/api-spec.md) | 31개 API 엔드포인트 명세 (REST + WebSocket) |
| [`data-models.md`](./docs/data-models.md) | 11개 DB 모델 · Prisma 스키마 |
| [`error-states.md`](./docs/error-states.md) | 에러/빈상태/로딩 화면 명세 (15종) |

### 시각 자료 (`assets/`)

| 파일 | 내용 |
|---|---|
| [`slides/presentation.png`](./assets/slides/presentation.png) | **발표 슬라이드 12장** ⭐ |
| [`mvp-scope.png`](./assets/mvp-scope.png) | **MVP 스코프 & 스프린트 로드맵** ⭐ |
| `ia-diagram.png` | 정보 구조 (Mission 03) |
| `flow-1.png`, `flow-2.png` | 사용자 플로우 (Mission 04) |
| `wireframes-1.png`, `wireframes-2.png` | 와이어프레임 6화면 (Mission 06) |
| `scenario.png` | 6컷 시나리오 (Mission 02) |
| `priority-matrix.png` | 개선안 우선순위 (Mission 09) |
| `before-after-01/02/03.png` | 개선안 Before/After (Mission 09) |

---

## 🎯 MVP 스코프

![MVP Scope](./assets/mvp-scope.png)

### v1.0 포함 사항 (P0)

**G1 · 인증 & 온보딩**
- 이메일 회원가입 · 로그인 · 온보딩 튜토리얼

**G3 · 거래소 연동 (업비트만)**
- API 키 입력 (AES-256 암호화)
- 권한 검증 + **출금권한 OFF 체크박스** ⭐ M09
- 3-Step 마법사 UI

**G4 · 전략 스튜디오 ★ CORE**
- GPT 자연어 전략 생성 + **로딩 인디케이터** ⭐ M09
- No-Code 3-블록 편집기 (Trigger/Action/StopLoss)
- **블록 hover 상태 + 편집 모달** ⭐ M09
- 백테스팅 실행 + KPI 리포트
- 실거래 배포 (2FA 재인증)

**G2 · 대시보드**
- 통합 대시보드 + **비수탁 배너** ⭐ M09
- 자산 위젯 + **"업비트 보관 중" 라벨** ⭐ M09
- 실시간 성과 차트
- 알림 피드

**G5 · 거래 & 리포트**
- 실시간 거래 내역 (WebSocket)
- RL 엔진 자동 복구 (E-API) ★

**예외 화면 (4개)**
- E-404 · E-500 · E-API · E-OFFLINE

### v1.1+ Post-MVP (P1-P2)
바이낸스 다중 거래소 · 전략 마켓플레이스 · 모바일 앱 · Pro 결제 등 → 자세한 로드맵은 [`mvp-scope.png`](./assets/mvp-scope.png) 참조.

---

## 🚀 개발 우선순위

### Sprint Roadmap (8주)

```
┌──────────────────────────────────────────────────────────────┐
│ Sprint 1 (WK 1-2)  │ 인프라 + 인증 (G1)                       │
│                    │ Auth · JWT · DB 스키마 · CI/CD           │
├──────────────────────────────────────────────────────────────┤
│ Sprint 2 (WK 3-4)  │ 거래소 + 대시보드 (G2, G3)              │
│                    │ 업비트 API · S-APILINK · S-DASH          │
├──────────────────────────────────────────────────────────────┤
│ Sprint 3 (WK 5-6)★ │ 전략 스튜디오 (G4) · CORE               │
│                    │ GPT · No-Code · 백테스팅                 │
├──────────────────────────────────────────────────────────────┤
│ Sprint 4 (WK 7-8)  │ RL 엔진 + QA (G5, E)                    │
│                    │ 실시간 거래 · E-API · 통합 테스트         │
├──────────────────────────────────────────────────────────────┤
│         🚀 LAUNCH (WK 8 end)                                 │
└──────────────────────────────────────────────────────────────┘
```

**우선순위 결정 근거:**
1. **인프라 우선** (Sprint 1) — 나머지 스프린트가 의존
2. **거래소 연동을 CORE 앞에** (Sprint 2) — 전략 스튜디오는 거래소 없으면 무의미
3. **CORE에 2주 배정** (Sprint 3) — 가장 복잡한 GPT + No-Code
4. **RL 엔진 + QA를 마지막에** (Sprint 4) — 다른 모듈 완성 후 통합

---

## 🏗 아키텍처

### 기술 스택

**Frontend**
- React 18 + TypeScript
- Tailwind CSS
- Zustand (상태 관리)
- React Query (서버 상태)
- Recharts (차트)

**Backend**
- Node.js + Express
- Prisma ORM
- PostgreSQL 15
- Redis 7 (캐시 + 세션)
- Bull (큐)

**Infra**
- AWS ECS (컨테이너)
- CloudFront (CDN)
- AWS KMS (키 관리)
- GitHub Actions (CI/CD)

**External APIs**
- OpenAI GPT-4 (전략 생성)
- 업비트 API (거래)

### 시스템 다이어그램

```
┌────────────┐         ┌──────────────┐
│  Frontend  │────────>│  API Gateway │
│  (React)   │<────────│  (Express)   │
└────────────┘  HTTPS  └──────┬───────┘
                              │
      ┌───────────────────────┼───────────────────────┐
      │                       │                       │
┌─────▼─────┐          ┌──────▼──────┐        ┌───────▼──────┐
│ PostgreSQL│          │    Redis    │        │  OpenAI API  │
│ (Prisma)  │          │ (Cache/Q)   │        │  (GPT-4)     │
└───────────┘          └─────────────┘        └──────────────┘
      │                       │
      │                ┌──────▼──────┐
      │                │  RL Engine  │
      │                │  (Python)   │
      │                └──────┬──────┘
      │                       │
      └───────────────────────┴──────>┌──────────────┐
                                       │   Upbit API   │
                                       │  (거래 실행)   │
                                       └──────────────┘
```

---

## 🎤 발표 자료

![Presentation](./assets/slides/presentation.png)

**총 12장 슬라이드 구성:**
1. 표지
2. 문제 정의
3. 페르소나 (박서연)
4. 시나리오 (CUT 5 자동 복구)
5. 3-Pillar Solution
6. 7개 그룹 · 25개 기능
7. IA 구조 (Depth 4)
8. UX 테스트 결과 (H1/H2/H3)
9. Before/After 개선안
10. MVP 스코프 (Includes/Excludes)
11. 8주 로드맵
12. Thank You / Q&A

발표 소요 시간 예상: **약 15-20분** + Q&A 10분

---

## 📊 프로젝트 여정 (Mission 01 → 10)

| Mission | 결과물 | 핵심 지표 |
|---|---|---|
| M01 | 페르소나 정의 | 박서연 (Primary) + 김재훈 (Secondary) |
| M02 | 6컷 시나리오 | CUT 5 자동 복구 = 차별화 핵심 |
| M03 | IA | 27 화면 · Depth 4 · 7 그룹 |
| M04 | 사용자 플로우 | Flow 1 (Happy) + Flow 2 (Unhappy) |
| M05 | 기능 명세서 | 63 UI 요소 · 38 예외 처리 |
| M06 | 와이어프레임 리뷰 | 6화면 · 94% 일치도 |
| M07 | 통합 기획서 | 11챕터 · Decision Log 6개 |
| M08 | UX 테스트 (3명) | H1 ✅ · H2 ❌ · H3 ✅ |
| M09 | 개선안 도출 | 9개 개선안 · Impact×Effort |
| **M10** | **개발 협업 문서** | **현재 문서** |

---

## 🎓 시작 가이드 (개발자용)

### 1. 저장소 복제 후

```bash
git clone <repo-url>
cd trading-platform
```

### 2. 환경 변수 설정

`.env` 파일 생성:
```bash
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
OPENAI_API_KEY=sk-...
JWT_SECRET=...
KMS_KEY_ID=arn:aws:kms:...
UPBIT_API_BASE=https://api.upbit.com
```

### 3. 개발 서버 실행

```bash
# Backend
cd backend
npm install
npx prisma migrate deploy
npm run dev

# Frontend
cd frontend
npm install
npm run dev
```

### 4. 첫 스프린트 시작 전 필독

1. [`docs/screen-specs.md`](./docs/screen-specs.md) — S-LANDING부터 시작
2. [`docs/api-spec.md`](./docs/api-spec.md) — Auth API 먼저 구현
3. [`docs/data-models.md`](./docs/data-models.md) — User 모델부터
4. [`docs/error-states.md`](./docs/error-states.md) — 로딩 상태 규칙 숙지

---

## ✅ Mission 10 체크리스트 충족 현황

- [x] **최종 기획서/명세서**가 개발 전달용으로 정리됨 (`docs/` 4개 문서)
- [x] **MVP 범위 명확화** (v1.0: 18화면 + Post-MVP 별도)
- [x] **개발 우선순위**가 명확 (Sprint 1-4 계획)
- [x] **와이어프레임과 기능 명세** 최종 확정 (M09 개선안 반영)
- [x] **예외 화면** (에러/빈상태/로딩) 정의 (`error-states.md`)
- [x] **화면별 기능 · 입출력 · 예외** 모두 명시 (`screen-specs.md`)
- [x] **프레젠테이션 자료** 준비 (12장 슬라이드)
- [x] README에 **작성자: 이제혁** 포함

---

## 📎 관련 미션 링크

- Mission 01: 서비스 정의 & 페르소나
- Mission 02: 사용자 시나리오 보드
- Mission 03: IA (정보 구조)
- Mission 04: 사용자 플로우
- Mission 05: 기능 명세서
- Mission 06: 와이어프레임 리뷰 & 피드백
- Mission 07: 통합 서비스 기획서
- Mission 08: UX 테스트 설계 & 실행
- Mission 09: 테스트 결과 분석 & 개선안 도출
- **Mission 10: 개발 협업 문서 (현재)**

---

## 🎉 마치며

7개월간의 PBL 프로젝트가 마무리됐습니다.

**시작:** 페르소나 박서연의 Pain Point 하나 → *"자리를 비워도 AI가 대신 매매해줬으면"*

**종료:** 실제로 개발 가능한 MVP 명세서 · 8주 개발 로드맵 · 발표 자료

이 문서를 받은 개발팀이 8주 뒤에 v1.0을 성공적으로 런칭할 수 있기를 바랍니다.

---

**작성자: 이제혁**
*LikeLion PBL Final · 2025.07*
