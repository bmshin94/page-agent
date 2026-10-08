# Page Agent 전수조사 분석 및 활용·수익화 정리 (한국어)

> 이 문서는 `page-agent` 저장소를 전수조사하여 **무엇을 하는 프로젝트인지, 어떻게 쓰는지, 어떤 가치가 있는지, 어떻게 수익화할 수 있는지**를 한국어로 정리한 자료입니다.
>
> - **작성일**: 2026-10-08
> - **분석 대상 버전**: `1.12.4`
> - **분석 커밋**: `5ebad72`

## 🔗 GitHub 주소

| 구분 | 주소 |
| --- | --- |
| **이 저장소 (포크)** | https://github.com/bmshin94/page-agent |
| **원본 저장소 (업스트림)** | https://github.com/alibaba/page-agent |
| 공식 문서 / 데모 | https://alibaba.github.io/page-agent/ |
| npm 패키지 (본체) | https://www.npmjs.com/package/page-agent |
| npm 패키지 (코어) | https://www.npmjs.com/package/@page-agent/core |
| npm 패키지 (MCP 서버) | https://www.npmjs.com/package/@page-agent/mcp |
| 크롬 확장 | https://chromewebstore.google.com/detail/page-agent-ext/akldabonmimlicnjlflnapfeklbfemhj |
| Hacker News 토론 | https://news.ycombinator.com/item?id=47264138 |
| 기반 프로젝트 (browser-use) | https://github.com/browser-use/browser-use |

**라이선스**: MIT — 상업적 이용 / 수정 / 재배포 / 클로즈드 소스 SaaS 판매 모두 허용. 의무는 저작권 표시 + 라이선스 사본 포함뿐.

---

## 목차

