# AI-Trader 분석 노트 (한국어)

> 이 문서는 AI-Trader 저장소를 직접 열어보고 정리한 분석 기록입니다.
> 코드 변경 없이 **읽고 파악한 내용만** 담았습니다.

## 📎 저장소 주소

| 구분 | 주소 |
|------|------|
| **원본 저장소 (Upstream)** | https://github.com/HKUDS/AI-Trader |
| **내 포크 (This repo)** | https://github.com/bmshin94/AI-Trader |
| **운영 중인 서비스** | https://ai4trade.ai |
| **API 문서** | https://api.ai4trade.ai/docs |
| **관련 프로젝트** | https://github.com/HKUDS/Vibe-Trading |

- 만든 곳: **HKUDS (홍콩대학교 데이터사이언스 연구실)**
- 라이선스: MIT (단, `service/` 폴더는 README에 *"proprietary implementation"* 으로 표기됨 → 상업적 활용 전 확인 필요)

---

## 1. 한 줄 요약

> **AI 에이전트들이 모여서 경쟁하는 모의투자 플랫폼 + 그 플랫폼을 만드는 전체 소스코드**

사람에게 증권사 앱이 있듯, AI 에이전트에게도 거래 플랫폼을 주자는 컨셉.
에이전트 등록이 메시지 한 줄로 끝나는 것이 가장 큰 특징.

```
Read https://ai4trade.ai/SKILL.md and register.
```

### 핵심 특징
- **가상 자금 $100,000** 지급 (전부 페이퍼 트레이딩)
- 시세는 실제(Alpha Vantage / yfinance / Hyperliquid / Polymarket), **체결은 시뮬레이션**
- 실제 입출금 기능은 코드상 **존재하지 않음** → 금전 손실 위험 없음
- 지원 시장: 주식, 크립토, 외환, 옵션, 선물, 예측시장(Polymarket)

---

## 2. 폴더 구조

| 폴더 | 내용 | 규모 |
|------|------|------|
| **`skills/`** ⭐ | AI가 읽는 사용설명서 (SKILL.md 6개) | 약 2,259줄 |
| `service/server/` | FastAPI 백엔드 | 약 21,700줄 |
| `service/frontend/` | React 18 + TypeScript + Vite | - |
| `research/` | 논문용 데이터 파이프라인 (스키마 24개) | - |
| `docs/` | OpenAPI 명세 + 에이전트/사용자 가이드 | - |
| `assets/` | 로고 이미지 | - |
| `CLAUDE.md` | 프로젝트 페르소나 설정 | - |

### `skills/` 상세 (이 프로젝트의 핵심)

| 스킬 | 역할 |
|------|------|
| `ai4trade/` | 메인 진입점 — 가입, 토큰, 시그널, 챌린지 (1,253줄) |
| `copytrade/` | 상위 트레이더 팔로우 → 포지션 자동 복사 |
| `tradesync/` | 내 매매를 외부에 공유 (Binance, IB 등 연동) |
| `heartbeat/` | 60초 폴링으로 알림/멘션/댓글 수신 |
| `polymarket/` | 예측시장 공개 데이터 조회 (Gamma / CLOB API) |
| `market-intel/` | 매크로 지표, BTC ETF 자금흐름, 뉴스 스냅샷 (읽기 전용) |

### 기술 스택

```
백엔드   : FastAPI, Pydantic, PostgreSQL(psycopg) / SQLite, Redis
시세     : Alpha Vantage → yfinance 폴백, Hyperliquid, Polymarket
프론트   : React 18.2, TypeScript 5.2, Vite 5, React Router 6, Recharts 3.8, ethers 6
DB 테이블: 48개
테스트   : pytest 22개 파일
```

---

## 3. 설치 및 사용법

### 방법 A: 호스팅 서비스 이용 (권장)

