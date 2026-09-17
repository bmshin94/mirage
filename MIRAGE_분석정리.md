# Mirage 분석 정리

> Mirage 레포지토리를 직접 분석하고 정리한 문서입니다.
> 작성일: 2026-09-17

## 📎 링크

| 구분 | 주소 |
| --- | --- |
| 원본 레포 | https://github.com/strukto-ai/mirage |
| 현재 포크 | https://github.com/bmshin94/mirage |
| 공식 문서 | https://docs.mirage.strukto.ai |
| 제작사 | https://www.strukto.ai/mirage |
| Python 패키지 | https://pypi.org/project/mirage-ai/ |
| npm 패키지 | https://www.npmjs.com/package/@struktoai/mirage-node |
| Discord | https://discord.gg/u8BPQ65KsS |
| 한국어 README | [readme/README.ko.md](readme/README.ko.md) |

---

## 1. Mirage가 뭔가

**"AI 에이전트용 가상 터미널"** — 세상 모든 데이터를 폴더처럼 마운트해서, AI가 `ls`, `grep`, `cat` 같은
유닉스 명령어로 다룰 수 있게 해주는 오픈소스 라이브러리.

기존에는 서비스마다 SDK와 MCP를 따로 붙여야 했지만, Mirage는 전부 **하나의 파일시스템**으로 통일한다.

```python
from mirage import Workspace
from mirage.resource.ram import RAMResource
from mirage.resource.s3 import S3Config, S3Resource

ws = Workspace({
    "/data": RAMResource(),
    "/s3":   S3Resource(S3Config(bucket="my-bucket")),
})

await ws.execute("cp /s3/report.csv /data/report.csv")
await ws.execute("grep alert /s3/data/log.jsonl | wc -l")
await ws.snapshot("demo.tar")
```

비유하자면 **USB C타입 어댑터**. 기기마다 다르던 케이블을 하나의 규격으로 통일하듯,
Slack·Drive·DB를 전부 "폴더"라는 하나의 규격으로 통일한다.

### 4개의 핵심 기둥

| 기둥 | 설명 |
| --- | --- |
| **1. 가상 파일시스템 (VFS)** | S3, GDrive, Slack, Gmail, Notion, Postgres, Redis, GitHub 등 **63개 리소스**를 한 루트 아래 마운트 |
| **2. 가상 CLI** | `git`, `slack`, `ntn`(Notion), `gh`, `linear` 등을 Mirage가 직접 흉내내서 응답. 실제 프로그램 설치 불필요 |
| **3. 가상 런타임** | 명령을 in-process(Monty), WASM(Pyodide), Docker, E2B, SSH 원격 등으로 라우팅. 스토리지와 연산 분리 |
| **4. 보안 / 정책 엔진** | `allow`/`ask`/`deny`로 명령 통제, `hide`/`show`로 파일 통제. **숨긴 파일은 "존재하지 않는 것"으로 보임** |

---

## 2. 폴더 구조

```
mirage/
├── python/mirage/        # 파이썬 본체 (pip: mirage-ai)
│   ├── core/             # 63개 백엔드 (s3, slack, notion, postgres, gmail...)
│   ├── commands/         # ls, grep, find, du, tar, zip 등 POSIX 명령어 구현
│   ├── commands/cli/     # git, ntn, slack 등 가상 CLI
│   ├── fuse/             # 실제 OS 마운트 (macOS FSKit, Linux FUSE, Windows WinFsp)
│   ├── policy/           # 정책 엔진 (rule, profile, script, decisions)
│   ├── observe/          # 모든 명령 기록 (observer, record, disk_store, redis_store)
│   ├── watch/            # 변경 감지 & 이벤트
│   ├── secrets/          # env, dotenv, AWS Secrets Manager, 1Password
│   ├── agents/           # LangChain / OpenAI Agents / Claude SDK / Pydantic AI 어댑터
│   └── runtime/, shell/, workspace/, server/
├── typescript/packages/  # TS 쌍둥이 구현 (core / node / browser / agents / cli / dsh)
├── examples/python/      # 64개 예제 폴더
├── integ/, conformance/, spec/   # 실제 GNU 도구와 1:1 대조 테스트
└── plugins/mirage/       # Claude Code / Codex 플러그인 + MCP 서버 설정
```

**특징:** Python과 TypeScript를 완전히 대칭 구조로 유지하며,
`scripts/check_layout_parity.py`가 CI에서 두 언어의 모듈 구조 차이를 검사한다.

