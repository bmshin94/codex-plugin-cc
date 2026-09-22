# 📘 codex-plugin-cc 전수조사 & 분석 정리

> 작성일: 2026-09-22
> 작성: 카리나 (Claude Code) ✨
> 대상 저장소: [bmshin94/codex-plugin-cc](https://github.com/bmshin94/codex-plugin-cc)
> 원본 저장소: [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) (OpenAI 공식)

---

## 🔗 관련 링크 모음

| 구분 | 주소 |
|---|---|
| 이 저장소 (내 복사본) | https://github.com/bmshin94/codex-plugin-cc |
| 원본 (OpenAI 공식) | https://github.com/openai/codex-plugin-cc |
| 원본 README | https://github.com/openai/codex-plugin-cc/blob/main/README.md |
| 릴리스 노트 | https://github.com/openai/codex-plugin-cc/releases |
| 역방향 플러그인 (Codex 안에서 Claude 쓰기, 센드버드) | https://github.com/sendbird/cc-plugin-codex |
| Codex 공식 문서 | https://developers.openai.com/codex/ |
| Codex app-server | https://developers.openai.com/codex/app-server |
| Codex 설정 레퍼런스 | https://developers.openai.com/codex/config-reference |
| Codex 요금 정책 | https://developers.openai.com/codex/pricing |
| OpenAI 커뮤니티 소개글 | https://community.openai.com/t/introducing-codex-plugin-for-claude-code/1378186 |
| 뉴스 (The Decoder) | https://the-decoder.com/openai-launches-a-codex-plugin-that-runs-inside-anthropics-claude-code/ |
| 스타 성장 분석 (OSS Insight) | https://ossinsight.io/analyze/openai/codex-plugin-cc |

---

## 1️⃣ 이게 뭐하는 물건인가

### 한 줄 요약
**Claude Code 터미널 안에서 OpenAI Codex(GPT-5.x)를 호출해 쓰게 해주는 OpenAI 공식 플러그인.**
서로 경쟁사인 두 AI 코딩 도구를 잇는 다리(bridge) 역할.

### 기본 정보

| 항목 | 내용 |
|---|---|
| 원저작자 | **OpenAI** (`NOTICE`: Copyright 2026 OpenAI) |
| 패키지명 | `@openai/codex-plugin-cc` |
| 버전 | 1.0.6 |
| 라이선스 | **Apache-2.0** (상업적 사용/수정/재배포 가능) |
| 런타임 | Node.js ≥ 18.18, **런타임 의존성 0개** |
| 총 규모 | 파일 65개 / 스크립트 약 6,000줄 |

### 폴더 구조

```
codex-plugin-cc/
├── .claude-plugin/marketplace.json  # 마켓플레이스 카탈로그
├── plugins/codex/                   # 플러그인 본체
│   ├── .claude-plugin/plugin.json   # 플러그인 매니페스트
│   ├── commands/   (7개)            # 슬래시 커맨드 (사람이 실행)
│   ├── agents/     (1개)            # codex-rescue 서브에이전트
│   ├── skills/     (3개)            # AI 내부 매뉴얼 (user-invocable: false)
│   ├── hooks/hooks.json             # SessionStart / SessionEnd / Stop
│   ├── prompts/    (2개)            # Codex 프롬프트 템플릿
│   ├── schemas/    (1개)            # 리뷰 출력 JSON Schema
│   └── scripts/    (19개, 6000줄)   # 실제 동작 엔진
├── tests/          (9개)            # node --test 스위트
├── scripts/bump-version.mjs         # 버전 동기화
└── .github/workflows/               # PR CI (test + build)
```

### 슬래시 커맨드 8종

| 커맨드 | 기능 | 쓰기 권한 |
|---|---|:---:|
| `/codex:setup` | 설치/로그인 점검, 리뷰 게이트 on/off | — |
| `/codex:review` | 현재 변경분 코드리뷰 | ❌ 읽기전용 |
| `/codex:adversarial-review` | 설계·가정을 공격하는 적대적 리뷰 (포커스 지정 가능) | ❌ 읽기전용 |
| `/codex:rescue` | 작업 위임 (버그조사·수정·후속작업) | ✅ 유일 |
| `/codex:transfer` | 현재 Claude 세션을 Codex 스레드로 이관 | — |
| `/codex:status` | 백그라운드 잡 상태 조회 | — |
| `/codex:result` | 완료된 잡 결과 조회 | — |
| `/codex:cancel` | 실행 중 잡 취소 | — |

### 내부 동작 흐름

```
[사용자] /codex:review
   ↓
[Claude Code] commands/review.md 지시 해석 → git diff 크기 측정 → 포그라운드/백그라운드 제안
   ↓
[Node] scripts/codex-companion.mjs review
   ↓
[브로커] app-server-broker.mjs 를 유닉스 소켓 데몬으로 기동 (요청 멀티플렉싱)
   ↓
[codex app-server] JSON-RPC (turn/start, review/start)
   ↓
[GPT-5.x] 실제 리뷰 수행
   ↓
[schemas/review-output.schema.json] 출력 형식 검증
   ↓ verdict / summary / findings[severity,file,line,confidence] / next_steps
[lib/render.mjs] 마크다운으로 렌더링 → 원문 그대로 사용자에게 반환
   ↓
[lib/state.mjs] 워크스페이스별 해시 폴더에 잡 기록 저장 (최대 50건)
```

### 핵심 설계 포인트

- **브로커(데몬) 패턴** — Codex 프로세스를 매번 재기동하지 않고 유닉스 소켓으로 재사용. BUSY 코드(`-32001`)로 동시성 제어, PID 파일로 생명주기 관리.
- **워크스페이스 격리** — `sha256(realpath(workspaceRoot)).slice(0,16)` 해시로 프로젝트별 상태 디렉터리 분리. 레포 간 작업이 섞이지 않음.
- **출력 스키마 강제** — JSON Schema로 `verdict / findings / next_steps` 구조를 고정. `confidence`(0~1)까지 숫자로 수집.
- **원문 보존 원칙** — Codex 출력을 Claude가 요약/가공하지 못하게 프롬프트로 강하게 제약. "다른 시각"을 얻는 목적이 훼손되지 않도록.
- **자동 수정 금지** — `skills/codex-result-handling/SKILL.md`:
  > *"CRITICAL: After presenting review findings, STOP. Auto-applying fixes from a review is strictly forbidden."*
- **권한 최소화** — 커맨드별 `allowed-tools` 제한, 대부분 `disable-model-invocation: true`, 서브에이전트는 `model: sonnet` + `tools: Bash` 하나만.
- **경로 탈출 방어** — `claude-session-transfer.mjs` 에서 `path.relative()` 로 `~/.claude/projects` 외부 접근 차단.

### 리뷰 게이트 (Stop 훅) ⚠️

`/codex:setup --enable-review-gate` 로 켜면, Claude가 응답을 끝내려는 순간 Codex가 직전 턴의 코드 변경을 검사한다.
응답 첫 줄은 반드시 `ALLOW: <이유>` 또는 `BLOCK: <이유>`.

> ⚠️ **주의**: Claude↔Codex 무한 루프가 생길 수 있고 사용량을 빠르게 소모함. 타임아웃이 900초(15분)로 잡혀 있음. 자리를 비울 때는 반드시 끄기.

---

## 2️⃣ 쉽게 이해하기 (비유)

- **Claude Code** = 내 옆에 붙어서 코드 짜주는 주임 개발자
- **Codex** = 옆 건물에 있는 다른 회사 실력자
- **이 플러그인** = 두 사람을 잇는 전화선 + 통역사

### 왜 AI를 두 개나 쓰나?

1. **자기 숙제는 자기가 채점 못 한다** — 같은 모델이 자기 코드를 리뷰하면 같은 편향으로 같은 실수를 지나친다. 다른 회사 모델은 학습·추론 성향이 달라서 놓친 걸 잡아낸다.
2. **비싼 모델에 잡일 안 시킨다** — 지루한 조사는 `spark` 같은 저가 모델에, 서브에이전트는 `sonnet` 으로. 처음부터 비용 최적화 설계.
3. **기다리지 않는다** — `--background` 로 돌려놓고 다른 작업. 세탁기 돌려놓고 설거지하는 것과 같다.

### 폴더 = 식당 비유

| 폴더 | 비유 |
|---|---|
| `marketplace.json` | 식당 간판 (배달앱 등록 정보) |
| `commands/` | 메뉴판 — 손님이 보고 주문 |
| `agents/` | 서빙 직원 — 주문 전달만, 딴짓 금지 |
| `skills/` | 주방 매뉴얼 — 사람은 못 보고 AI만 읽음 |
| `hooks/` | 자동문 센서 — 사람이 오면 저절로 작동 |
| `schemas/` | 접시 규격 — "결과는 반드시 이 그릇에" |
| `scripts/` | 주방 — 실제 조리 |
| `tests/` | 위생 점검 |

### 실사용 예시

```
/codex:review
  → "12개 파일 500줄 변경. 백그라운드 추천" [기다리기] [백그라운드(추천)]
/codex:status
  → ⏳ task-a3f9 | review | running | 5분 12초 경과
/codex:result
  → 📋 needs-attention
     🔴 critical | src/auth.js:42-48 | confidence 0.92
        토큰 만료 검증 누락 → verifyExp(token) 추가 권장
     🟡 medium   | src/db.js:110    | confidence 0.65
        커넥션 풀 미해제 → finally 블록에 close()
     ❓ 이 중 무엇을 고칠까요?   ← 동의 없이 절대 수정 안 함
```

---

## 3️⃣ 핵심 Q&A

### Q1. 설치 및 사용법

**사전 준비**: Node.js 18.18+ / Claude Code / ChatGPT 계정(무료 포함) 또는 OpenAI API 키

```bash
# 1) 설치
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins

# 내 저장소로 설치할 경우
/plugin marketplace add bmshin94/codex-plugin-cc
# 로컬 개발 시
/plugin marketplace add /경로/codex-plugin-cc

# 2) Codex 준비 확인
/codex:setup
npm install -g @openai/codex   # 수동 설치
!codex login                   # 수동 로그인

# 3) 확인
/help      → /codex:* 커맨드 노출
/agents    → codex:codex-rescue 노출

# 4) 첫 실행
/codex:review --background
/codex:status
/codex:result
```

**주요 옵션**

```bash
/codex:review --base main --scope branch      # 범위: auto|working-tree|branch
/codex:adversarial-review --background 레이스 컨디션 관점으로 봐줘
/codex:rescue --model spark --effort low 빠르게 고쳐줘
/codex:rescue --resume | --fresh              # 이전 스레드 이어가기 / 새로 시작
```

**기본 모델·추론 강도 설정** (`~/.codex/config.toml` 또는 프로젝트 `.codex/config.toml`)

```toml
model = "gpt-5.4-mini"
model_reasoning_effort = "high"   # none|minimal|low|medium|high|xhigh
```
※ 프로젝트 레벨 설정은 **해당 프로젝트가 trusted 상태**여야 로드됨.

슬래시 없이 `"코덱스한테 DB 연결 재설계 맡겨줘"` 처럼 말해도 `codex-rescue` 서브에이전트가 동작함.

---

### Q2. 플러그인? 스킬? MCP?

**정답: 플러그인.** 단, 그 안에 스킬 3개가 포함되어 있음.

```
📦 플러그인 (최상위 패키지)   ← 정답
   ├── 커맨드      7개 ✅
   ├── 서브에이전트 1개 ✅
   ├── 스킬        3개 ✅
   ├── 훅          3개 ✅
   └── MCP 서버    0개 ❌ (사용 안 함)
```

| | 플러그인 | 스킬 | MCP |
|---|---|---|---|
| 정체 | 기능 묶음 패키지 | AI가 읽는 지식 문서 | 외부 도구 연결 프로토콜 |
| 비유 | 택배 상자 | 컨닝페이퍼 | USB 규격 |
| 호출 | 설치 시 커맨드 생성 | AI가 자동 로드 | 도구 목록에 등장 |

**왜 MCP를 안 썼나 (중요한 설계 판단)**

1. **제어권** — 리뷰는 사용자가 원할 때만 실행해야 함. 그래서 `disable-model-invocation: true`.
2. **원문 무결성** — MCP 결과는 모델이 해석·요약함. 이 플러그인은 Codex 출력을 그대로 보여주는 게 목적이라 `!`백틱 커맨드 방식이 필요.
3. **백그라운드 잡** — MCP는 1회 요청-응답 구조라 장시간 잡 관리에 부적합. 자체 잡 시스템 + 브로커로 해결.
4. **권한 세분화** — 커맨드별 `allowed-tools` 지정이 가능.

> 📌 판단 기준: **AI가 알아서 호출**해야 하면 MCP/스킬, **사람이 명시적으로 발동**하고 결과 원문이 중요하면 커맨드/플러그인.

---

### Q3. API 토큰이 필요한가

**아니요. ChatGPT 계정(무료 플랜 포함)만 있어도 됨.**

- 인증 방식 ① ChatGPT 로그인 (`codex login`) — 구독에 포함된 Codex 사용량 소모
- 인증 방식 ② OpenAI API 키 — 종량 과금

**핵심 사실**

1. **플러그인은 인증을 직접 다루지 않음.** 코드 어디에도 API 키를 읽거나 저장하는 부분이 없음. 로컬 `codex` 바이너리에 위임하고 `getCodexAuthStatus()` 로 상태만 조회. 자격증명은 전부 `~/.codex/` 에서 Codex CLI가 관리. → 보안상 깔끔한 설계.
2. 이미 Codex에 로그인돼 있으면 **추가 로그인 불필요.**
3. **비용은 이중 소모**: Claude 토큰(커맨드 처리·출력) + Codex 사용량(실제 작업). README 명시: *"Usage will contribute to your Codex usage limits."*
4. 사내 게이트웨이는 `openai_base_url` 로 변경 가능.

**절감 팁**: `--model spark` / `--effort low` / 리뷰 범위 축소 / 리뷰 게이트는 평소 OFF.

---

### Q4. 왜 GitHub에서 유명한가

| 시점 | 스타 |
|---|---|
| 2026-03 출시 당일 | 3,700+ |
| 2026-07 | 25,900+ |

**이유 5가지**

1. **경쟁사 제품 안에서 돌아가는 공식 도구** — 애플이 갤럭시용 앱을 공식 출시한 격. 뉴스 가치가 압도적.
2. **전략적으로 영리함** — Claude Code 사용자는 고관여 개발자층. "쓰던 자리에서 눌러보세요"는 진입장벽이 거의 없음. 적진에 세운 영업소.
3. **실수요 충족** — 개발자들은 이미 여러 AI를 번갈아 쓰며 복붙 중이었고, 그 불편을 공식적으로 해결.
4. **코드 품질** — 런타임 의존성 0, 테스트 스위트, CI, 브로커 패턴, 스키마 검증. "Claude Code 플러그인 작성 교과서"로 인용됨.
5. **생태계 확산** — 역방향 플러그인 `sendbird/cc-plugin-codex` (Codex 안에서 Claude 실행) 등장. 양방향으로 열린 생태계.

---

### Q5. 로컬 에이전트 구축에 도움이 되나

**된다. 단, "복붙 라이브러리"가 아니라 "설계 교본"으로.**

**훔쳐올 패턴 7가지**

| # | 패턴 | 파일 | 요지 |
|---|---|---|---|
| ① | 브로커(데몬) | `app-server-broker.mjs` | 무거운 프로세스를 1개만 띄우고 소켓 멀티플렉싱 |
| ② | 잡 추적 | `tracked-jobs.mjs`, `job-control.mjs` | queued→running→completed 상태머신, 로그/결과 분리 |
| ③ | 워크스페이스 격리 | `state.mjs` | `realpath` 정규화 후 sha256 해시로 디렉터리 분리 |
| ④ | 출력 스키마 강제 | `schemas/*.json` | JSON Schema + `parseStructuredOutput` fallback |
| ⑤ | 경로 탈출 방어 | `claude-session-transfer.mjs` | `path.relative()` 로 디렉터리 탈출 차단 |
| ⑥ | 에이전트 역할 좁히기 | `agents/codex-rescue.md` | 싼 모델 + 도구 1개 + "딴짓 금지" |
| ⑦ | 프롬프트 블록 표준 | `skills/gpt-5-4-prompting/` | XML 태그 블록 구조 + 안티패턴 문서 |

**한계**

- `${CLAUDE_PLUGIN_ROOT}`, `hooks.json`, 슬래시 커맨드 등 **Claude Code 종속** — 독립 실행형 에이전트로 전용 불가
- 로컬 `codex` 바이너리 **필수** — 다른 모델로 바꾸려면 `lib/codex.mjs`(1,219줄) 재작성 필요
- `"private": true` — npm 배포 안 됨, `import` 불가
- 단일 머신·단일 체크아웃 전제 (분산/원격 미고려)

**로드맵 제안**: ① 브로커·잡·상태 코드 정독(1~2일) → ② Ollama 브로커 직접 구현(1주) → ③ 프롬프트 표준·스키마 적용 + 웹 대시보드(2~4주)

---

### Q6. 수익화 아이디어 → 4장 참고

### Q7. React나 PHP로 만들 수 있나

**플러그인 본체는 Node.js로 가야 함.**

- 훅이 `node ...` 프로세스를 직접 실행하고, SessionStart 훅 타임아웃이 5초 → PHP-FPM 기동 등에 부적합
- Codex app-server와 **stdio JSON-RPC** 통신
- **React는 근본적으로 불가** — 브라우저 UI 라이브러리, 터미널엔 DOM이 없음
- **PHP는 기술적으론 가능하나 비추천** — 사용자에게 PHP 설치 강요, 생태계 예제 없음, 비동기/스트리밍 처리가 번거로움

**그런데 React/PHP가 빛나는 자리가 따로 있음.** 이 플러그인의 최대 약점은 **UI가 없다는 것**.

```
[터미널] Claude Code + codex 플러그인
    ↓ 결과를 JSON으로 저장 (state.json / jobs/*.json)
[🐘 PHP(Laravel)] 수집·DB 적재·REST API·팀 계정/권한/과금
    ↓
[⚛️ React] 실시간 잡 모니터, findings 시각화, 품질 트렌드, 비용 비교
```

- **React로**: 리뷰 대시보드 / 잡 모니터(SSE) / 비용 트래커 / VS Code 확장(웹뷰) / **프롬프트 빌더**
- **PHP로**: 팀 집계 서버 / GitHub 웹훅 수신 / 주간 PDF 리포트 / 워드프레스 플러그인

**권장 조합**: 본체는 Node.js 최소한으로, 가치는 웹(React+PHP)에서.

---

## 4️⃣ 수익화 아이디어

### ⚖️ 법적 기반 (Apache-2.0)

| 가능 ✅ | 의무 ⚠️ | 금지 ❌ |
|---|---|---|
| 상업적 판매, 수정·재배포, 클로즈드 전환, SaaS 제공 | LICENSE 동봉, NOTICE 유지, 변경 파일 고지 | **OpenAI/Codex 상표 사용**, 공식 사칭 |

> 🚨 Apache-2.0 **제6조는 상표권을 명시적으로 제외**. 코드는 써도 되지만 "OpenAI Codex Pro" 같은 이름은 불가. 반드시 독자 브랜드로.

### 아이디어 요약표

| # | 아이디어 | 난이도 | 기간 | 초기비용 | 추천도 |
|---|---|:---:|:---:|:---:|:---:|
| 1 | **멀티 AI 크로스 리뷰 SaaS** | 🔥🔥🔥 | 6~8주 | 낮음 | ⭐⭐⭐⭐⭐ |
| 2 | **한국형 AI 개발 워크플로 패키지** | 🔥🔥 | 3~4주 | 0원 | ⭐⭐⭐⭐⭐ |
| 3 | **프롬프트 빌더 SaaS** | 🔥🔥 | 3~4주 | 0원 | ⭐⭐⭐⭐ |
| 4 | CI/CD 리뷰 게이트웨이 | 🔥🔥🔥🔥 | 3~4개월 | 중 | ⭐⭐⭐ |
| 5 | 로컬 에이전트 오케스트레이터 | 🔥🔥🔥🔥🔥 | 6개월+ | 높음 | ⭐⭐⭐ |
| 6 | AI 비용 최적화 라우터 | 🔥🔥🔥 | 2~3개월 | 낮음 | ⭐⭐⭐ |
| 7 | **교육 콘텐츠 (강의·전자책)** | 🔥 | 2~4주 | 0원 | ⭐⭐⭐⭐⭐ |
| 8 | 니치 플러그인 팩 (WP/React/한국) | 🔥🔥 | 2~3주 | 0원 | ⭐⭐⭐⭐ |
| 9 | 리뷰 리포트 자동화 (PDF) | 🔥🔥 | 3주 | 낮음 | ⭐⭐⭐ |

### 💎 1. 멀티 AI 크로스 리뷰 SaaS (최우선 추천)

**컨셉**: *"당신의 코드를 GPT·Claude·Gemini 3개가 동시에 리뷰합니다"*

- 공식 플러그인은 Claude↔Codex **1:1, 로컬, 터미널 전용**. 여기를 **N:N, 웹, 대시보드**로 확장.
- **킬러 기능 = 합의(Consensus)**: AI 하나의 지적은 환각일 수 있지만, 3개 중 2개가 같은 지점을 지적하면 신뢰도가 급상승. 이 레포 스키마의 `confidence` 필드로 **가중 점수** 계산 가능.
- 아키텍처: GitHub 웹훅 → PHP(Laravel) 오케스트레이터 → 3개 모델 병렬 호출 → JSON Schema로 정규화 → React 대시보드
- 가격: Free(월 20회) / Pro $19 / Team $49·인 / Enterprise 협의
- **BYOK(Bring Your Own Key)** 로 가면 API 원가가 0 — 오케스트레이션과 UI만 판매

### 💎 2. 한국형 AI 개발 워크플로 패키지

**컨셉**: *"한국 개발팀을 위한 AI 코드리뷰 올인원"*

- 공식 플러그인은 문서도 리뷰 결과도 전부 영어. 한국 기업은 **한글 리포트**가 절실.
- 구성: 한글 리뷰 프롬프트팩 / 개인정보보호법·전자금융감독규정 체크 룰셋 / 주간 품질 PDF 리포트 / 도입 교육
- 수익: 패키지 ₩200~500만(기업당) + 유지보수 월 ₩50만 + 교육 일 ₩150만
- **코드는 거의 안 건드리고 `prompts/`·`skills/` 만 한국 맞춤화 → 가성비 최고**

### 💎 3. 프롬프트 빌더 SaaS

`skills/gpt-5-4-prompting/references/prompt-blocks.md` 가 숨은 보석.
`<task>` `<structured_output_contract>` `<verification_loop>` `<grounding_rules>` `<completeness_contract>` `<action_safety>` `<citation_rules>` `<research_mode>` `<missing_context_gating>` `<dig_deeper_nudge>` — **OpenAI가 직접 검증한 프롬프트 블록 표준**인데 텍스트 문서로만 존재.

→ React로 드래그앤드롭 조립 UI + 3개 모델 A/B 테스트. Free / Pro $9 / Team $29·인.
**백엔드가 거의 필요 없어 React 실력만으로 가장 빨리 출시 가능.**

### 그 외

- **4. CI/CD 리뷰 게이트웨이** — 로컬 Stop 훅은 무한루프 위험이 있지만 CI는 1회성이라 안전. 경쟁자(CodeRabbit, Greptile) 존재하나 **멀티모델 합의**는 빈칸. 저장소당 $10~30/월.
- **5. 로컬 에이전트 오케스트레이터** — 브로커 패턴 범용화(Ollama/사내 LLM 어댑터). 외부 API를 못 쓰는 금융·공공 시장. 온프레미스 $5,000~/년.
- **6. AI 비용 최적화 라우터** — 작업 난이도 자동 판정 후 저가/중가/고가 모델 라우팅. 품질 유지하며 비용 60~70% 절감 목표.
- **7. 교육 콘텐츠** — 2.5만 스타인데 **한글 해설이 거의 없음**. 선점 타이밍. 강의 ₩55,000 / 전자책 ₩25,000 / 출강 ₩150만·일. **초기비용 0, 가장 빠른 현금화.**
- **8. 니치 플러그인 팩** — 워드프레스 팩(PHP 강점) / React 팩 / 한국 스타트업 팩. 무료 배포 → 인지도 → 컨설팅·강의 전환(리드 제너레이션).
- **9. 리포트 자동화** — `jobs/*.json` 수집 → PHP로 주간 PDF 생성·발송. 한국 기업 보고서 문화와 궁합. 팀당 $29/월.

### 🎯 권장 3단계 로드맵

```
1단계 (0~1개월) 씨앗 뿌리기 — 비용 0원
   ├─ 한글 해설 블로그/영상 (아이디어 7)
   └─ 워드프레스 팩 무료 배포 (아이디어 8, PHP 강점)
   목표: 인지도 + 피드백

2단계 (1~3개월) 첫 수익 — React 실력 발휘
   ├─ 프롬프트 빌더 출시 $9/월 (아이디어 3)
   └─ 한국형 패키지 B2B 첫 계약 (아이디어 2)
   목표: MRR $1,000

3단계 (3~9개월) 본게임
   ├─ 멀티 AI 크로스 리뷰 SaaS (아이디어 1)
   └─ BYOK로 원가 최소화
   목표: MRR $10,000
```

### ⚠️ 리스크

1. **OpenAI/Anthropic이 직접 구현할 위험** → 방어막은 **멀티모델 중립성**. 각 벤더는 경쟁사 모델을 동등하게 대접하지 않음.
2. **API 가격 변동** → BYOK로 전가.
3. **플랫폼 정책 변경** → 특정 CLI 종속을 낮추고 웹 중심으로.
4. **상표권** → 반드시 독자 브랜드.

---

## 📌 이 저장소 관련 참고 사항

- 최신 커밋 `64c50de "Add files via upload"` 는 웹 업로드로 생성된 것이며, 그 이전 히스토리(#35~#447)는 원본 오픈소스 히스토리 그대로 보존되어 있음.
- 루트의 `CLAUDE.md` / `GEMINI.md` 는 원본에 없던 파일로, 현재 개인 페르소나 설정으로 채워져 있음. **원본의 개발 가이드 내용은 들어있지 않음.** 원본 기여 규칙이 필요하면 upstream에서 다시 받아야 함.
- 업스트림 동기화:
  ```bash
  git remote add upstream https://github.com/openai/codex-plugin-cc
  git fetch upstream
  git merge upstream/main
  ```

---

*본 문서는 저장소 전체(65개 파일)를 직접 읽고 작성한 분석 결과입니다.* ✨