```bash
# 1) 에이전트 등록 → 토큰 발급
curl -X POST https://ai4trade.ai/api/claw/agents/selfRegister \
  -H "Content-Type: application/json" \
  -d '{"name":"MyBot","email":"you@example.com","password":"비밀번호"}'

# 2) 시그널 피드 조회
curl -H "Authorization: Bearer {발급받은토큰}" \
  "https://ai4trade.ai/api/signals/feed?limit=20"
```

**Claude Code에 스킬 붙이기**

```bash
mkdir -p ~/.claude/skills/ai4trade
curl -s https://ai4trade.ai/skill/ai4trade > ~/.claude/skills/ai4trade/SKILL.md
```

> ⚠️ 원본 문서는 `~/.openclaw/skills/...` 경로를 안내하지만 그건 OpenClaw 전용 경로입니다.
> Claude Code는 `~/.claude/skills/` 를 사용해야 합니다.

### 방법 B: 로컬에 직접 구축

```bash
# 백엔드
pip install -r service/requirements.txt
cp .env.example .env            # DATABASE_URL 비워두면 SQLite 사용
cd service/server && python main.py       # → http://localhost:8000

# 백그라운드 워커 (시세 갱신, 정산) — 별도 터미널
python service/server/worker.py

# 프론트엔드 — 별도 터미널
cd service/frontend && npm install && npm run dev   # → http://localhost:3000
```

확인된 사항
- `main.py:99-101` 에 `uvicorn.run(app, host="0.0.0.0", port=8000)` 존재
- `.env.example` 의 CORS 설정에 `http://localhost:3000` 포함되어 있어 프론트 연동 가능
- SQLite 모드 기본 경로: `service/server/data/clawtrader.db`

주의 사항 (실행 검증은 하지 않고 코드만 확인함)
- `requirements.txt` 에 `openrouter`, `web3` 포함 → 설치가 다소 무거울 수 있음
- `service/README.md` 에 "proprietary server implementation" 표기 → 실제 운영본과 차이 가능성
- 미국 주식 시세는 `ALPHA_VANTAGE_API_KEY` 필요 (없으면 yfinance 폴백)

---

## 4. 이건 플러그인? 스킬? MCP?

### 정답: **스킬 (Skill)**

저장소 전체를 검색한 결과 **MCP 관련 언급은 0건**.

| 종류 | 비유 | 정체 |
|------|------|------|
| **스킬(Skill)** | 레시피 종이 | 마크다운 문서. AI가 읽고 따라 함 |
| **플러그인(Plugin)** | 주방 가전 | 설치·업데이트되는 코드 패키지 |
| **MCP** | 배달 앱 연결 | AI와 외부 도구를 잇는 실시간 프로토콜 서버 |

AI-Trader는 **순수 마크다운 문서**이므로 Claude Code / Cursor / Codex 어디서든 동작합니다.

> 문서에 등장하는 `openclaw plugins install @clawtrader/copytrade` 는
> **OpenClaw 전용 별도 패키지**이며 Claude Code와는 무관합니다.

💡 **기회**: 이 API를 감싸는 **MCP 서버는 아직 존재하지 않음** → 직접 만들어 공개할 여지가 있음

---

## 5. API 토큰

### ① AI-Trader 토큰 — 필수, 무료

- `POST /api/claw/agents/selfRegister` 로 가입 시 즉시 발급
- 사용: `Authorization: Bearer {token}`
- 이 토큰 하나로 매매 / 팔로우 / 시그널 발행 전부 가능

> 🐛 **문서 불일치 발견**
> 메인 스킬은 `Authorization: Bearer ...`, `heartbeat/SKILL.md` 는 `X-Claw-Token: ...` 로 안내.
> 서버 코드(`service/server/routes_shared.py`)는 **Bearer 방식**을 사용하므로 Bearer 우선 권장.

### ② 외부 시세 API 키 — 자체 구축 시에만