---

## 3. 설치 및 사용법

### 설치

```bash
# 파이썬 (Python >= 3.11)
uv add mirage-ai

# 타입스크립트 (Node.js >= 20)
npm install @struktoai/mirage-node
npm install @struktoai/mirage-browser   # 브라우저 / 엣지 런타임
npm install @struktoai/mirage-agents    # 에이전트 프레임워크 어댑터

# CLI
npm install -g @struktoai/mirage-cli
# 또는
curl -fsSL https://strukto.ai/mirage/install.sh | sh
```

### 사용 모드 3가지

**① 코드에 심기 (가장 일반적)**

```python
ws = Workspace({"/data": RAMResource()})
await ws.execute("echo hello > /data/a.txt")
```

**② CLI로 사용**

```bash
mirage workspace create ./workspace.yaml --id myws
mirage execute -w myws -c 'ls /slack'
mirage daemon stop
```

**③ 실제 폴더로 마운트 (FUSE)** — macOS/Linux에서 탐색기로 직접 열람 가능

### 추천 학습 순서

1. `RAMResource` 예제 (토큰 불필요)
2. `disk` / `glob` 예제 (토큰 불필요)
3. 실제 서비스 연결

---

## 4. 플러그인? 스킬? MCP?

**전부 다 해당한다.** 본체는 라이브러리이고, 나머지는 포장지다.

| 형태 | 실제 위치 | 설명 |
| --- | --- | --- |
| **라이브러리** | `python/mirage/`, `typescript/packages/` | **본체** |
| **MCP 서버** | `mirage mcp` | AI에게 도구 6개 제공: `execute_command`, `read`, `write`, `edit`, `ls`, `grep` |
| **플러그인** | `plugins/mirage/plugin.json` | Claude Code / Codex에 설치하는 패키지 |
| **스킬** | `plugins/mirage/skills/mirage-filesystem/SKILL.md` | 플러그인 안에 포함된 사용 설명서 |
| **CLI** | `mirage` 명령어 | 터미널에서 직접 사용 |
| **HTTP 서버** | `mirage daemon` | REST API 제공 |

MCP는 "AI와 연결하는 방법 중 하나"일 뿐이고, Mirage 자체는 그보다 큰 라이브러리다.

---

## 5. API 토큰이 필요한가

### 토큰 불필요

- `ram` — 메모리 가상 디스크
- `disk` — 로컬 폴더
- `opfs` — 브라우저 저장소

→ 연습과 개발은 토큰 없이 가능하다.

### 토큰 필요

S3, Slack, Notion, Gmail, GitHub, Postgres 등 외부 서비스는 각각의 인증이 필요하다.

### 시크릿 관리 (`python/mirage/secrets/`)

| 방식 | 파일 |
| --- | --- |
| 환경변수 | `env.py` |
| `.env` 파일 | `dotenv.py` |
| AWS Secrets Manager | `aws.py` |
| 1Password | `onepassword.py` |

코드에 토큰을 하드코딩하지 않아도 되며, Mirage 데몬 자체 인증용 토큰(`MIRAGE_TOKEN`)도 별도로 존재한다.

---

## 6. 왜 GitHub에서 인기가 많은가

**원본 레포 기준 ⭐ 3.6k, 포크 269개** (Preview 단계 기준으로는 매우 높은 수치)

1. **타이밍** — "에이전트에게 회사 데이터를 어떻게 안전하게 줄 것인가"라는 현재 최대 과제를 정조준
2. **직관적 컨셉** — "MCP 서버 50개 대신 폴더로 마운트해라"가 한 번에 이해됨
3. **넓은 지원 범위** — 리소스 63개, 예제 64개, Python + TypeScript 동일 구현
4. **높은 완성도** — `grep`/`du`/`tar`를 Docker(`debian:stable-slim`)로 실제 GNU 도구와 출력까지 대조하고,
   Notion CLI(`ntn`)는 실제 npm 바이너리와 동일한 명령줄을 비교 검증
5. **낮은 진입장벽** — Apache-2.0 + `pip install` 한 줄

---

## 7. 로컬 에이전트 구축에 도움이 되는가

**매우 도움이 된다.**

### 이미 준비된 어댑터 (`python/mirage/agents/`)

`claude_agent_sdk`, `openai_agents`, `langchain`, `pydantic_ai`, `agno`, `openhands`, `camel`, `mcp`