1. [한 줄 요약](#1-한-줄-요약)
2. [핵심 발상 — 스크린샷을 안 찍는다](#2-핵심-발상--스크린샷을-안-찍는다)
3. [저장소 구조 전수조사](#3-저장소-구조-전수조사)
4. [에이전트 동작 원리](#4-에이전트-동작-원리)
5. [지원 모델](#5-지원-모델)
6. [할 수 있는 것 / 못 하는 것](#6-할-수-있는-것--못-하는-것)
7. [보안 설계와 남는 리스크](#7-보안-설계와-남는-리스크)
8. [설치 및 사용법](#8-설치-및-사용법)
9. [플러그인? 스킬? MCP?](#9-플러그인-스킬-mcp)
10. [API 토큰 필요 여부](#10-api-토큰-필요-여부)
11. [AI 에이전트 구축에 도움이 되는가](#11-ai-에이전트-구축에-도움이-되는가)
12. [React / PHP 로 만들 수 있는가](#12-react--php-로-만들-수-있는가)
13. [유튜브 강의 제작 가능성](#13-유튜브-강의-제작-가능성)
14. [수익화 아이디어 10가지](#14-수익화-아이디어-10가지)
15. [최종 권고 로드맵](#15-최종-권고-로드맵)

---

## 1. 한 줄 요약

> **"웹페이지 안에 사는 AI 비서를 심어주는 자바스크립트 라이브러리"**

`<script>` 태그 한 줄(또는 `npm install page-agent`)만 넣으면, 그 웹페이지에 채팅창이 생기고 → 사용자가 자연어로 말하면 → AI가 그 페이지의 버튼을 직접 클릭하고 입력창을 채워줍니다.

- 알리바바가 공개한 오픈소스 (MIT)
- `browser-use`의 DOM 처리/프롬프트를 가져와 **브라우저 안에서 돌도록** 재설계
- 이 저장소는 원본의 포크이며, 포크 후 변경점은 `CLAUDE.md` 문서 1개 추가(PR #1)뿐. 코드는 원본과 동일

### 장면으로 이해하기

쇼핑몰 관리자에서 반품 100건 처리:

- **전**: 주문검색 → 클릭 → 상세 → 반품버튼 → 사유선택 → 메모입력 → 저장 → 뒤로 ... (×100) = **클릭 800회, 2시간**
- **후**: 채팅창에 *"반품 대기 목록 전부, 사유는 단순변심, 메모는 고객요청으로 처리해줘"* → **커피 마시러 간다**

---

## 2. 핵심 발상 — 스크린샷을 안 찍는다

AI에게 페이지를 알려주는 방식이 일반 도구와 근본적으로 다릅니다.

| | 스크린샷 방식 (browser-use, Operator 등) | **Page Agent (DOM 텍스트 방식)** |
| --- | --- | --- |
| 입력 | 화면 이미지 | DOM을 번호 붙인 텍스트로 변환 |
| 필요 모델 | Vision(멀티모달) 필수 | **아무 텍스트 모델이나** |
| 비용 | 💸💸💸 이미지 토큰 | 💸 텍스트만 — 훨씬 저렴 |
| 정확도 | 좌표 틀릴 수 있음 | **번호는 틀릴 수 없음** |
| 설치 | 파이썬 + 헤드리스 브라우저 | **스크립트 1줄** |
| 창 크기 변경 | 깨짐 | 무관 |
| 로컬 모델 | 어려움 | **가능 (API 비용 0원)** |
| 로그인 세션 | 다시 맞춰야 함 | 사용자 브라우저 그대로 |

### AI에게 실제로 전달되는 데이터

```text
Current URL: https://myshop.com/admin/orders

[1]<input placeholder='주문번호 검색'></input>
[2]<button>검색</button>
[3]<select>상태 필터</select>
[4]<button>반품 처리</button>
주문 #10024 · 김철수 · 2026-10-07     ← 번호 없음 = 클릭 불가, 그냥 글자
	*[5]<div>자동완성 목록</div>        ← * = 직전 단계 이후 새로 생긴 요소
```

- `[n]` 번호가 붙은 것만 조작 가능
- `\t` 들여쓰기 = DOM 부모·자식 관계
- `*[n]` = **변화 감지(delta)**. "뭐가 바뀌었는지"를 저렴하게 알려주는 장치
- AI 응답: `{"action": {"click_element_by_index": {"index": 4}}}` → **"4번 눌러"**

> 비유: 스크린샷 방식 = "사진 보고 설명해주는 사람에게 전화로 지시", Page Agent = "번호 붙은 리모컨 버튼 누르기"

이 "DOM → 번호 붙은 텍스트" 변환 엔진이 프로젝트의 심장이며 `packages/page-controller/src/dom/dom_tree/index.js`에 **1,745줄**로 구현돼 있습니다.

---

## 3. 저장소 구조 전수조사

npm workspaces 모노레포, 8개 패키지. `package.json`의 `workspaces`는 **의존 위상 순서**로 정렬돼 있습니다.

```text
page-agent/
├── packages/
│   ├── page-controller/   ① 손과 눈  (DOM 조작)       3,969줄
│   ├── ui/                ② 얼굴     (채팅 패널)       1,028줄
│   ├── llms/              ③ 입       (LLM 통신)        1,571줄
│   ├── core/              ④ 뇌       (에이전트 루프)    1,878줄
│   ├── page-agent/        ⑤ 완제품   (npm: page-agent)    93줄
│   ├── mcp/               ⑥ MCP 서버 (베타)            3파일
│   ├── extension/         ⑦ 크롬 확장 (멀티탭)         5,058줄
│   └── website/           ⑧ 공식 문서 사이트           8,679줄
├── docs/                  체인지로그, 보안정책, 중국어 README, agentic-testing
├── scripts/               빌드/배포/CI (pre-publish, post-publish, sync-version 등)
├── .agents/skills/        ⭐ AI 코딩도구용 스킬 5개
└── .github/workflows/     CI 4개 (ci, main-ci, release, deploy-website)
```

### 몸으로 비유하면

| 패키지 | 비유 | 역할 |
| --- | --- | --- |
| `page-controller` | **손 + 눈** | 페이지 읽고 번호 붙이고 실제 클릭/타이핑. LLM을 전혀 모름 |
| `ui` | **얼굴 + 입** | 우하단 채팅 패널. 바닐라 JS (React 의존성 없음 → 어떤 사이트에도 충돌 없이 주입) |
| `llms` | **전화기** | AI 회사에 전화 걸고, 회사별로 다른 말투를 통역 |
| `core` | **뇌** | "읽고→생각하고→움직이고" 반복 (최대 40스텝) |
| `page-agent` | **완성된 사람** | 위 4개를 조립. 조립 코드가 93줄뿐 |
| `mcp` | **외부 리모컨 수신기** | Claude Desktop이 내 브라우저를 조종 |
| `extension` | **다리** | 여러 탭을 넘나듦 (선택 사항) |
| `website` | **설명서** | 공식 문서 |

### ① `page-controller` — 손과 눈 (가장 큼)

| 파일 | 하는 일 |
| --- | --- |
| `dom/dom_tree/index.js` (1,745줄) | DOM 전체를 훑어 클릭 가능한 요소를 찾아 번호 매김. 가려진/투명/화면 밖 요소 필터링 |
| `dom/index.ts` (569줄) | 결과를 AI가 읽을 텍스트로 압축 (dehydration) |
| `actions.ts` | 실제 클릭/입력/스크롤/select. **단순 `.click()`이 아니라** 포인터 이동 → mousedown → mouseup → 네이티브 value setter 호출 |
| `mask/SimulatorMask.ts` | 작업 중 화면을 덮는 투명 오버레이 + 가상 마우스 커서 애니메이션 |
| `mask/checkDarkMode.ts` | 사이트 다크모드 감지 |
| `patches/react.ts`, `patches/antd.ts` | **React / Ant Design 전용 우회 패치** |

> `patches/` 폴더의 존재가 실전성을 증명합니다. React는 자체 Synthetic Event 시스템 때문에 일반 input 이벤트를 무시하는데, 이를 뚫는 코드가 들어 있습니다. 실제 기업 어드민(대부분 React + antd)에서 돌리려고 만든 티가 납니다.

### ② `ui` — 얼굴

- `panel/Panel.ts` — 떠있는 채팅 패널. 상태(`thinking`/`executing`/`retrying`/`error`), 단계별 히스토리, 중단 버튼
- `i18n/locales.ts` — **영어(en-US) / 중국어(zh-CN) 2개뿐. 한국어 없음** → 기여 포인트
- `PanelAgentAdapter` 인터페이스로 분리 → 패널만 떼서 다른 에이전트에 붙일 수 있음

### ③ `llms` — 입 (유일하게 테스트 완비)

- `OpenAIClient.ts` — OpenAI 호환 클라이언트
- 모델별 호환 패치 내장 (GPT / Claude / Qwen / Gemini / DeepSeek가 각자 거부하는 파라미터 자동 조정)
- 재시도 로직 + `transformRequestBody`로 직접 요청 조작
- 테스트: 일반 테스트 + `*.live.test.ts`(실제 API 호출, CI 제외) 분리

### ④ `core` — 뇌

`PageAgentCore.ts` (661줄)가 ReAct 루프를 돌립니다. **AI에게 주는 도구는 딱 9개**:

| 도구 | 설명 |
| --- | --- |
| `click_element_by_index` | 번호로 클릭 |
| `input_text` | 번호에 텍스트 입력 |
| `select_dropdown_option` | 드롭다운 선택 |
| `scroll` / `scroll_horizontally` | 세로/가로 스크롤 (특정 컨테이너 지정 가능) |
| `wait` | 대기 (LLM 응답 시간을 뺀 실제 대기시간 계산) |
| `ask_user` | **사용자에게 되묻기** ← 중요 |
| `done` | 작업 종료 + 최종 답변 |
| `execute_javascript` | 임의 JS 실행 (**기본 OFF**, opt-in) |

도구를 의도적으로 적게 유지합니다 — 선택지가 많으면 AI가 헷갈립니다. 코드에 솔직한 주석까지 있습니다:

```js
/**
 * @todo Tables need a dedicated parser to extract structured data. This tool is useless.
 */
tools.set('scroll_horizontally', ...)
```

확장 포인트 (`core/src/types.ts`):

| 옵션 | 용도 |
| --- | --- |
| `customTools` | 내 도구 추가 / 기존 도구 덮어쓰기 / `null`로 삭제 |
| `instructions.system` | 전역 시스템 지침 |
| `instructions.getPageInstructions(url)` | **URL별 동적 지침 주입** |
| `transformPageContent` | AI 전송 직전 내용 가공 → **개인정보 마스킹** |
| `onBeforeStep` / `onAfterStep` / `onBeforeTask` / `onAfterTask` / `onDispose` | 생명주기 훅 |
| `customSystemPrompt` | 시스템 프롬프트 완전 교체 |
| `experimentalScriptExecutionTool` | JS 실행 도구 활성화 (위험) |
| `experimentalLlmsTxt` | 사이트의 `/llms.txt`를 컨텍스트로 활용 |
| `maxSteps` (기본 40), `stepDelay` (기본 0.4초) | 루프 제어 |

### ⑤ `page-agent` — 완제품

`core` + `ui` + `page-controller` 조립만 (93줄). `demo.ts`는 CDN용 IIFE 빌드로, 스크립트 URL 쿼리스트링(`?model=`, `?baseURL=`, `?apiKey=`, `?lang=`, `?showPanel=`, `?autoInit=`)으로 설정 가능합니다.

### ⑥ `mcp` — MCP 서버 (베타)

```text
Claude Desktop ──stdio(MCP)──► @page-agent/mcp (Node)
                                    │ WebSocket (localhost:38401)
                                    ▼
                              크롬 확장의 Hub 탭 ──► 실제 브라우저 조작
```

MCP 도구 3개: `execute_task(task)`, `get_status()`, `stop_task()`
빌드 없는 순수 ESM 3파일(`index.js`, `hub-bridge.js`, `launcher.html`) — 소스가 그대로 배포물.

### ⑦ `extension` — 크롬 확장

- `MultiPageAgent.ts` — 여러 탭을 넘나드는 에이전트
- `TabsController.ts` — 탭 열기/전환/닫기 (MV3 서비스워커가 죽어도 살아남는 stateless 설계)
- `RemotePageController` — background ↔ content script ↔ main-world 3단 메시지 브리지
- `entrypoints/hub/` — MCP 서버와 WebSocket으로 붙는 허브 탭
- 사이드패널 UI(React + shadcn/ui), IndexedDB 작업 이력 저장/내보내기
- 권한: `tabs`, `tabGroups`, `sidePanel`, `storage`, `<all_urls>`
- **토큰 인증**: 웹페이지 JS가 확장을 호출하려면 사용자가 사이드패널에서 토큰을 복사해 `localStorage`에 설정
- 안전장치: MultiPageAgent에서는 `execute_javascript` **강제 비활성화**

### ⑧ `website` — 공식 문서

React + Vite + Tailwind + magic-ui. 문서 구조: overview / quick-start / limitations / troubleshooting / models / chrome-extension / mcp-server / data-masking / custom-tools / custom-instructions / third-party-agent / security-permissions / page-agent / page-agent-core

### 🎁 보너스: `.agents/skills/` — AI 코딩도구용 스킬 5개

이 저장소는 **자기 자신을 AI가 개발하도록** 스킬을 품고 있습니다.

| 스킬 | 용도 |
| --- | --- |
| `maintain-model-list` | 지원 모델 목록을 3개 파일에서 동기화 |
| `pre-impl-discussion` | 큰 변경 전 반드시 토론 먼저 |
| `submit-pr-from-current-changes` | 현재 변경사항 → 브랜치/커밋/PR 자동화 |
| `update-changelog` | git 히스토리에서 체인지로그 생성 |
| `git-cleanup` | 로컬 브랜치/리모트 정리 |

> Claude Code를 쓴다면 이 스킬들을 참고/복사하는 것만으로도 가치가 있습니다.

---

## 4. 에이전트 동작 원리

### ReAct 루프

```text
관찰(DOM 읽기) → 사고(LLM 호출) → 행동(DOM 조작) → 반복
└ 최대 40스텝 (maxSteps 기본값), 스텝 간 0.4초 딜레이
```

### ⭐ MacroTool 패턴 — 가장 베낄 만한 설계

AI가 아무 말이나 못 하게 응답 스키마를 강제합니다. 매 단계마다 **반드시** 4칸을 채워야 합니다.

```json
{
  "evaluation_previous_goal": "로그인 버튼을 눌렀고 폼이 떴다. 판정: 성공",
  "memory": "100건 중 23건 처리 완료",
  "next_goal": "이메일 칸에 주소를 입력한다",
  "action": { "input_text": { "index": 13, "text": "a@b.com" } }
}
```

**왜 중요한가?** 그냥 "클릭해"만 시키면 이런 사고가 납니다.

- 🤖 "저장 버튼 누름" → (안 눌림) → "저장 버튼 누름" → ... **무한 반복**
- 🤖 100건 중 몇 건 했는지 **망각**

`evaluation_previous_goal`을 강제로 쓰게 만들면 AI가 **"아까 눌렀는데 화면이 안 변했네 → 실패 → 다른 방법"** 을 스스로 깨닫습니다. `memory`는 메모장 역할로 진행 상황을 유지합니다.

공식 명칭은 **"행동 전에 반성하기(reflection-before-action)"** 이고, **웹 자동화와 무관하게 모든 에이전트에 그대로 적용 가능한 패턴**입니다.

### 두 종류 정보 스트림 분리 (실무 핵심)

```text
History Events  → AI 컨텍스트에 포함 (영구 기억)
                  types: step / observation / user_takeover / retry / error
Activity Events → UI에만 표시 (일회성)
                  types: thinking / executing / executed / retrying / error
```

섞으면 토큰이 터지거나 UI가 멈춥니다. 이 분리는 바로 가져다 쓸 설계입니다.

### 이벤트 4종 (`EventTarget` 기반)

`statuschange` / `historychange` / `activity` / `dispose`
상태: `idle` → `running` → `completed` | `error` | `stopped`

### 중단(Abort) 신호 전파 — 과소평가된 난제

```text
사용자가 Stop 누름
→ AbortController.abort()
→ LLM fetch 취소
→ 실행 중인 도구에 ctx.signal 전달
→ 비동기 콜백까지 전파
→ stop()이 "완전히 멈춘 뒤에야" resolve
```

체인지로그에 *"Rewrote the aborting system"* 이라고 **한 번 다시 썼다**고 적혀 있습니다. 직접 만들어 보면 난이도를 알 수 있습니다.

### 설계 철학 (AGENTS.md / 시스템 프롬프트에 명문화)

> **"추적가능성과 예측가능성이 성공률보다 중요하다."**
> **"에러나 위험을 숨기려 하지 마라. 그건 개발자와 사용자에게 가치 있는 피드백이다."**

시스템 프롬프트 실제 내용(번역):

- *"실패해도 괜찮다."*
- *"너무 열심히 하면 해로울 수 있다. 같은 행동을 왔다갔다 반복하거나, 잘 모르는 복잡한 절차를 밀어붙이면 원치 않는 결과와 해로운 부작용이 생긴다. 사용자는 네가 무리하는 것보다 실패로 끝내는 걸 선호한다."*
- *"사용자가 틀릴 수도 있다. 요청이 불가능하거나 부적절하면, 더 나은 요청을 해달라고 말해라."*
- *"웹페이지는 고장나 있을 수 있다. 네 피드백(실패 포함)은 사용자에게 가치가 있다."*
- *"CAPTCHA가 나오면 못 푼다고 사용자에게 말하고 종료하라."*

→ **폭주하지 않는 에이전트**를 지향합니다.

---

## 5. 지원 모델

`packages/website/src/pages/docs/features/models/page.tsx`에서 확인한 **테스트 완료 모델 58개**:

| 계열 | 모델 |
| --- | --- |
| **Claude** | opus-5, fable-5 / fable-5-1, sonnet-5, opus-4-5 ~ 4-8, sonnet-4-5, haiku-4-5 |
| **GPT** | gpt-6-astra, gpt-5.6-luna/sol/terra, 5.5, 5.4(+mini/nano), 5.2, 5.1, 5, 5-mini, 4.1(+mini) |
| **Gemini** | 3.8-flash, 3.7-flash, 3.6-flash, 3.5-flash(+lite), 3.1-pro, 3.1-flash-lite, 2.5-pro/flash |
| **Qwen** | qwen3.6-max/flash, qwen3.5-plus/flash, qwen3-max, qwen3-coder-next |
| **DeepSeek** | v4-pro, v4-flash, v4-flash-vision-exp, 3.2 |
| **GLM** | 5.3(+flash), 5.2, 5.1, 5, 4.7 |
| **Kimi** | k3, k2.7-code, k2.6, k2.5 |

**OpenAI 호환 API면 전부 동작** → Ollama / vLLM / LM Studio 같은 **로컬 모델도 가능** (= API 비용 0원 + 데이터 외부 유출 0).

> 실서비스 팁: `qwen3.5-flash`, `gemini-3.8-flash`, `gpt-5.4-nano`, `glm-5.3-flash` 같은 **저가 플래시급**으로 충분한 경우가 많습니다. 어려운 작업만 상위 모델로 올리는 2단 라우팅이 비용 효율적입니다.

---

## 6. 할 수 있는 것 / 못 하는 것

### ✅ 가능

클릭, 텍스트 입력, select 선택, 세로/가로 스크롤, 폼 제출, 포커스, **동일 출처 iframe(1단계만)**, JS 실행(opt-in)

### ❌ 불가능

| 못 하는 것 | 이유 |
| --- | --- |
| 🎨 **canvas 기반 그림/지도/차트 조작** | DOM에 안 나타남. 설계도엔 "여기 그림 한 장"만 적혀 있음 |
| 🖐️ **호버, 드래그&드롭, 우클릭** | 미지원 (의도적 설계) |
| ⌨️ **키보드 단축키** | 미지원 |
| 📍 **좌표 기반 조작** | DOM 기반이라 불가 |
| 🖼️ **중첩 iframe, 크로스 오리진 iframe** | 접근 제한 |
| 🤖 **CAPTCHA** | 애초에 못 풀게 설계 |
| 🔗 **다른 페이지/사이트로 이동** | in-page 버전은 SPA 전용. `<a target="_blank">` 클릭 금지가 프롬프트에 박힘 → **확장 버전**으로 해결 |

### 🎯 적합 / 부적합

- **최적**: ERP, CRM, 어드민, 그룹웨어, 예약 시스템, 쇼핑몰 관리자, 폼이 많은 사이트 (= 글자와 버튼으로 이루어진 업무용 웹)
- **부적합**: 지도 서비스, 그래픽 툴, 게임 (= 그림과 좌표로 이루어진 화면)

### 두 가지 버전 비교

| | **① 라이브러리 버전** (script 1줄) | **② 크롬 확장 버전** |
| --- | --- | --- |
| 누가 설치? | **웹사이트 개발자**가 자기 사이트에 심음 | **사용자**가 자기 브라우저에 설치 |
| 작업 범위 | 그 페이지 하나 (SPA 내부) | **모든 웹사이트, 여러 탭** |
| 탭 조작 | ❌ | ✅ 열기/전환/닫기 |
| 용도 | "내 서비스에 AI 비서 탑재" | "내가 여러 사이트 돌며 일 시키기" |
| 설치 난이도 | 매우 쉬움 | 확장 설치 + 토큰 설정 |

→ **서비스 제공자라면 ①**, **업무 자동화 사용자라면 ②**

---

## 7. 보안 설계와 남는 리스크

### 내장 안전장치

| 장치 | 내용 |
| --- | --- |
| **요소 허용/차단 목록** | 특정 요소는 AI가 아예 건드릴 수 없게 |
| **지침 안전 제약** | 시스템 프롬프트 레벨 제약 |
| **고위험 작업 통제** | **완전 금지** / **사용자 확인 필수** 2단계 |
| **데이터 마스킹** | `transformPageContent`로 AI 전송 전 가림 |
| **`execute_javascript` 기본 OFF** | 문서 경고: *"예측 불가능한 부작용 + 데이터 마스킹 우회 가능"* |
| **확장 토큰 인증** | 웹페이지가 확장을 쓰려면 사용자가 직접 토큰 발급 |
| **SimulatorMask** | 작업 중 사용자 오조작 차단 |
| **`ask_user` 도구** | 막히면 추측하지 말고 사람에게 물어보게 |

### 데이터 마스킹 예시

```js
transformPageContent: (content) => content
  .replace(/010-\d{4}-\d{4}/g, '010-****-****')   // 전화번호
  .replace(/\d{6}-\d{7}/g, '******-*******')      // 주민번호
  .replace(/\d{4}-\d{4}-\d{4}-\d{4}/g, '****')    // 카드번호
```

### ⚠️ 근본적으로 남는 리스크

**페이지 DOM 전체가 LLM API로 전송됩니다.** 고객 이름, 전화번호, 주소가 화면에 있으면 그게 다 나갑니다.

해결책 3가지:

1. **마스킹 설정**을 반드시 걸기
2. **로컬 모델** 사용 (Ollama 등 — 외부 전송 0)
3. 민감 데이터 없는 페이지에서만 쓰기

> 금융·의료·법률 분야라면 이건 **선택이 아니라 필수**입니다.

---

## 8. 설치 및 사용법

### 방법 A. 가장 빠른 체험 — 설치 0초, API 키 불필요

```html
<script
  src="https://cdn.jsdelivr.net/npm/page-agent@1.12.4/dist/iife/page-agent.demo.js"
  crossorigin="anonymous"
></script>
```

중국 CDN 미러: `https://registry.npmmirror.com/page-agent/1.12.4/files/dist/iife/page-agent.demo.js`

> ⚠️ 이건 알리바바가 제공하는 **무료 테스트 LLM**을 씁니다. 공식 문서에 **"기술 평가 목적 전용(For technical evaluation only)"** 으로 명시 → **상업적 사용 금지**. 입력 내용이 그쪽 서버로 갑니다.

**쿼리스트링 옵션**

| 파라미터 | 기본값 | 설명 |
| --- | --- | --- |
| `model` | `qwen3.5-plus` | 모델명 |
| `baseURL` | 알리바바 테스트 엔드포인트 | API 주소 |
| `apiKey` | `NA` | API 키 |
| `lang` | `zh-CN` | `en-US` / `zh-CN` |
| `showPanel` | `true` | 패널 표시 여부 |
| `autoInit` | `true` | 자동 생성 여부 |

```html
<!-- 자동 시작 끄고 직접 만들기 -->
<script src=".../page-agent.demo.js?autoInit=false"></script>
<script>
  const agent = new window.PageAgent({ model: '...', baseURL: '...', apiKey: '...' })
  agent.panel.show()
</script>
```

### 방법 B. 북마클릿 — 아무 웹사이트에나 주입 (실험용)

브라우저 북마크 URL에 입력:

```js
javascript:(function(){var s=document.createElement('script');s.src='https://cdn.jsdelivr.net/npm/page-agent@1.12.4/dist/iife/page-agent.demo.js';s.crossOrigin='anonymous';document.body.appendChild(s)})()
```

`demo.ts`에 북마클릿 중복 주입 방지 코드가 있어 의도된 사용법입니다. 단 CSP가 엄격한 사이트는 차단됩니다.

### 방법 C. 실서비스용 — npm (권장)

```bash
npm install page-agent zod   # zod는 peerDependency(^3.25.0 || ^4.0.0) → 직접 설치 필요
```

```js
import { PageAgent } from 'page-agent'

const agent = new PageAgent({
  // ── 필수 ──
  model: 'qwen3.5-plus',
  baseURL: 'https://dashscope.aliyuncs.com/compatible-mode/v1',
  apiKey: 'YOUR_API_KEY',

  // ── 선택 ──
  language: 'en-US',       // 'en-US' | 'zh-CN'  (한국어 없음)
  maxSteps: 40,
  stepDelay: 0.4,
  enableMask: true,        // 작업 중 화면 보호막
  promptForNextTask: true,
})

agent.panel.show()                                  // 사용자 UI로 쓰기
const result = await agent.execute('로그인 버튼을 클릭해줘')   // 코드로 실행
console.log(result.success, result.data, result.history)
await agent.stop()                                  // 중단 (완전히 멈출 때까지 await)
agent.dispose()                                     // 정리
```

### 실전 설정 예시 (고급 기능 전부)

```js
const agent = new PageAgent({
  model: 'gpt-5.2',
  baseURL: 'https://api.openai.com/v1',
  apiKey: process.env.OPENAI_KEY,

  // ① 개인정보 마스킹
  transformPageContent: (content) => content
    .replace(/010-\d{4}-\d{4}/g, '010-****-****')
    .replace(/\d{6}-\d{7}/g, '******-*******'),

  // ② 전역 지침 + ③ URL별 동적 지침
  instructions: {
    system: '너는 우리 ERP 전문가다. 결제·삭제 작업은 반드시 ask_user로 먼저 확인해라.',
    getPageInstructions: (url) => {
      if (url.includes('/orders'))   return '주문 목록에서는 상태 필터를 먼저 적용해라.'
      if (url.includes('/settings')) return '설정 변경은 절대 하지 마라.'
      return null
    },
  },

  // ④ 도구 커스터마이즈
  customTools: {
    ask_user: null,   // 되묻기 비활성화 (완전 무인 운영용)
  },

  // ⑤ 생명주기 훅 — 로깅/모니터링
  onBeforeStep: (agent, step) => console.log(`[${step}] 시작`),
  onAfterTask:  (agent, result) => sendToAnalytics(result),

  // ⑥ 위험 기능 (기본 OFF)
  experimentalScriptExecutionTool: false,
  experimentalLlmsTxt: true,
})
```

### 방법 D. UI 없이 헤드리스로 (`@page-agent/core`)

```bash
npm install @page-agent/core @page-agent/page-controller zod
```

```js
import { PageAgentCore } from '@page-agent/core'
import { PageController } from '@page-agent/page-controller'

const agent = new PageAgentCore({
  model: '...', baseURL: '...', apiKey: '...',
  pageController: new PageController({ enableMask: true }),
})

agent.addEventListener('activity',      (e) => myCustomUI.render(e.detail))
agent.addEventListener('statuschange',  () => console.log(agent.status))
agent.addEventListener('historychange', () => setHistory([...agent.history]))
agent.onAskUser = async (q) => await myCustomDialog(q)

await agent.execute('...')
```

### 방법 E. 크롬 확장 (여러 탭 작업)

1. [Chrome Web Store](https://chromewebstore.google.com/detail/page-agent-ext/akldabonmimlicnjlflnapfeklbfemhj)에서 설치 (또는 [GitHub Releases](https://github.com/alibaba/page-agent/releases)에서 최신판 — 보통 더 빠름)
2. 사이드패널 열기 → LLM 설정 입력 → 바로 사용

**웹페이지 JS에서 확장을 호출하려면 토큰 필요:**

```js
// 1. 확장 사이드패널에서 토큰 복사 후
localStorage.setItem('PageAgentExtUserAuthToken', 'your-token')

// 2. 호출
const result = await window.PAGE_AGENT_EXT.execute(
  '이메일 칸에 test@example.com 넣고 제출 눌러',
  {
    baseURL: 'https://api.openai.com/v1',
    apiKey: process.env.OPENAI_API_KEY,
    model: 'gpt-5.2',
    includeInitialTab: false,
    onStatusChange: (s) => console.log(s),
    onActivity: (a) => console.log(a),
  }
)
window.PAGE_AGENT_EXT.stop()
```

타입 정의: `npm install @page-agent/core --save-dev`

> **왜 토큰이 필요한가** (공식 문서): 확장은 광범위한 브라우저 권한(페이지 접근, 네비게이션, 멀티탭 제어)을 갖고, 악용되면 프라이버시·보안을 해칠 수 있으므로, 사용자가 **신뢰하는 앱에만 명시적으로** 토큰을 제공해야 합니다.

### 방법 F. MCP 서버 (Claude Desktop에서 내 브라우저 조종)

**전제조건**: Node.js ≥ 20, 크롬 확장 설치됨, LLM API 키

`~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "page-agent": {
      "command": "npx",
      "args": ["-y", "@page-agent/mcp"],
      "env": {
        "LLM_BASE_URL": "https://dashscope.aliyuncs.com/compatible-mode/v1",
        "LLM_API_KEY": "sk-xxx",
        "LLM_MODEL_NAME": "qwen3.5-plus"
      }
    }
  }
}
```

Cursor / Copilot도 같은 형식. 포트는 `PORT`(기본 `38401`).

### 방법 G. 소스 직접 빌드 (개발/학습)

```bash
git clone https://github.com/bmshin94/page-agent.git
cd page-agent
npm install           # Node ^22.22.1 || >=24, npm ^11.6.3 필요

npm start             # 문서 웹사이트 dev 서버
npm run dev:demo      # 데모 빌드 watch + localhost:5174
npm run dev:ext       # 확장 개발모드
npm run build         # 전체 빌드
npm run build:ext     # 확장 zip 패키징
npm run typecheck     # 타입 검사
npm test              # 테스트
npm run test:live     # 실제 API 호출 테스트
npm run lint          # ESLint
npm run ci            # CI 전체
```

---

## 9. 플러그인? 스킬? MCP?

**정답: 전부 다 아니고, "전부 다"입니다.**

본질은 **npm 라이브러리(SDK)** 이고, 나머지는 부가 형태입니다.

```text
📦 본체   = npm 라이브러리 (page-agent)        ← 정체성
🧩 부가 1 = 크롬 확장 (브라우저 플러그인)
🔌 부가 2 = MCP 서버 (@page-agent/mcp, Beta)
🛠️ 부가 3 = .agents/skills/ (AI 코딩도구용 스킬 5개)
```

| 질문 | 답 | 설명 |
| --- | --- | --- |
| **플러그인이야?** | **반쯤 예** | 크롬 확장이 있지만 **선택적 부가 기능**. 본체는 확장 없이 완전히 동작. "VSCode 플러그인" 같은 의미는 아님 |
| **스킬이야?** | **아니오. 단, 스킬을 품고 있음** | Claude Code 스킬이 아닙니다. 대신 `.agents/skills/`에 **이 프로젝트를 개발하기 위한** 스킬 5개가 들어 있습니다. 방향이 반대 — "page-agent가 스킬"이 아니라 "page-agent가 스킬을 **쓴다**" |
| **MCP야?** | **반쯤 예 (베타)** | `@page-agent/mcp`는 진짜 MCP 서버. 단 **크롬 확장이 반드시 설치돼 있어야** 작동하고 공식적으로 **Beta** |

### 4가지 역할

```text
① 웹 개발자용 SDK       → npm i page-agent → 내 사이트에 AI 비서 탑재
② 브라우저 사용자용 확장  → Chrome Web Store → 모든 사이트에서 자동화
③ AI 클라이언트용 MCP    → Claude Desktop이 내 브라우저를 조종
④ 다른 에이전트의 도구    → 내 에이전트의 tool로 등록 (third-party-agent 문서)
```

④번: 공식 문서 "Third-party Agent Integration"에 정식 지원으로 문서화. LangChain/CrewAI 같은 내 에이전트에 "웹페이지 조작" 도구로 끼워넣을 수 있습니다. 문서가 드는 용도 — 🤖 스마트 고객센터 / 📋 업무 프로세스 비서 / 🎯 개인 생산성 비서 / 🔧 DevOps 자동화.

---

## 10. API 토큰 필요 여부

**결론: 거의 항상 필요합니다. 단 3가지 예외가 있습니다.**

토큰이 **두 종류**라는 점을 구분해야 합니다.

### 🔑 토큰 ①: LLM API 키 (필수)

```js
new PageAgent({ baseURL: '...', apiKey: 'sk-...', model: 'gpt-5.2' })
```

**안 써도 되는 예외 3가지**

| 예외 | 방법 | 주의 |
| --- | --- | --- |
| **1. 데모 CDN** | 스크립트 1줄 | `apiKey: 'NA'` 기본값. **"기술 평가 전용"**, 실서비스 금지 |
| **2. 로컬 모델** ⭐ | Ollama / LM Studio / vLLM | `baseURL: 'http://localhost:11434/v1'`, `apiKey: 'ollama'`(아무값). **비용 0원 + 유출 0** |
| **3. 사내 LLM 게이트웨이** | 게이트웨이가 인증 처리 | `customFetch`로 헤더 주입 |

### 🔑 토큰 ②: 확장 인증 토큰

`localStorage.setItem('PageAgentExtUserAuthToken', '...')` — **웹페이지 JS가 확장을 호출할 때만** 필요. 확장 사이드패널에서 직접 쓸 때는 불필요.

### 🚨 가장 중요한 보안 경고

```js
// ❌❌❌ 절대 금지 — 프론트엔드에 API 키 노출
const agent = new PageAgent({ apiKey: 'sk-proj-abc123...' })
```

이건 **브라우저에서 돌아가는 코드**입니다. 개발자도구 Network 탭에서 **누구나 훔쳐볼 수 있습니다.**

### ✅ 실서비스 올바른 패턴 — 프록시 서버

```js
// 프론트엔드: 키 없음
const agent = new PageAgent({
  baseURL: 'https://my-api.com/llm-proxy',   // 내 서버
  model: 'gpt-5.2',
  // apiKey 생략!
  customFetch: (url, init) => fetch(url, { ...init, credentials: 'include' }),
})
```

```js
// 내 백엔드 (Express)
app.post('/llm-proxy/chat/completions', requireLogin, rateLimit, async (req, res) => {
  const r = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_KEY}`,  // ← 서버에만 존재
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(req.body),
  })
  r.body.pipe(res)
})
```

프록시를 두면 덤으로 **사용량 제한, 과금 추적, 감사 로그, 프롬프트 검열**까지 가능합니다.

### 💰 비용 감각

| 항목 | 값 |
| --- | --- |
| 1스텝 = LLM 호출 1회 | DOM 텍스트가 길면 입력 토큰이 큼 |
| 1작업 = 보통 5~15스텝 | 복잡하면 40스텝(기본 상한)까지 |
| 비용 절감 요소 | 스크린샷 없음 + 프롬프트 캐싱 지원 |
| 모니터링 | `AgentStepEvent.usage`에 `promptTokens` / `completionTokens` / `cachedTokens` / `reasoningTokens` 기록 |

---

## 11. AI 에이전트 구축에 도움이 되는가

**결론: 엄청나게 도움됩니다.** 방향이 두 가지입니다.

### 🅰️ 교재로서 (★★★★★ — 진짜 가치)

에이전트를 처음 만들 때 가장 어려운 건 **"안이 어떻게 생겼는지"** 를 모르는 것입니다. LangChain은 추상화가 두꺼워 내부가 안 보이고, 논문은 코드가 없습니다. **이 프로젝트의 뇌는 661줄입니다. 하루면 다 읽습니다.**

#### 베낄 수 있는 설계 패턴 7개

| # | 패턴 | 설명 |
| --- | --- | --- |
| ① | **MacroTool (반성 강제)** | ⭐ 최고의 수확. 단 하나의 매크로 도구만 노출하고 평가/기억/목표를 스키마로 강제. 모든 에이전트에 재사용 가능 |
| ② | **도구 최소화** | 9개뿐. 선택지가 많으면 AI가 헷갈림 |
| ③ | **컨텍스트 압축** | DOM 수만 노드 → 수백 줄. "토큰이 곧 돈"인 에이전트의 핵심 기술 |
| ④ | **변화 감지(delta) 표현** | `*[14]`로 새 요소만 표시. 저렴하고 효과적 |
| ⑤ | **Abort 신호 전파** | 중단 버튼을 *진짜로* 동작하게 만드는 법 |
| ⑥ | **두 정보 스트림 분리** | History(AI 기억) vs Activity(UI 표시) |
| ⑦ | **모델별 호환 패치 레이어** | 멀티모델 지원 시 반드시 겪는 지옥을 미리 봄 |

⑦ 관련 체인지로그 실례:

- *"Rewrote per-model request patching for GPT, Claude, Qwen, Gemini, and DeepSeek"*
- *"Skip `reasoning_effort` and `temperature` patches for `*-chat-latest` models"*
- *"Do not enable reasoning by default on OpenRouter"*
- *"Deprecate `temperature` — many models reject it outright"*

### 🅱️ 부품으로서 (★★★★☆)

```js
// LangChain / 자체 에이전트에 도구로 등록
const webTool = {
  name: 'operate_webpage',
  description: '현재 웹페이지를 자연어 지시로 조작한다',
  execute: async ({ instruction }) => (await pageAgent.execute(instruction)).data,
}
```

상위 에이전트는 **의도**만 넘기면 됩니다. 셀렉터·클릭 순서는 Page Agent가 처리 → **셀렉터 변경에 안 깨지는 자동화**.

### ⚠️ 도움이 안 되는 경우

| 상황 | 이유 |
| --- | --- |
| **서버사이드 스크래이핑** | README 명시: *"client-side web enhancement용이며 server-side automation용이 아니다"* → browser-use / Playwright |
| **멀티 에이전트 오케스트레이션** | 단일 에이전트 루프 설계. 에이전트 간 협업/위임 구조 없음 |
| **RAG / 지식베이스** | 전혀 다른 영역 |
| **범용 에이전트 프레임워크** | 이건 "웹 조작 특화". LangGraph/CrewAI 영역이 다름 |
| **이미지/좌표 기반 GUI 제어** | DOM 기반이라 불가. computer-use 계열 필요 |

### 📚 추천 학습 순서

```text
1일차: packages/core/src/prompts/system_prompt.md
       → "에이전트에게 뭘 어떻게 말해야 하는가"의 모범 답안
2일차: packages/core/src/types.ts + tools/index.ts
       → 도구 설계 + 설정 API 설계
3일차: packages/core/src/PageAgentCore.ts (661줄)
       → 에이전트 루프 전체
4일차: packages/llms/src/ + 테스트 코드
       → LLM 통신, 재시도, 모델 호환성
5일차: packages/mcp/src/ (3파일)
       → MCP 서버 만드는 법
보너스: .agents/skills/ 5개
       → Claude Code 스킬 작성법 실전 예시
```

> **결론**: "에이전트 구축에 도움이 되냐"가 아니라, **에이전트 구축을 배우기에 현존하는 가장 좋은 소규모 레퍼런스 중 하나**입니다.

---

## 12. React / PHP 로 만들 수 있는가

### 🅰️ React: 네, 완벽하게 가능합니다

증거: `packages/page-controller/src/patches/react.ts`와 `patches/antd.ts`가 **존재합니다.** React의 Synthetic Event 시스템을 뚫는 패치가 이미 들어 있습니다.

```jsx
// hooks/usePageAgent.js
import { useEffect, useRef } from 'react'
import { PageAgent } from 'page-agent'

export function usePageAgent(config) {
  const agentRef = useRef(null)
  useEffect(() => {
    const agent = new PageAgent(config)
    agentRef.current = agent
    agent.panel.show()
    return () => agent.dispose()   // ← 언마운트 시 정리 필수
  }, [])
  return agentRef
}
```

내 UI를 쓰려면 headless 코어 사용:

```jsx
import { PageAgentCore } from '@page-agent/core'
import { PageController } from '@page-agent/page-controller'

function MyAICopilot() {
  const [status, setStatus] = useState('idle')
  const [activity, setActivity] = useState(null)
  const [history, setHistory] = useState([])
  const agentRef = useRef(null)

  useEffect(() => {
    const agent = new PageAgentCore({
      model: 'gpt-5.2',
      baseURL: '/api/llm-proxy',
      pageController: new PageController({ enableMask: true }),
    })
    agent.addEventListener('statuschange',  () => setStatus(agent.status))
    agent.addEventListener('activity',      (e) => setActivity(e.detail))
    agent.addEventListener('historychange', () => setHistory([...agent.history]))
    agent.onAskUser = async (q) => window.prompt(q) ?? ''
    agentRef.current = agent
    return () => agent.dispose()
  }, [])

  return <MyChatUI status={status} activity={activity} history={history}
                   onSend={(t) => agentRef.current.execute(t)} />
}
```

**React 주의 4가지**

1. `useEffect` cleanup에서 **`dispose()` 필수** — 안 하면 중복 인스턴스 + 메모리 누수
2. **StrictMode 이중 마운트** — 개발 중 두 번 생성. `useRef` 가드 권장
3. **SSR (Next.js)** — `window`/`document` 사용 → `'use client'` + `dynamic(..., { ssr: false })`
4. **`zod` peerDependency** 직접 설치

### 🅱️ PHP: "네"와 "아니오"가 섞입니다

```text
┌──────────────────────────────────────────────┐
│  브라우저 (클라이언트)                        │
│  Page Agent는 ★여기서만★ 돌아갑니다           │
│  JavaScript 전용. 선택권 없음.                │
└──────────────────────────────────────────────┘
                    ↕ HTTP
┌──────────────────────────────────────────────┐
│  서버 — PHP / Node / Python / Java 아무거나    │
│  Page Agent를 ★실행할 수는 없지만★,            │
│  ★떠받쳐주는 역할★은 완벽하게 가능             │
└──────────────────────────────────────────────┘
```

#### ❌ PHP로 안 되는 것

- **PHP로 Page Agent 재구현** — 불가능. DOM을 읽고 클릭 이벤트를 쏘는 건 브라우저 JS만 가능
- **PHP 포팅** — 의미 없음

#### ✅ PHP로 되는 것 (아주 유용함)

**1) PHP 사이트에 Page Agent 얹기** — Laravel / WordPress / CodeIgniter / 생 PHP 전부 가능. 프레임워크 무관.

```php
<!-- Laravel Blade -->
<script type="module">
  import { PageAgent } from 'https://cdn.jsdelivr.net/npm/page-agent@1.12.4/dist/esm/page-agent.js'
  const agent = new PageAgent({
    model: 'gpt-5.2',
    baseURL: '{{ route("llm.proxy") }}',          // ← PHP 라우트
    instructions: { system: @json(auth()->user()->ai_instructions ?? '') },
  })
  agent.panel.show()
</script>
```

**2) PHP를 LLM 프록시로 쓰기 — ⭐ 가장 중요한 역할**

```php
<?php
class LlmController extends Controller
{
    public function proxy(Request $request)
    {
        $user = $request->user();

        // ① 사용량 제한 (과금 폭탄 방지)
        if ($user->ai_calls_today >= $user->plan->daily_limit) {
            return response()->json(['error' => '일일 한도 초과'], 429);
        }

        // ② 감사 로그
        Log::channel('ai_audit')->info('llm_call', [
            'user_id' => $user->id,
            'model'   => $request->input('model'),
            'ip'      => $request->ip(),
        ]);

        // ③ 실제 LLM 호출 — ★API 키는 서버에만 존재★
        $response = Http::withToken(config('services.openai.key'))
            ->timeout(120)
            ->post('https://api.openai.com/v1/chat/completions', $request->all());

        // ④ 토큰 사용량 과금 집계
        if ($usage = $response->json('usage')) {
            $user->increment('ai_calls_today');
            $user->increment('ai_tokens_used', $usage['total_tokens'] ?? 0);
        }

        return response($response->body(), $response->status())
                 ->header('Content-Type', 'application/json');
    }
}
```

PHP가 해주는 일: 🔐 API 키 은닉 / 🚦 사용자별 쿼터 / 📊 토큰 집계(= 과금 로직) / 📝 감사 로그 / 🛡️ 로그인 검사 + Rate limiting

**3) AI 지침을 DB·관리화면에서 동적 관리**

```js
instructions: {
  getPageInstructions: async (url) => {
    const r = await fetch(`/api/ai-instructions?url=${encodeURIComponent(url)}`)
    return (await r.json()).page
  },
}
```

→ **코드 배포 없이** 관리자 화면에서 AI 행동 변경. SaaS로 팔 때 큰 무기.

**4) 작업 로그 저장 / 리플레이**

```js
onAfterTask: async (agent, result) => {
  await fetch('/api/ai-logs', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      task: agent.task, success: result.success,
      steps: result.history.length, history: result.history,
    }),
  })
}
```

**5) WordPress 플러그인** ⭐ 수익화 유망

```php
<?php
/** Plugin Name: AI Copilot for WP Admin */
add_action('admin_footer', function () {
    $cfg = ['model' => get_option('aicp_model', 'gpt-5.2'),
            'baseURL' => admin_url('admin-ajax.php?action=aicp_proxy')];
    ?>
    <script type="module">
      import { PageAgent } from '<?= plugins_url('assets/page-agent.js', __FILE__) ?>'
      new PageAgent(<?= wp_json_encode($cfg) ?>).panel.show()
    </script>
    <?php
});
```

### 정리표

| 하고 싶은 것 | React | PHP |
| --- | --- | --- |
| Page Agent **실행** | ✅ 네이티브 | ❌ 불가 (브라우저 JS 전용) |
| 내 사이트에 **얹기** | ✅ 매우 쉬움 | ✅ 매우 쉬움 (script 태그) |
| 커스텀 **채팅 UI** | ✅ `@page-agent/core` | ✅ (JS 부분만) |
| **API 키 숨기기** | ❌ 프론트라 불가 | ✅✅ **PHP의 핵심 역할** |
| **과금/쿼터/로그** | ❌ | ✅✅ PHP 담당 |
| AI **지침 동적 관리** | ⭕ | ✅✅ DB + 관리화면 |
| **PHP로 재구현** | — | ❌ 의미 없음 |

> **최적 조합**: 프론트는 React(또는 그냥 script 태그) + 백엔드는 PHP 프록시. 경쟁 관계가 아니라 **역할 분담**입니다.

---

## 13. 유튜브 강의 제작 가능성

**결론: 매우 좋은 소재입니다.** 특히 시각적 임팩트가 압도적입니다.

### 왜 유리한가

| 요소 | 평가 |
| --- | --- |
| **시각적 임팩트** | ⭐⭐⭐⭐⭐ 마우스가 저절로 움직이며 클릭하는 화면. 썸네일/첫 10초 이탈률을 잡아줌 |
| **진입장벽** | ⭐⭐⭐⭐⭐ `<script>` 1줄. 환경설정 지옥 없음 → **시청자가 실제로 따라함** |
| **신뢰도** | ⭐⭐⭐⭐ 알리바바 공식 오픈소스 + Trendshift 랭킹 + HN 토론 |
| **주제 신선도** | ⭐⭐⭐⭐⭐ "AI 에이전트" 최고 관심 키워드 |
| **한국어 콘텐츠 부재** | ⭐⭐⭐⭐⭐ 거의 없음. **선점 가능** |
| **코드 양** | ⭐⭐⭐⭐ 적어서 영상 호흡이 안 늘어짐 |
| **시리즈화** | ⭐⭐⭐⭐⭐ 입문 → 실전 → 수익화 → 심화 |

### 리스크와 대응

| 리스크 | 대응 |
| --- | --- |
| **API 키 노출 사고 유발** | 영상에서 **반드시** 경고 + 프록시 패턴 한 편 따로. 안 하면 시청자 피해 + 채널 신뢰 손상 |
| **데모 LLM 상업 사용 오해** | "기술 평가 전용" 자막으로 명시 |
| **개인정보 전송 리스크** | 마스킹 편 필수. 실무자 1순위 질문 |
| **버전 변화 빠름** | 영상에 버전 명시(`1.12.4`) + 고정 댓글 업데이트 |
| **촬영 시 실제 계정 노출** | 더미 사이트 / 로컬 테스트 앱 사용 |
| **"AI가 멋대로 결제" 우려** | 안전장치 편(마스킹/허용목록/`ask_user`) 필수 |

### 추천 시리즈 커리큘럼 (10편)

| # | 제목 | 길이 | 후킹 포인트 |
| --- | --- | --- | --- |
| **1** | **"웹사이트에 한 줄 넣으면 AI가 대신 클릭해줍니다"** | 8분 | 🔥 첫 10초에 자동 클릭 장면. 전환율 최고 |
| 2 | 내 API 키로 바꾸고 모델 고르기 (+ 비용 계산) | 12분 | 실제 토큰 비용을 숫자로 |
| **3** | **⚠️ API 키 노출 사고 막기 — 프록시 서버 만들기** | 15분 | Node/PHP 양쪽. **가장 실용적** |
| 4 | React 앱에 AI 코파일럿 넣기 (내 UI로) | 18분 | `@page-agent/core` + 커스텀 UI |
| **5** | **개인정보 안 새게 막기 — 마스킹 & 안전장치** | 12분 | 실무자 최대 관심사 |
| 6 | 프롬프트로 AI 길들이기 — instructions 전략 | 15분 | "AI가 엉뚱하게 할 때" 해결 |
| 7 | 커스텀 도구 만들기 — customTools | 15분 | 내 비즈니스 로직을 AI 손에 |
| 8 | 크롬 확장으로 여러 탭 넘나들기 | 12분 | 멀티탭 자동화 데모 |
| **9** | **Claude Desktop이 내 브라우저를 조종한다 (MCP)** | 15분 | 🔥 MCP 관심 폭발 중 |
| **10** | **661줄로 읽는 AI 에이전트 내부 구조** ⭐ | 25분 | **가장 차별화되는 편** |

### 보너스 편 아이디어

- "AI로 세금신고 사이트 자동화해봤다" (실험 콘텐츠)
- "Page Agent vs browser-use vs Playwright 비교" (검색 유입)
- "이걸로 돈 벌 수 있나? 수익화 10가지"
- **"알리바바 오픈소스에 한국어 번역 기여하기"** — i18n이 없으므로 실제 PR까지 가는 영상. 신입/주니어 타겟 ⭐
- "AI가 못 하는 것들 — 솔직한 한계" (역설적으로 신뢰도 상승)
- "프롬프트 하나로 사이트 전체 테스트하기 (QA 자동화)"

### 제작 실무 팁

- **더미 데모 사이트**를 하나 만들어두세요 (가짜 쇼핑몰 관리자 / 가짜 ERP). 실제 사이트는 약관 위반 + 개인정보 노출 위험
- `enableMask: true`면 **가상 커서 애니메이션**이 나와 화면이 훨씬 "AI가 일하는 것처럼" 보임
- `stepDelay`를 0.8~1초로 올리면 시청자가 따라가기 쉬움
- 썸네일: 자동 움직이는 커서 순간 / `<script>` 1줄 vs 결과 Before-After / "코드 1줄" 숫자 강조

**설명란 필수 항목**

```text
📦 GitHub: https://github.com/alibaba/page-agent
📖 공식문서: https://alibaba.github.io/page-agent/
🏷️ 사용 버전: 1.12.4
⚠️ 데모 CDN의 무료 LLM은 기술 평가 전용입니다.
⚠️ 프론트엔드에 API 키를 넣지 마세요 (3편 참고)
```

> **라이선스**: MIT이므로 강의 제작·유료 강의 판매 모두 문제없습니다. 저작권 표시만 지키면 됩니다.

---

## 14. 수익화 아이디어 10가지

> ⚖️ **법적 전제**: MIT 라이선스 — 상업 이용/수정/재배포/클로즈드 소스 SaaS 판매 모두 허용. 의무는 저작권 표시 + 라이선스 사본뿐.
> ⚠️ **단 하나의 제약**: 데모 CDN의 무료 LLM API는 **"기술 평가 목적 전용"**. 상업 서비스에서는 반드시 내 API 키 또는 로컬 모델로 교체.

### 전체 비교

| # | 아이디어 | 초기 투입 | 현금화 속도 | 수익 상한 | 경쟁 | 추천도 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | AI 코파일럿 수탁 개발 | 1주 | ⚡ 즉시 | 💰💰💰 | 낮음 | ⭐⭐⭐⭐⭐ |
| 2 | 버티컬 SaaS (업종 특화) | 3~6개월 | 느림 | 💰💰💰💰💰 | 중간 | ⭐⭐⭐⭐⭐ |
| 3 | Copilot-as-a-Service | 4~8개월 | 느림 | 💰💰💰💰💰 | 높음 | ⭐⭐⭐ |
| 4 | 웹 접근성 솔루션 | 2~3개월 | 중간 | 💰💰💰 | **매우 낮음** | ⭐⭐⭐⭐⭐ |
| 5 | AI QA 테스팅 도구 | 2~4개월 | 중간 | 💰💰💰 | 중간 | ⭐⭐⭐⭐ |
| 6 | 노코드 웹 자동화 | 4~6개월 | 느림 | 💰💰💰 | 높음 | ⭐⭐ |
| 7 | 교육 (강의/코스/책) | 2주 | ⚡ 빠름 | 💰💰 | 낮음 | ⭐⭐⭐⭐⭐ |
| 8 | 한국어화 + 국내 배포판 | 2주 | 간접 | 💰 (간접) | 없음 | ⭐⭐⭐⭐ |
| 9 | LLM 프록시/게이트웨이 | 1~2개월 | 중간 | 💰💰 | 높음 | ⭐⭐ |
| 10 | 플랫폼 플러그인 (WP/ERP) | 1~2개월 | 중간 | 💰💰💰 | 낮음 | ⭐⭐⭐⭐ |

---

### 💼 1. "AI 코파일럿 심어드립니다" — 수탁 개발/컨설팅 ⭐⭐⭐⭐⭐

**가장 현실적이고 지금 당장 가능**

거의 모든 기업이 "우리 서비스에 AI 좀 넣어주세요"라는 숙제를 안고 있지만, 발주처는 뭘 넣어야 할지 모르고, 백엔드 재작성은 몇 달 + 수억이며, 결과물이 "그냥 챗봇"이면 아무도 안 씁니다.

**결정적 장점: 백엔드를 건드리지 않습니다.** 레거시 PHP든 JSP든 .NET이든 무관.

```text
기존 AI 챗봇 도입: 백엔드 API 설계 → 데이터 연동 → 권한 체계 → LLM 통합 → UI
                  = 3~6개월, 수억
Page Agent 방식:   script 1줄 + 프롬프트 튜닝 + 안전장치 + 프록시
                  = 2~6주, 수백~수천만
```

**가격 모델**

| 상품 | 범위 | 가격대 | 기간 |
| --- | --- | --- | --- |
| **PoC / 데모 제작** | 핵심 업무 3개 시나리오 | 300~800만원 | 1~2주 |
| **기본 구축** | 프록시 + 마스킹 + 프롬프트 튜닝 + 커스텀 UI | 1,500~3,000만원 | 4~8주 |
| **엔터프라이즈** | 위 + 감사로그 + SSO + 온프레미스 LLM + 교육 | 5,000만~1.5억 | 3~6개월 |
| **유지보수** | 프롬프트 개선, 모델 업데이트, 모니터링 | 월 100~300만원 | 연간 계약 |

**🎯 핵심 영업 무기: "PoC를 1주일에 만들어 보여준다"**

```text
1일차: 고객사 사이트 접속 → 북마클릿으로 Page Agent 주입
2일차: 고객의 실제 업무 3개를 자연어로 시연 → 녹화
3일차: 영상 + 분석 보고서로 제안
```

경쟁사가 제안서만 쓰고 있을 때 **동작하는 데모**를 들고 가면 게임이 끝납니다. 설치가 script 1줄이라 가능한 속도입니다.

**타겟 고객 (우선순위)**

| 순위 | 업종 | 이유 |
| --- | --- | --- |
| 🥇 | **ERP / CRM / 그룹웨어 중견기업** | 클릭 노동 최악. 효과 체감 즉시 |
| 🥇 | **쇼핑몰 / 커머스** | 주문·반품·재고가 반복 클릭 |
| 🥈 | **병원 / 약국 (EMR)** | 차트 입력 지옥. 단 민감정보 → 로컬 모델 필수 |
| 🥈 | **보험 / 금융 백오피스** | 폼 입력 많음. 규제 → 온프레미스 LLM |
| 🥉 | **공공기관 / 지자체** | 예산 + 접근성 의무. 단 조달 느림 |
| 🥉 | **물류 / 유통** | 송장, 배차, 입출고 |

**실행 체크리스트**

```text
□ 더미 데모 사이트 1개 제작 (가짜 ERP) — 영업용
□ "30초 자동화" 영상 3개 (주문처리/반품/보고서)
□ 프록시 서버 템플릿 (Node + PHP 둘 다)
□ 마스킹 설정 템플릿 (주민번호/전화/카드/계좌)
□ 감사로그 스키마 + 관리화면
□ 제안서 템플릿 (ROI 계산: "클릭 800회 → 1회")
```

**리스크**: 고객사가 "AI가 잘못 누르면 누가 책임지냐"고 묻습니다 → `ask_user` 강제 + 허용목록 + 감사로그로 답하고, **계약서에 책임 범위 명시 필수**. 고객사 DOM이 엉망이면(div만 1만개) 성능이 안 나오므로 **사전 기술 검증(1일) 유료화** 권장.

---

### 🏥 2. 업종별 버티컬 SaaS ⭐⭐⭐⭐⭐

**시간은 걸리지만 수익 상한이 가장 높음**

수탁은 일한 만큼만 돈이 됩니다. 같은 걸 여러 고객에게 팔아야 스케일합니다.

**왜 업종 특화여야 하는가**: 범용 "AI 웹 비서"는 아무도 안 삽니다. 하지만 **"병원 EMR 전용 AI 차트 입력 비서"** 는 병원장이 바로 이해합니다. 업종을 좁히면 프롬프트 최적화로 **성공률이 눈에 띄게 올라가고**, 영업 메시지가 뾰족해지고, 같은 업종 안에서 입소문이 전파됩니다.

**유망 버티컬 Top 5**

| 버티컬 | 타겟 업무 | 월 구독료 | 시장 규모 | 난관 |
| --- | --- | --- | --- | --- |
| **🏥 의료 (EMR)** | 차트/처방 입력, 보험 청구 | 10~30만원/의원 | 의원 3만+ | 의료정보 규제 → **온프레미스/로컬 LLM 필수** |
| **🎓 학원/교육 (LMS)** | 출결, 성적 입력, 학부모 알림 | 5~15만원/원 | 학원 7만+ | 상대적으로 쉬움 ⭐ |
| **⚖️ 법무/세무** | 서면 작성, 신고 사이트 입력 | 20~50만원/사무소 | 2만+ | 정확도 요구 극도로 높음 |
| **🏭 제조/물류 (ERP)** | 입출고, 발주, 송장 | 30~100만원/사업장 | 중소제조 수만 | ERP 종류 다양 |
| **🏪 커머스 (관리자)** | 주문/반품/재고/상품등록 | 10~30만원/몰 | 수십만 | 플랫폼 정책 확인 |

**🥇 가장 먼저 노릴 곳: 학원 / 교육** — 규제 리스크 낮음 + 반복 업무 극심 + 의사결정 빠름(원장 1인) + 월 10만원대 수용 가능 + 업계 입소문 강함.

**제품 구조**

```text
┌─ 고객 브라우저 ─────────────────────────┐
│ 기존 LMS/EMR 사이트                      │
│  + 우리 크롬 확장 (또는 북마클릿)         │
│    → Page Agent + 업종 특화 프롬프트      │
└──────────────┬──────────────────────────┘
               │ HTTPS
┌──────────────▼──────────────────────────┐
│ 우리 SaaS 백엔드                          │
│ · LLM 프록시 (키 은닉 + 쿼터 + 과금)      │
│ · 업종별 프롬프트 라이브러리 (DB 관리)     │
│ · 작업 템플릿 ("출결 일괄 입력" 등)        │
│ · 감사 로그 / 리플레이                    │
│ · 마스킹 룰 엔진                          │
│ · 관리자 대시보드 (사용량, 성공률)         │
└─────────────────────────────────────────┘
```

**가격 전략**

```text
무료        : 월 20작업 — 체험용
베이직      : 월 9만원  — 월 500작업, 1계정
프로        : 월 29만원 — 월 3,000작업, 5계정, 커스텀 프롬프트
엔터프라이즈 : 별도 견적 — 무제한, 온프레미스 LLM, SSO, 전담지원
```

핵심은 **LLM 원가를 넘는 마진 구조**. 플래시급 모델로 돌리면 작업당 원가가 매우 낮아 월 9만원에 500작업도 충분히 남습니다.

**로드맵**

```text
0~1개월 : 타겟 업종 결정 + 그 업종 5곳 심층 인터뷰 (무조건 먼저!)
1~2개월 : 가장 아픈 업무 1개만 완벽하게 → 무료 베타 3곳
2~4개월 : 프롬프트 반복 개선 (성공률 70% → 95%가 승부처)
4~6개월 : 프록시/과금/대시보드 완성 → 유료 전환
6개월~  : 같은 업종 수평 확장 (레퍼런스 영업)
```

**가장 큰 리스크: 성공률.** 70%면 아무도 안 씁니다. "10번 중 3번 틀리는 비서"는 직접 하는 것보다 나쁩니다. → **업무 범위를 극단적으로 좁혀 95%+** 를 만드세요. "모든 걸 해주는 비서"보다 **"출결 입력만 완벽하게 하는 비서"** 가 팔립니다.

---

### 🔌 3. Copilot-as-a-Service (임베드형) ⭐⭐⭐

> **"Intercom이 채팅 위젯을 팔듯, 우리는 'AI 조작 위젯'을 판다"**

```html
<!-- 고객사는 이 한 줄만 -->
<script src="https://cdn.우리서비스.com/copilot.js?key=pk_live_xxx"></script>
```

**제품 구성**: 관리 대시보드(프롬프트 편집, 허용/차단 요소, 마스킹 룰, 사용량·성공률 통계) / 브랜딩 커스터마이징 / 안전장치 / 작업 템플릿(원클릭 자동화) / 분석("사용자들이 AI에게 뭘 가장 많이 시키나" → 고객사 UX 인사이트로 판매)

**과금 모델**

```text
Free      : 월 100작업, 브랜딩 노출
Starter   : $49/월  — 월 2,000작업
Growth    : $199/월 — 월 10,000작업, 브랜딩 제거, 커스텀 프롬프트
Scale     : $699/월 — 월 50,000작업, SSO, SLA
Enterprise: 별도    — 온프레미스, BYO-LLM
+ 초과분 작업당 $0.02~0.05
```

**차별화 포인트 (중요)**: 그냥 "Page Agent 호스팅"은 가치가 없습니다(오픈소스니까). 팔리는 건 **위에 올린 운영 레이어**입니다.

| 가치 | 설명 |
| --- | --- |
| 🔐 API 키 관리 | 고객사가 프록시 안 만들어도 됨 |
| 📊 **관측성** | 성공률/실패 원인/단계별 리플레이 — 없으면 운영 불가 |
| 🛡️ 거버넌스 | 누가 AI에게 뭘 시켰나 전수 로그. 규제 산업 필수 |
| 🧠 프롬프트 자산 | 업종별 검증된 프롬프트 라이브러리 (쌓이면 진짜 자산) |
| 🔁 모델 추상화 | 새 모델 나오면 우리가 호환 처리 |
| 💰 비용 최적화 | 쉬운 작업 → 저가 모델 자동 라우팅 |

**리스크**: 아이디어가 명확해 진입장벽이 낮음(속도 + 버티컬 집중으로만 이김) / 고객이 직접 만들 수 있음(→ "직접 만들면 6개월, 우리 쓰면 1일"로 설득) / 작업 수 과금 + 토큰 원가 구조에서 긴 작업 손실 → **작업당 토큰 상한 필수**

---

### ♿ 4. 웹 접근성 솔루션 — 숨은 블루오션 ⭐⭐⭐⭐⭐

**경쟁이 거의 없고 법적 수요가 있음.** 이건 "있으면 좋은 것"이 아니라 **법으로 해야 하는 것**입니다.

- 한국: 「장애인차별금지법」 — 공공기관·일정 규모 이상 민간 웹사이트 접근성 의무
- 미국: ADA 소송 매년 수천 건 / EU: European Accessibility Act

그런데 기존 접근성 솔루션은 **"읽어주기 + 글자 크게"** 수준입니다. **"조작해주기"** 는 없습니다.

```text
기존 스크린리더: "저기에 로그인 버튼이 있습니다" → 사용자가 직접 찾아 눌러야 함
우리 솔루션:     "로그인해줘"                    → 실제로 눌러줌
```

**수혜 대상**: 👁️ 시각장애인(복잡한 폼을 자연어로) / 🖐️ 운동장애인(정밀 클릭 불필요) / 👴 고령자("어디 눌러야 해?"가 사라짐) / 🧠 인지장애인(20단계를 한 문장으로)

**🎤 음성 결합이 핵심**

```js
const recognition = new webkitSpeechRecognition()
recognition.lang = 'ko-KR'
recognition.onresult = (e) => agent.execute(e.results[0][0].transcript)
// "민원 신청 눌러줘" → 실제로 누름
```

→ **말로 웹을 쓰는 경험.** 데모 임팩트가 엄청납니다.

**비즈니스 모델**

| 타겟 | 상품 | 가격 |
| --- | --- | --- |
| **공공기관 (B2G)** | 접근성 개선 구축 + 유지보수 | 2,000만~1억 / 건 |
| **대기업 웹사이트** | 임베드 솔루션 연간 라이선스 | 연 1,000~5,000만원 |
| **중소 사이트** | SaaS 위젯 | 월 10~50만원 |
| **정부 과제 / R&D** | 국책과제, 사회적기업 지원 | 수천만~수억 (비희석 자금) ⭐ |

**이 영역의 특별한 이점**

1. **경쟁이 거의 없습니다** — 접근성 업체들은 AI 에이전트를 안 씁니다
2. **정부 지원금 접근 가능** — 사회적 가치 + AI = 과제 선정에 매우 유리
3. **언론 노출이 쉽습니다** — "AI로 장애인 웹 접근성 해결"은 기사가 됩니다 → 무료 마케팅
4. **ESG 니즈** — 대기업이 ESG 보고서 거리를 찾고 있습니다
5. **도덕적 명분** — 영업 시 거절하기 어려운 제품

**주의**: 접근성은 정확성이 생명(장애인 사용자가 오클릭 피해를 더 크게 봄) → `ask_user` 적극 활용 / 공공 조달은 느림(6개월~1년) → 민간 레퍼런스 먼저 / 웹 접근성 인증(WA 인증 등) 전문가 제휴 권장

---

### 🧪 5. AI QA 테스팅 도구 ⭐⭐⭐⭐

E2E 테스트의 영원한 문제:

```js
// ❌ 디자이너가 클래스명 하나 바꾸면 전부 깨짐
await page.click('.btn-primary.submit-order-v2')

// ✅ 의도로 작성 → 셀렉터 변경에 안 깨짐
await agent.execute('상품을 장바구니에 넣고 결제까지 진행해')
expect(agent.lastResult.success).toBe(true)
```

**제품 구성**: 자연어 테스트 시나리오 작성(기획자/QA도 작성 가능 ⭐) / CI 통합(GitHub Actions) / 실패 리플레이(`history`에 모든 단계 기록) / 자동 치유(UI가 바뀌어도 AI가 찾아감) / 회귀 리포트

> 실제로 이 저장소에도 `docs/agentic-testing/`이 있어 **프로젝트 스스로 이 용도로 쓰고 있습니다.**

**가격**

```text
Free      : 월 100 테스트 실행
Team      : $99/월  — 월 2,000 실행, CI 통합
Business  : $399/월 — 월 10,000 실행, 병렬 실행, 리플레이 보관
Enterprise: 별도    — 온프레미스, BYO-LLM
```

**리스크**: **비결정성** — 테스트는 "같은 입력 → 같은 결과"여야 하는데 AI는 매번 조금 다릅니다 → **"스모크/회귀 테스트" 용도로 포지셔닝**하고 정밀 단위 테스트는 기존 도구와 병행 권장 / 비용(테스트 1회당 LLM 5~15회 호출 → 저가 모델 + 캐싱 필수) / 기존 플레이어 존재 → **가격과 온프레미스**로 차별화

---

### 🤖 6. 노코드 웹 자동화 서비스 ⭐⭐

Zapier / Make / n8n이 지배하는 시장이지만 **구조적 약점**이 있습니다 — **API가 없는 사이트는 연결할 수 없습니다.** 세상에는 API 없는 웹사이트가 훨씬 많습니다(사내 레거시, 정부 민원, 오래된 ERP, 중소 쇼핑몰 관리자...).

**포지셔닝: "Zapier가 못 붙는 곳에 붙습니다"**

```text
개인     : 월 1.9만원 — 자동화 5개, 월 500실행
비즈니스 : 월 9.9만원 — 자동화 무제한, 월 5,000실행
팀       : 월 29만원  — 팀 공유, 스케줄링, 알림
```

**리스크 (큼)**: "남의 사이트를 자동 조작"은 약관 위반 가능 → 대상 사이트 ToS 확인 필수 / README가 명시적으로 **"server-side automation용이 아니다"** 라고 하므로 **사용자 브라우저에서 돌리는 확장 모델**로 설계해야 함 / 경쟁 극심 → **"API 없는 사이트" 니치에만 집중**해야 승산

---

### 🎓 7. 교육 — 강의 / 코스 / 책 ⭐⭐⭐⭐⭐

**가장 빨리 현금이 나오는 길**

| 상품 | 가격 | 제작 기간 | 예상 |
| --- | --- | --- | --- |
| **유튜브 무료 시리즈** | 0원 | 1~2개월 | 리드 생성 엔진 ⭐ |
| **유료 온라인 강의** (인프런/클래스101/유데미) | 8~15만원 | 2~3개월 | 수강생 500명 → 4,000만~7,500만 |
| **기업 사내 교육** | 1일 200~500만원 | 준비 2주 | 월 2~4회면 월 400~2,000만 |
| **전자책 / PDF 가이드** | 2~5만원 | 3주 | 부수익 |
| **1:1 컨설팅** | 시간당 20~50만원 | — | 고단가, 수탁 리드로 연결 |
| **유료 커뮤니티 / 멤버십** | 월 2~5만원 | — | 안정적 MRR |

**🎯 교육의 진짜 가치: 수탁/SaaS 영업 깔때기**

```text
유튜브 시청자 (무료, 수천~수만)
      ↓
유료 강의 수강생 (수백)
      ↓
"우리 회사에 적용해주세요" 문의 (수십)   ← 1번 수탁 개발로 연결 💰💰💰
      ↓
SaaS 초기 고객 (수개)                    ← 2번 버티컬 SaaS로 연결 💰💰💰💰
```

교육 자체 수익도 괜찮지만, **고단가 수탁 리드를 공짜로 얻는 것**이 더 큽니다.

**콘텐츠 각도**: "AI 에이전트 내부 구조 해부"(661줄로 배우는 ReAct — 희소성) / "회사 업무 자동화 실전"(B2B 교육 수요 직결) / "MCP 서버 만들기"(최고 관심 키워드) / "AI 보안: API 키 노출 사고 막기"

---

### 🇰🇷 8. 한국어화 + 국내 특화 배포판 ⭐⭐⭐⭐

**현재 상태**: `packages/ui/src/i18n/locales.ts`에 `en-US`, `zh-CN` **2개뿐. 한국어 없음.**

**할 일**

1. **한국어 i18n 기여 (업스트림 PR)** — 공식 저장소 기여 이력 확보
2. **한국어 프롬프트 최적화** — 한국 업무 환경에 맞게 튜닝
3. **국내 서비스 특화 패치** — 국내 주요 ERP/그룹웨어/쇼핑몰 관리자의 DOM 특이점 대응 (React/antd 패치처럼)
4. **국내 민감정보 마스킹 룰셋** — 주민번호, 사업자번호, 계좌번호, 전화번호 프리셋
5. **국내 LLM 연동 프리셋** — 네이버 하이퍼클로바X, 업스테이지 Solar 등 (OpenAI 호환 엔드포인트면 바로 됨)

**수익화 경로 (직접 X, 간접 O)**

```text
한국어 배포판 공개 (무료, 오픈소스)
      ↓
"한국에서 이거 제일 잘 아는 사람" 포지션
      ↓
수탁 문의 / 강의 문의 / SaaS 신뢰도   💰💰💰
```

**마케팅 투자**로 보는 게 맞습니다. 2주 투입으로 "국내 1인자" 포지션을 살 수 있습니다.

> ⚠️ README 경고: *"사람의 실질적 개입 없이 봇이나 AI가 전부 생성한 기여는 받지 않는다."* → AI로 초안을 뽑는 건 괜찮지만 **직접 검수하고 실제로 테스트**해서 PR을 내야 합니다.

---

### 🔐 9. LLM 프록시 / 게이트웨이 SaaS ⭐⭐

**니즈는 확실합니다.** Page Agent를 쓰려는 모든 개발자가 같은 벽에 부딪힙니다 — *"API 키를 프론트에 넣으면 털린다. 프록시 서버를 만들어야 하는데... 귀찮다."*

```js
// 고객은 이렇게만
new PageAgent({ baseURL: 'https://gateway.우리서비스.com/v1', apiKey: 'pk_live_xxx' })
```

우리가 해주는 것: 키 은닉, 사용자별 쿼터, 모델 라우팅, 캐싱, 요청/응답 로그, 프롬프트 검열, 비용 대시보드

**추천도가 낮은 이유**: 이미 강력한 플레이어들(OpenRouter, Portkey, Helicone, LiteLLM)이 있어 범용으로는 못 이깁니다.

**승산 있는 각도** — **"Page Agent 전용"** 으로 좁히기: 긴 DOM 프롬프트에 최적화된 캐싱 / **작업 단위**(토큰 아님) 과금·모니터링 / `history` 리플레이 내장 / 모델별 호환 패치를 게이트웨이 레벨에서 처리

→ 단독 사업보다 **3번(Copilot-as-a-Service)의 핵심 부품**으로 만드는 게 맞습니다.

---

### 🧩 10. 플랫폼 플러그인 (WordPress / ERP 마켓) ⭐⭐⭐⭐

직접 고객을 찾는 대신 **이미 수십만 사용자가 있는 플랫폼 마켓**에 올립니다.

| 플랫폼 | 규모 | 상품 | 가격 |
| --- | --- | --- | --- |
| **WordPress** | 전 세계 웹의 40%대 | "AI Copilot for WP Admin" 플러그인 | Free + Pro $49/년 |
| **Shopify** | 상점 수백만 | 상점 관리자 AI 비서 앱 | 월 $19~99 |
| **국내 ERP** (이카운트, 영림원 등) | 중소기업 수만 | 제휴 애드온 | 벤더와 수익 배분 |
| **국내 그룹웨어** (하이웍스, 다우오피스 등) | 수만 사업장 | 결재 자동화 애드온 | 제휴 |
| **Chrome Web Store** | — | 업종별 특화 확장 | Free + 인앱 구독 |

**🥇 가장 유망: WordPress 플러그인** — 설치 기반 압도적 + PHP 플러그인 구조 단순 + 유료 플러그인 생태계 성숙 + 구현이 간단(12번 코드 참조)

가치 예시: WP 관리자에서 *"지난달 주문 중 미배송 건 전부 배송완료로 바꿔"*, *"이 글 카테고리를 정리해줘"* 를 자연어로.

**WP 플러그인 수익 현실**: Freemium 표준(무료로 설치 수 확보 → Pro 전환) / 활성 설치 1만 + 전환율 1~3% → **연 $5,000~15,000** / 상위권은 연 수억원대 / **레버리지: 한 번 만들면 계속 팔림** (수탁과 반대)

**리스크**: 플랫폼 정책 변경 / WP는 지원 문의 많음(별점 관리 부담) / Chrome Web Store의 `<all_urls>` 권한 심사 엄격

---

## 15. 최종 권고 로드맵

```text
┌─ 0~1개월 ─────────────────────────────────────┐
│ ① 더미 데모 사이트 + 자동화 영상 3개 제작       │
│ ② 유튜브 1편 공개 (후킹 편)                    │
│ ③ 한국어 i18n 기여 PR (포지셔닝)               │
│   → 투입: 낮음 / 산출: 신뢰도 + 리드           │
└───────────────────────────────────────────────┘
              ↓
┌─ 1~3개월 ─────────────────────────────────────┐
│ ④ 수탁 PoC 영업 시작 (1번)                     │
│    "1주일에 데모 만들어드립니다"                │
│ ⑤ 유튜브 시리즈 계속 (3편 보안편 필수)          │
│ ⑥ 프록시/마스킹/감사로그 템플릿 자산화          │
│   → 투입: 중간 / 산출: 💰 첫 현금 + 실전 노하우 │
└───────────────────────────────────────────────┘
              ↓
┌─ 3~6개월 ─────────────────────────────────────┐
│ ⑦ 수탁 3~5건 하며 "가장 반복되는 업종" 발견     │
│ ⑧ 그 업종으로 버티컬 SaaS 착수 (2번)            │
│ ⑨ 유료 강의 출시 (7번)                          │
│   → 투입: 높음 / 산출: 💰💰 MRR 시작            │
└───────────────────────────────────────────────┘
              ↓
┌─ 6~12개월 ────────────────────────────────────┐
│ ⑩ 버티컬 SaaS 유료 전환 + 수평 확장            │
│ ⑪ 선택: 접근성(4번) 정부과제 / WP플러그인(10번) │
│   → 산출: 💰💰💰 스케일                        │
└───────────────────────────────────────────────┘
```

### 핵심 원칙 5가지

| # | 원칙 | 이유 |
| --- | --- | --- |
| **1** | **수탁으로 시작해서 SaaS로 끝내라** | 수탁은 현금 + 도메인 지식을 줍니다. SaaS는 수탁 경험 없이 만들면 실패합니다 |
| **2** | **범위를 극단적으로 좁혀라** | "모든 웹 자동화"는 성공률 70%. "출결 입력 하나"는 98%. 팔리는 건 후자 |
| **3** | **보안을 상품으로 만들어라** | API 키 은닉 + 마스킹 + 감사로그. 고객 1순위 질문이고 경쟁사가 소홀한 지점 |
| **4** | **교육은 마케팅 비용이 아니라 수익원이다** | 유튜브 → 리드 → 수탁 → SaaS. 깔때기 전체가 돈이 됩니다 |
| **5** | **"AI가 실수하면?"에 답을 준비하라** | `ask_user` 강제, 허용목록, 감사로그, 계약서 책임 범위. 없으면 엔터프라이즈 계약 불가 |

### ⚠️ 공통 리스크 체크리스트

```text
□ 데모 CDN의 무료 LLM을 상업 사용하고 있지 않은가? (금지)
□ API 키가 프론트엔드에 노출되어 있지 않은가?
□ 개인정보 마스킹이 설정되어 있는가? (주민번호/카드/전화/계좌)
□ 민감 업종이면 로컬/온프레미스 LLM을 쓰는가?
□ 대상 사이트의 약관을 위반하지 않는가?
□ 감사 로그가 남는가? (분쟁 대응)
□ AI 오작동 시 책임 범위가 계약서에 있는가?
□ MIT 라이선스 고지를 포함했는가?
□ LLM 토큰 비용이 과금 수익을 넘지 않는가? (작업당 토큰 상한)
```

---

## 부록: 빠른 참조

### 주요 파일 위치

| 보고 싶은 것 | 파일 |
| --- | --- |
| 시스템 프롬프트 (프롬프트 엔지니어링 교재) | `packages/core/src/prompts/system_prompt.md` |
| 에이전트 루프 (661줄) | `packages/core/src/PageAgentCore.ts` |
| 도구 9개 정의 | `packages/core/src/tools/index.ts` |
| 설정 옵션 전체 | `packages/core/src/types.ts` |
| DOM 추출 엔진 (1,745줄) | `packages/page-controller/src/dom/dom_tree/index.js` |
| DOM → 텍스트 압축 | `packages/page-controller/src/dom/index.ts` |
| React / antd 우회 패치 | `packages/page-controller/src/patches/` |
| LLM 클라이언트 | `packages/llms/src/OpenAIClient.ts` |
| 지원 모델 목록 | `packages/website/src/pages/docs/features/models/page.tsx` |
| MCP 서버 (3파일) | `packages/mcp/src/` |
| AI 코딩 스킬 5개 | `.agents/skills/` |
| 기여 가이드 | `CONTRIBUTING.md`, `AGENTS.md`, `docs/developer-guide.md` |
| 체인지로그 | `docs/CHANGELOG.md` |

### 개발 명령어

```bash
npm start          # 문서 웹사이트 dev 서버
npm run dev:demo   # 데모 빌드 watch + localhost:5174
npm run dev:ext    # 확장 개발모드
npm run build      # 전체 빌드
npm run build:libs # 라이브러리만 빌드
npm run build:ext  # 확장 zip 패키징
npm run typecheck  # 타입 검사
npm test           # 테스트 (현재 llms 패키지만)
npm run test:live  # 실제 API 호출 테스트 (CI 제외)
npm run lint       # ESLint
npm run ci         # CI 전체
npm run cleanup    # dist / .output 삭제
```

### 프로젝트 철학 (AGENTS.md 발췌)

- 모든 변경은 기능 구현만이 아니라 **코드베이스 품질도 개선**해야 한다
- 모든 코드와 주석은 **영어**로
- **에러나 위험을 숨기려 하지 마라.** 개발자와 사용자에게 가치 있는 피드백이므로 **보이게, 조치 가능하게** 만들어라
- **추적가능성과 예측가능성이 성공률보다 중요하다**
- 테스트: 중복 테스트나 구현 코드를 그대로 단정문으로 옮기는 테스트를 추가하지 마라. **의미 있는 행위와 현실적인 실패 모드**를 테스트하라