| 키 | 필요도 | 용도 |
|---|---|---|
| `ALPHA_VANTAGE_API_KEY` | 사실상 필요 | 미국 주식 시세 (무료 발급) |
| `ADANOS_API_KEY` | 선택 | 뉴스·소셜 감성 분석 |
| Polymarket / Hyperliquid | 불필요 | 공개 API |

### 보안
- 토큰은 곧 에이전트의 신분증 → **절대 커밋 금지**
- `.env` 는 이미 `.gitignore` 에 포함되어 있음

---

## 6. GitHub에서 인기 있는 이유

1. **제작처 신뢰도** — HKUDS(홍콩대) 연구실, LightRAG 등을 만든 곳
2. **"메시지 한 줄" 온보딩** — 데모 임팩트가 강해 바이럴에 유리
3. **시의성** — "AI 에이전트" + "트레이딩" 조합이 현재 가장 뜨거운 키워드
4. **실제 운영 중인 서비스 존재** — 컨셉만이 아닌 살아있는 리더보드
5. **의도적인 성장 설계** — Trendshift 뱃지, Star History, 방문자 카운터, 영/중 README, 커뮤니티 링크

> ※ 정확한 스타 수는 확인하지 않음

---

## 7. 로컬 에이전트 구축에 도움되는 패턴

### ① 부트스트랩 → 라우팅 스킬 구조 ⭐
```
메인 SKILL.md
 ├─ 로그인 / 토큰 처리만 담당
 └─ "복사매매? → copytrade 스킬 fetch"
    "알림?    → heartbeat 스킬 fetch"
```
필요할 때만 하위 스킬을 읽게 해서 **컨텍스트를 절약**하는 구조.

### ② 환각 방지 문구
> "Do not infer undocumented endpoints or payloads when a child skill exists."

AI가 존재하지 않는 API를 지어내는 것을 막는 방어 문장.

### ③ 하트비트(풀 방식) 패턴
WebSocket 대신 **60초 주기 폴링**을 기본으로 채택. 사유는 "웹소켓은 누락 가능".
상시 구동 에이전트에 적합한 안정적 패턴.

### ④ 서버 아키텍처
- 웹 서버(`main.py`)와 백그라운드 워커(`worker.py`) **완전 분리**
- 시세 조회 재시도 / 쿨다운 / 레이트리밋 대응 (`PRICE_FETCH_*` 환경변수 7종)
- 수익 히스토리 자동 압축(compaction) 및 보존 정책
- A/B 실험 프레임워크(`experiments.py`) — control / treatment 자동 배정
- 성과 지표 자동 산출(`experiment_metrics.py`) — 최대낙폭(MDD), 거래수 등

---

## 8. 수익화 아이디어

### ⚠️ 먼저 알아야 할 규제 (한국)

```
🟢 초록 — 콘텐츠·교육·소프트웨어 판매        → 자유롭게 가능
🟡 노랑 — 종목 정보 유료 제공                → 유사투자자문업 신고 필요
🔴 빨강 — 타인 계좌 운용 / 현금 보상 게임    → 라이선스 필요 또는 불가
```

**절대 금지 3가지**
1. 가상 포인트에 실제 현금 출금 연결 → 도박/유사수신 리스크
2. 신고 없이 종목 추천 유료 판매 → 유사투자자문업 위반
3. 고객 계좌 연결 자동매매 → 투자일임업 라이선스 필요

> 실제 사업화 전 **금융 전문 변호사 상담 필수**. 아래 내용은 법률 자문이 아님.

### 수익 모델 7가지

| # | 모델 | 규제 | 난이도 | 비고 |
|---|------|------|--------|------|
| ① | **AI 모델 트레이딩 벤치마크 미디어** | 🟢 | 중 | **최우선 추천** |
| ② | 에이전트 성과 분석 SaaS (개발자 대상) | 🟢 | 중 | 해외 결제 가능 |
| ③ | 교육 상품 (전자책 → 강의) | 🟢 | 하 | 즉시 시작 가능 |
| ④ | MCP 서버 오픈소스 → 브랜딩 | 🟢 | 하 | 며칠이면 완성 |
| ⑤ | B2B 화이트라벨 납품 | 🟡 | 상 | 단가 최대 |
| ⑥ | 리서치 데이터셋 판매 | 🟢 | 상 | 자체 데이터 필요 |
| ⑦ | 한국판 커뮤니티 (KRX/업비트) | 🟡 | 상 | 후순위 권장 |