### 로컬 에이전트에 특히 좋은 이유

| 이유 | 설명 |
| --- | --- |
| 데이터 유출 방지 | 로컬 디스크 마운트 시 내부에서 모두 처리 |
| 샌드박스 내장 | 에이전트의 `rm -rf`도 가상 파일시스템 안에서만 발생 |
| 숨김 기능 | `.env`, `~/.ssh` 등을 아예 안 보이게 |
| 토큰 절약 | 파일 전체 대신 `grep`으로 필요한 줄만 |
| 스냅샷 | `ws.snapshot("demo.tar")`로 작업 상태 전체 저장/복원 |
| 버전 관리 | HTTP API에 `commit`, `branch`, `diff`, `checkout` 제공 |

### 실제 데모 시나리오 (`examples/python/demo/design_feedback.py`)

> Slack 장애 채널의 사용자 피드백을 읽고 → GitHub 레포에서 관련 코드를 찾아 → Linear에 디자인 이슈 등록

---

## 8. React / PHP로 만들 수 있는가

### React — 가능 (구조 주의)

브라우저 전용 패키지 `@struktoai/mirage-browser`가 실제로 존재한다.

**하지 말아야 할 구조:** React(브라우저)에서 직접 토큰 사용 → 토큰이 클라이언트에 노출됨

**권장 구조:**

```
┌─────────────┐        ┌──────────────────┐
│   React     │ ─────> │ Node/Python 서버  │ ──> Slack, S3, DB...
│  (화면만)    │  API   │  (Mirage 본체)    │
└─────────────┘        └──────────────────┘
      토큰 없음               토큰 보관
```

### PHP — 직접 SDK는 없지만 REST API로 가능

Mirage 데몬의 HTTP 엔드포인트 (`python/mirage/server/routers/`):

```
POST   /v1/workspaces                   워크스페이스 생성
POST   /v1/execute                      명령 실행
GET    /v1/workspaces/{id}
POST   /v1/sessions
POST   /v1/workspaces/{id}/commit       버전 커밋
GET    /v1/workspaces/{id}/diff         변경 비교
POST   /v1/workspaces/{id}/snapshot     스냅샷
```

```php
$ch = curl_init('http://127.0.0.1:8765/v1/execute');
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Authorization: Bearer ' . getenv('MIRAGE_TOKEN'),
]);
curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode([
    'workspace_id' => 'myws',
    'command'      => 'grep -rln 결제오류 /slack',
]));
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$result = curl_exec($ch);
```

> 위 PHP 예시는 라우터 파일에서 확인한 엔드포인트 기반으로 작성한 것으로,
> 실제 요청 필드명은 직접 호출해보며 확인이 필요하다.

### 데몬 환경변수

| 변수 | 기본값 | 용도 |
| --- | --- | --- |
| `MIRAGE_DAEMON_URL` | `http://127.0.0.1:8765` | CLI가 접속할 데몬 주소 |
| `MIRAGE_TOKEN` | (없음) | CLI가 데몬에 보내는 Bearer 토큰 |
| `MIRAGE_AUTH_MODE` | `local` | `local`, `token`, `jwt` |
| `MIRAGE_ALLOWED_HOSTS` | `127.0.0.1,localhost,::1` | Host 헤더 허용 목록 |
| `MIRAGE_HOME` | `~/.mirage` | 설정/상태 루트 |

---

## 9. 수익화 아이디어

### 전제 조건

**① 라이선스: Apache-2.0**

| 가능 | 금지 |
| --- | --- |
| 상업적 판매 | "Mirage" 상표 사용 (§6 상표 조항) |
| 수정 후 배포 | 저작권/라이선스 고지 삭제 |
| 자체 코드 비공개 | |
| 특허 사용권 포함 | |

`CONTRIBUTING.md`에 CLA(저작권 양도 계약)가 없어 기여해도 저작권을 유지한다.

**② 아직 Preview 단계** — API 변경 가능성이 있으므로 버전 고정 필수. 반대로 선점 기회이기도 하다.

**③ 원조 회사와 겹치지 않는 영역을 노려야 한다** — 한국 시장이 그 지점.

---

### 아이디어 1: 한국형 커넥터 (최우선 추천)

리소스 63개를 전수 확인한 결과 **한국 서비스 리소스는 0개**다.

미지원 목록: 카카오톡/카카오워크, 네이버웍스, 잔디(JANDI), 더존/이카운트,
네이버 클라우드, 토스페이먼츠, 아임포트, 카페24/고도몰, HWP(한글)