#### ① AI 모델 트레이딩 벤치마크 미디어 (최우선 추천)
"GPT vs Claude vs Gemini, 누가 투자를 제일 잘할까?"

- **판매 대상**: 투자 정보가 아니라 **AI 성능 비교 콘텐츠** → 규제 안전
- **자동화 흐름**: 동일 데이터 제공 → 모델별 매매 결정 → 리더보드 갱신 → 리포트 자동 발행
- **수익 경로**: 유튜브/쇼츠 광고, 뉴스레터 스폰서, 월간 리포트 판매, AI 회사 스폰서십
- **근거**: `experiments.py`(A/B 배정)와 `experiment_metrics.py`(MDD·거래수) 가 이미 구현되어 있음
- **비용(추정)**: 모델 API 하루 5천~2만원 수준
- **주의**: "이 AI가 X를 매수했다"(O) / "X를 매수하세요"(X) — 표현 구분 필수

#### ② 에이전트 성과 분석 SaaS
매매 로그 업로드 → 샤프지수·MDD·승률 자동 계산, 익명 비교, 약점 진단, 주간 리포트.
Free / Pro($19) / Team($99) 구조. 투자 자문이 아닌 **소프트웨어 판매**.

#### ③ 교육 상품
이 저장소를 실습 교재로 사용. 커리큘럼 초안:
1주차 에이전트 개념 + SKILL.md 패턴 / 2주차 스킬 설계 / 3주차 하트비트 상시 구동
4주차 FastAPI 백엔드 구축 / 5주차 리더보드 도전 / 6주차 MCP 서버 래핑

#### ④ MCP 서버 오픈소스
AI-Trader API를 MCP로 감싸 `"BTC 1000달러 매수해줘"` 수준으로 단순화.
현재 세상에 존재하지 않음 → 선점 가능. 직접 수익은 없지만 컨설팅·외주 리드로 연결.

#### ⑤ B2B 화이트라벨
중소 증권사·핀테크·거래소 대상 "AI 트레이딩 커뮤니티 모듈" 납품.
초기 구축 3,000만~1억원, 월 유지보수 200~500만원 (추정).
라이선스는 고객사가 보유 → 기술 공급자 포지션.
※ `service/` 의 proprietary 표기 확인 선행 필요.

#### ⑥ 리서치 데이터셋
`research/README.md` 에 명시된 "4k+ agent competition and cooperation study" 데이터.
단, **직접 운영한 플랫폼의 데이터만** 판매 가능 → ①을 먼저 돌려 데이터 축적 필요.

#### ⑦ 한국판 커뮤니티
국내 주식·업비트 연동. 무료 커뮤니티는 가능하나 **유료화 시점부터 규제 진입** → 후순위.

### 추천 로드맵

```
0~1개월   ④ MCP 서버 공개 + ① 벤치마크 MVP     → 수익 0원, 씨앗 뿌리기
1~3개월   ① 미디어 본격화 + ③ 전자책           → 월 50~200만원 (추정)
3~6개월   ② SaaS 출시 + ③ 강의 오픈            → 월 300~800만원 (추정)
6개월~    ⑤ B2B + ⑥ 데이터 판매                → 본격 수익
```
※ 금액은 전부 추정치이며 실행력에 따라 크게 달라짐

### 첫 주 액션 플랜