**제작 난이도:** `examples/python/other/custom_resource.py`가 전체 243줄.
async 함수 4개(`readdir`, `read_bytes`, `stat`, `write`)만 구현하면
`ls`, `cat`, `grep`, `find`, `head`, `wc` 등 모든 제네릭 명령이 자동 동작한다.

**수익 구조 3단계**

1. 오픈소스 기여 (무료) → "한국 커넥터 제작자" 인지도 확보
2. 인지도 → 문의 유입
3. 유료 전환 — 구축 대행(건당), 프리미엄 커넥터, 유지보수 구독

**공개/비공개 분리 전략**

| 공개 (유입용) | 비공개 유료 (수익) |
| --- | --- |
| 카카오워크, 잔디 기본 커넥터 | 더존/이카운트 ERP 커넥터 |
| 네이버웍스 읽기 전용 | HWP 파서 |
| | 커스텀 정책/보안 모듈 |

HWP 커넥터는 공공기관·대기업 수요가 확실한 반면 해외 업체가 만들 이유가 없는 영역이다.

---

### 아이디어 2: 업종 특화 AI 비서 SaaS

Mirage가 배관(plumbing)을 담당하고, UI와 도메인 로직으로 차별화한다.

| 업종 | 마운트 대상 | 킬러 기능 | 지불 의사 |
| --- | --- | --- | --- |
| 법무/노무 | 판례DB + 계약서 드라이브 + 메일 | "이 계약서 독소조항 찾기" | 매우 높음 |
| 이커머스 | 상품DB + CS채팅 + 재고시트 | "이번주 CS 불만 유형 TOP5" | 높음 |
| 병원/의원 | 예약시스템 + 문서 | "이번달 노쇼 패턴 분석" | 중간 |
| 세무/회계 | 더존 + 영수증 + 메일 | "누락 증빙 찾기" | 매우 높음 |

**가격 설계 예시 (추정치 — 실제로는 고객 인터뷰 후 결정 필요)**

- 스타터: 월 9~19만원 (마운트 3개, 사용자 5명)
- 프로: 월 39~79만원 (마운트 10개, 사용자 20명, 정책 설정)
- 엔터프라이즈: 별도 견적 (온프레미스, 커스텀 커넥터)

**핵심 차별점: 온프레미스**
한국 기업(금융·공공·의료)의 절대 조건인 "데이터 외부 반출 금지"를 만족한다.
Mirage는 로컬에서 전부 동작하므로 "당신 회사 서버 안에서만 동작합니다"가 성립한다.

---

### 아이디어 3: SI 구축 + 유지보수 (현금흐름 최고)

| 작업 | 기존 방식 | Mirage 사용 |
| --- | --- | --- |
| 슬랙 연동 | 3일 | 30분 |
| 드라이브 연동 | 3일 | 30분 |
| DB 연동 | 2일 | 30분 |
| 권한 제어 | 1주 | 설정 파일 몇 줄 |
| **합계** | **3~4주** | **2~3일** |

**견적 구조 예시 (추정)**

- 초기 구축비: 500만 ~ 3,000만원 (규모별)
- 월 유지보수: 30만 ~ 200만원/월
- 추가 커넥터: 건당 200~500만원

**주의:** SI는 시간을 파는 구조라 확장이 어렵다.
반드시 재사용 가능한 템플릿으로 정리해야 이후 SaaS로 전환할 수 있다.

---

### 아이디어 4: AI 에이전트 감사·거버넌스 도구

레포에 실제 근거가 되는 모듈이 존재한다.

- `python/mirage/observe/` — 모든 명령을 타임스탬프와 함께 기록
- `python/mirage/policy/` — allow/ask/deny 정책 엔진
- `python/mirage/watch/` — 변경 감지 & 이벤트
- HTTP API — `commit`, `branch`, `diff`, `checkout` 버전 관리

**제품 컨셉:** "우리 회사 AI가 무엇을 보고 무엇을 건드렸는가"를 보여주는 감사 대시보드
(실행 명령 수, 접근 파일, 차단된 시도, 고위험 작업, 감사 로그 CSV, 시점 롤백)

**타겟:** 금융·의료·공공 (감사 대응이 법적 의무)
**타이밍:** 규제 강화에 따라 1~2년 후 수요 폭발 예상 → 아이디어 2에 얹는 기능으로 시작하는 것이 현실적

---

### 아이디어 5: 매니지드 호스팅 (비추천)

"Mirage를 Vercel처럼" 제공하는 모델이나, 오픈소스 회사의 표준 수익 모델이라
원조 회사(strukto.ai)가 직접 할 가능성이 높다.

**확인 필요:** strukto.ai 홈페이지에 Pricing 메뉴가 있는지 확인할 것.
(이번 분석에서는 네트워크 차단으로 확인하지 못함)

**예외:** "한국 리전 + 국내 컴플라이언스 특화"(CSAP, 망분리, 세금계산서)는
해외 업체가 대응하기 어려운 규제 기반 해자가 된다.

---

### 아이디어 6: 교육 & 콘텐츠 (지금 시작 가능)

| 방식 | 예상 수익 | 실제 가치 |
| --- | --- | --- |
| 블로그/유튜브 한국어 가이드 | 거의 없음 | 검색 선점 |
| 인프런/유데미 강의 | 월 수십~수백만원 | 신뢰도 |
| 기업 워크샵 | 회당 100~300만원 | SI 고객 발굴 |
| 유료 뉴스레터 | 소액 | 리드 확보 |

한국어 자료가 README 번역본 하나뿐이므로 선점 기회가 크다.

---

### 추천 로드맵

```
0~1개월 - 씨 뿌리기
  - 예제 실행하며 Mirage 숙달
  - 한국어 기술 블로그 3~5편 (SEO 선점)
  - 카카오워크 또는 잔디 커넥터 프로토타입

2~3개월 - 이름 알리기
  - 커넥터 오픈소스 공개 + 기여 PR
  - 커뮤니티 공유
  - 지인 회사 1곳 무료/저가 구축 → 레퍼런스 확보

4~6개월 - 수익화
  - SI 프로젝트 수주
  - 공통 부분 템플릿화
  - 업종 1개 선정 후 SaaS 설계

7~12개월 - 확장
  - 업종 특화 SaaS 베타 런칭
  - 프리미엄 커넥터 판매 (HWP, 더존)
  - 감사 대시보드 기능 추가
```

**전략: SI로 현금을 벌면서 SaaS로 자산을 만든다.**

---

### 하지 말아야 할 것

| 항목 | 이유 |
| --- | --- |
| Mirage 그대로 리패키징해서 판매 | 상표 위반 + 부가가치 없음 |
| 매니지드 호스팅으로 정면승부 | 원조 회사에 열세 |
| Preview 버전을 버전 고정 없이 프로덕션 투입 | API 변경 시 장애 |
| 63개 리소스 전부 지원한다고 영업 | 실제 검증한 것만 약속해야 함 |
| 처음부터 완벽한 SaaS 제작 | 고객 검증 없이 시간 소모 |

---

### 아이디어 우선순위 종합

| 아이디어 | 추천도 | 착수 시기 | 수익 속도 |
| --- | --- | --- | --- |
| 1. 한국형 커넥터 | ★★★★★ | 지금 | 중간 |
| 2. 업종 특화 SaaS | ★★★★☆ | 4개월 후 | 느림 (규모 최대) |
| 3. SI 구축 | ★★★★☆ | 2개월 후 | 빠름 |
| 4. 감사 대시보드 | ★★★☆☆ | 곁다리로 | 나중 |
| 5. 매니지드 호스팅 | ★★☆☆☆ | 비추천 | - |
| 6. 콘텐츠/교육 | ★★★☆☆ | 지금 | 느림 (영업채널) |

**결론: 1번 + 6번을 동시에 시작 → 3번으로 현금 확보 → 2번으로 확장**

---

## 10. 전체 요약

| 질문 | 답 |
| --- | --- |
| 설치 | `uv add mirage-ai` 또는 `npm install @struktoai/mirage-node` |
| 정체 | 라이브러리가 본체, MCP/플러그인/스킬/CLI는 포장지 |
| 토큰 | 로컬(ram/disk)은 불필요, 외부 서비스는 필요. 1Password·AWS 연동 지원 |
| 인기 이유 | ⭐3.6k. 타이밍 + 직관적 컨셉 + 높은 완성도 |
| 로컬 에이전트 | 최적. 어댑터 8개 + 샌드박스 + 스냅샷 + 버전관리 |
| 수익화 | 한국형 커넥터 → 업종별 AI SaaS 루트 추천 |
| React/PHP | React 가능(서버 분리 필수), PHP는 REST API 경유 |