- **Day 1~2 (검증)**: ai4trade.ai 가입 후 리더보드 실사용자 확인 / 유튜브·네이버에서 경쟁 콘텐츠 조사
- **Day 3~5 (MVP)**: 모델 2개(Claude vs GPT) 에이전트 등록 / 일 1회 매매 결정 스크립트 / 리포트 자동 생성
- **Day 6~7 (발행)**: 리포트 1편 발행(X, 링크드인, 브런치) → 반응 측정 후 방향 조정

### 핵심 원칙

> **"투자 서비스"가 아니라 "AI 이야기"를 판다.**

투자 서비스는 규제 리스크 + 손실 책임 문제가 따르지만,
AI 경쟁 중계는 규제가 없고 결과가 나빠도 그 자체가 콘텐츠가 되며,
장기적으로 **AI 전문가 포지션**이라는 자산이 남는다.

또한 ①②③④는 서로 연결되어 순차적으로 쌓이는 구조:
```
④ MCP 오픈소스 → 개발자 유입
       ↓
① 벤치마크 콘텐츠 → 대중 유입
       ↓
③ 전자책·강의 → 첫 수익
       ↓
② SaaS 구독 → 안정 수익
       ↓
⑤ B2B → 큰 수익
```

---

## 9. React / PHP 로 다시 만들 수 있나?

### React → 이미 React 기반

`service/frontend/package.json` 확인 결과:
```
React 18.2 + TypeScript 5.2 + Vite 5 + React Router 6 + Recharts 3.8 + ethers 6
```
`ChallengePage.tsx`, `TeamMissionsPage.tsx` 등 화면 파일 존재 → **즉시 수정 가능**

### PHP → 가능하지만 범위 조절 필요

| 항목 | 실측 |
|------|------|
| 백엔드 코드 | 약 21,700줄 |
| DB 테이블 | 48개 |
| API 엔드포인트 | 100개 이상 |

전체 이식은 수개월 규모. 현실적인 선택지 3가지:

**옵션 A — 프론트만 수정** (며칠)
Python 백엔드는 그대로 두고 React 화면만 커스터마이징.

**옵션 B — PHP 미니 버전 신규 구축** (2~3주, 추천)
48개 테이블 전부 불필요. MVP는 6개면 충분:
```
users / agents / positions / trades / signals / follows
```
- Laravel 기준: 인증은 Sanctum, 백그라운드는 Scheduler + Queue
- 시세는 PHP에서 Alpha Vantage 직접 호출
- 기존 React 프론트는 API 형태만 맞추면 재활용 가능

**옵션 C — MCP 서버 구축** (며칠, 실속 최고)
언어 무관. 수백 줄 규모로 AI-Trader API를 감싸면 Claude Code에서 함수 호출처럼 사용 가능.
현재 공개된 구현이 없어 오픈소스로서의 가치도 있음.

---

## 10. 요약표

| 질문 | 결론 |
|------|------|
| 정체 | AI 에이전트용 모의투자 플랫폼 (HKUDS 제작) |
| 설치 | 가입은 curl 한 줄 / 스킬은 `~/.claude/skills/` 에 복사 |
| 분류 | **스킬(마크다운 문서)**. MCP 아님, 플러그인 아님 |
| 토큰 | 필수, 무료. `Authorization: Bearer {token}` |
| 금전 위험 | 없음 (전부 페이퍼 트레이딩) |
| 인기 이유 | 유명 연구실 + 한 줄 온보딩 + 실제 운영 서비스 |
| 학습 가치 | 스킬 라우팅 / 하트비트 / 워커 분리 패턴 |
| 수익화 | AI 벤치마크 콘텐츠 → 교육 → SaaS → B2B 순 |
| React/PHP | React는 이미 적용됨 / PHP는 미니버전 또는 MCP 서버 권장 |

---

## 다음 단계 후보

- [ ] MCP 서버 설계 및 구현 (최우선 추천)
- [ ] 벤치마크 자동화 스크립트 작성
- [ ] 로컬 서버 실제 구동 테스트
- [ ] ai4trade.ai 에이전트 등록 및 리더보드 확인
