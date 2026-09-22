# Agent Governance Toolkit (AGT) 분석 정리

> 이 문서는 Claude Code 세션에서 진행한 AGT 레포지토리 전수조사 및 Q&A 내용을 정리한 것입니다.
>
> - 작성일: 2026-09-22
> - 분석 대상 버전: **v5.0.0 (Public Preview)**
> - 라이선스: **MIT**

## 🔗 GitHub 주소

| 구분 | 주소 |
|---|---|
| **원본 레포 (Upstream)** | https://github.com/microsoft/agent-governance-toolkit |
| **포크 레포 (작업 대상)** | https://github.com/bmshin94/agent-governance-toolkit |
| **공식 문서** | https://microsoft.github.io/agent-governance-toolkit |
| **PyPI** | https://pypi.org/project/agent-governance-toolkit/ |
| **npm (TypeScript SDK)** | https://www.npmjs.com/package/@microsoft/agent-governance-sdk |
| **NuGet (.NET)** | https://www.nuget.org/packages/Microsoft.AgentGovernance |
| **crates.io (Rust)** | https://crates.io/crates/agent-governance |
| **Discord 커뮤니티** | https://discord.gg/TxMRqY3pFr |

---

## 1. 이게 뭐하는 프로젝트인가

### 한 줄 요약
Microsoft가 만든 **"자율 AI 에이전트를 위한 정책 집행 · 신원 · 샌드박싱 · SRE 툴킷"** 입니다.

### 해결하려는 문제
AI 에이전트가 툴을 호출하고, 웹을 탐색하고, DB를 조회하고, 다른 에이전트에게 위임하기 시작하면
아래 세 가지 질문에 답할 수 없게 됩니다.

| 질문 | 기존 방식의 한계 |
|---|---|
| **이 동작이 허용되는가?** | OAuth 스코프 / IAM 역할은 "어떤 서비스에 접근 가능한가"만 통제하고, "접속 후 무엇을 하는가"는 통제하지 못함 |
| **어떤 에이전트가 했는가?** | 다중 에이전트가 API 키 하나를 공유하면 "에이전트가 했다"는 말은 사고 대응이 될 수 없음 |
| **무슨 일이 있었는지 증명 가능한가?** | 감사·규제 대응에는 변조 불가능한(tamper-evident) 결정 기록이 필요 |

### 핵심 철학
프롬프트 수준의 안전성("규칙을 따르세요")은 통제 수단이 아니라 **확률론적 시스템에 대한 정중한 부탁**일 뿐입니다.

- OWASP LLM01:2025 — "프롬프트 인젝션을 완벽히 방지하는 방법이 존재하는지 불분명하다"
- Andriushchenko et al. (ICLR 2025) — GPT-4o / GPT-3.5 / Claude 3 / Llama-3 대상 적응형 공격 **성공률 100%**
- Microsoft AI Red Teaming — "완화 조치가 위험을 완전히 제거하지는 않는다"

따라서 AGT는 프롬프트 안에서 싸우지 않고, **모델의 의도가 실행되기 전 결정론적 애플리케이션 코드에서 가로챕니다.**
AGT 커널이 거부한 동작은 "가능성이 낮은" 것이 아니라 **구조적으로 불가능**합니다.

---

## 2. 레포 구조 (실측)

### 규모
| 항목 | 수치 |
|---|---|
| 전체 추적 파일 | **4,813개** |
| Python 파일 / 라인 | 1,930개 / 약 29,780줄 |
| TypeScript(.ts/.tsx) / 라인 | 347개 / 약 72,197줄 |
| Rust / 라인 | 104개 / 약 39,361줄 |
| Markdown 문서 / 라인 | 910개 / 약 172,029줄 |
| C# | 159개 |
| Rego(OPA) 정책 | 130개 |
| GitHub Actions 워크플로 | **43개** |

### 디렉터리 개요
```
agent-governance-toolkit/
├── agent-governance-python/       # 메인 구현 (풀스택)
├── agent-governance-typescript/   # TS SDK + VS Code 확장(agent-os-vscode)
├── agent-governance-rust/         # Rust 구현
├── agent-governance-dotnet/       # .NET 구현
├── agent-governance-golang/       # Go 구현
├── agent-governance-claude-code/  # Claude Code 거버넌스 플러그인
├── agent-governance-copilot-cli/  # GitHub Copilot CLI 연동
├── agent-governance-opencode/     # OpenCode 연동
├── agent-governance-antigravity-cli/
├── policy-engine/                 # ACS: Rust 기반 정책 판정 런타임
├── docs/                          # 문서 (ADR 29개, 명세 10개, i18n 4개 언어)
├── examples/                      # 실전 예제 41개
├── tests/ benchmarks/ schemas/ scripts/
└── .github/workflows/             # CI 43종
```

### Python 핵심 패키지
| 패키지 | 역할 |
|---|---|
| **Agent OS** | 정책 엔진, 에이전트 생명주기, 거버넌스 게이트 |
| **Agent Control Specification (policy-engine)** | 무상태·결정론적·fail-closed 정책 판정 런타임 (Rust 코어) |
| **Agent Mesh** | 에이전트 탐색, 라우팅, 신뢰 메시 |
| **Agent Runtime** | 실행 샌드박싱 (특권 링 4단계) |
| **Agent SRE** | 킬 스위치, SLO 모니터링, 카오스 테스트 |
| **Agent Compliance** | OWASP 검증, 정책 린팅, 무결성 검사 |
| **Agent Marketplace** | 플러그인 거버넌스 및 신뢰 점수 |
| **Agent Lightning** | 강화학습 훈련 거버넌스, 위반 페널티 |
| **Agent Hypervisor** | 실행 감사, 델타 엔진, 명령어 denylist 강제 |
| **Agent Discovery** | Shadow AI 탐지 (미등록 에이전트 발견) |

### 동작 흐름
```
Agent ──► Policy Engine ──► Identity ──► Audit Log
            (YAML/OPA/Cedar)  (SPIFFE/DID/mTLS)  (Tamper-evident)
                 │                                      │
                 ├── Allowed ──► Tool executes          │
                 └── Denied  ──► GovernanceDenied       ▼
                                                 Decision Record
```
모든 레이어는 선택 사항이며, `govern()` 하나로 시작해 위험도에 따라 레이어를 추가하는 구조입니다.

---

## 3. 가장 쉬운 사용 예시

```python
from agentmesh.governance import govern

safe_tool = govern(my_tool, policy="policy.yaml")   # 모든 호출 검사 · 기록 · 강제
```

```yaml
# policy.yaml
apiVersion: governance.toolkit/v1
name: production-policy
default_action: allow
rules:
  - name: block-destructive
    condition: "action.type in ['drop', 'delete', 'truncate']"
    action: deny
    description: "Destructive operations require human approval"
```

```python
>>> safe_tool(action="read", table="users")
{'table': 'users', 'rows': 42}

>>> safe_tool(action="drop", table="users")
GovernanceDenied: Action denied by policy rule 'block-destructive'
```

---

## 4. Claude Code 플러그인 상세

### 구성
```
agent-governance-claude-code/
├── hooks/
│   ├── hooks.json              # SessionStart / UserPromptSubmit / PreToolUse 등록
│   ├── session-start.mjs
│   ├── user-prompt-submit.mjs
│   └── pre-tool-use.mjs        # 실제 차단 지점
├── lib/
│   ├── policy.mjs              # 정책 로딩 및 평가
│   ├── audit.mjs               # 감사 로그(롤오버 지원)
│   └── poisoning.mjs           # 프롬프트 오염 탐지
├── server/agt-mcp.mjs          # MCP 서버 (JSON-RPC)
├── commands/                   # /agt-status, /agt-check
├── config/default-policy.json  # 기본 정책
└── test/                       # node --test 기반 테스트 5종
```

### 기본 정책이 차단하는 것 (`config/default-policy.json`)
- `rm -rf` 재귀 삭제
- `curl ... | bash`, `wget ... | sh`, `bash <(curl ...)` 형태의 다운로드-즉시실행
- 클라우드 메타데이터 엔드포인트 접근 (`169.254.169.254`, `100.100.100.200`, `metadata.google.internal`) — SSRF 자격증명 탈취 방어
- 자격증명 파일 읽기: `.env`, `id_rsa`, `id_ed25519`, `~/.ssh`, `~/.aws`, `~/.azure`, `~/.kube/config`, `.npmrc`, `.pypirc`, `/proc/<pid>/environ` 등
- `printenv`, `env`, `Get-ChildItem Env:` 등 환경변수 덤프
- 지속성 경로 쓰기 검토(review): `.bashrc`, `.zshrc`, `.gitconfig`, `.git/hooks`, `.vscode/tasks.json`
- 프롬프트 인젝션 문구: "ignore previous instructions", "reveal the system prompt"

### 안전 설계
- `mode: "enforce"`, `denyOnPolicyError: true` → **정책 평가 실패 시 무조건 거부(fail-closed)**
- `defaultEffect: "review"` → 명시되지 않은 툴은 기본적으로 검토 대상
- `minimumPromptDefenseGrade: "B"`

### MCP 툴
| 툴 | 역할 |
|---|---|
| `agt_policy_status` | 현재 세션의 거버넌스 정책 상태 조회 |
| `agt_policy_check_text` | 임의 텍스트의 프롬프트 인젝션 여부 검사 |

---

## 5. 설치 및 사용법

### Claude Code 플러그인
```text
/plugin marketplace add microsoft/agent-governance-toolkit
/plugin install agt-governance@agent-governance-toolkit
```
로컬 디렉터리로 직접 사용:
```bash
claude --plugin-dir ./agent-governance-claude-code
```
환경변수(선택):
```bash
export AGT_CLAUDE_POLICY_PATH=./my-policy.json
export AGT_CLAUDE_AUDIT_PATH=./agt-audit.log
```
슬래시 커맨드: `/agt-status`, `/agt-check`

### Python
```bash
pip install "agent-governance-toolkit[full]"   # [full] 필수
agt doctor                                     # 설치 점검
agt verify                                     # OWASP 컴플라이언스 검사
agt verify --evidence ./agt-evidence.json --strict
agt red-team scan ./prompts/ --min-grade B     # 프롬프트 인젝션 감사
agt lint-policy policies/                      # 정책 검증
```

### 기타 언어
```bash
npm install @microsoft/agent-governance-sdk        # TypeScript
dotnet add package Microsoft.AgentGovernance       # .NET
cargo add agent-governance                         # Rust
go get github.com/microsoft/agent-governance-toolkit/agent-governance-golang
```

### 요구 사항
- Python 3.11+ (문서 일부는 3.10+ 표기)
- Node.js 18+ / npm 9+ (TS SDK), **Claude Code 플러그인은 Node 22+**
- .NET 8+ / Go 1.25+ / Rust 1.70+

---

## 6. 플러그인인가, 스킬인가, MCP인가?

**세 가지 중 하나가 아니라, 여러 배포 형태를 동시에 가진 툴킷입니다.**

| 형태 | 제공 | 근거 |
|---|:---:|---|
| Claude Code 플러그인 | ✅ | `.claude-plugin/marketplace.json` (플러그인명 `agt-governance`) |
| MCP 서버 | ✅ | `server/agt-mcp.mjs` — `tools/list`, `tools/call` 구현 |
| Hooks | ✅ | `hooks/hooks.json` — SessionStart / UserPromptSubmit / PreToolUse |
| 슬래시 커맨드 | ✅ | `commands/agt-status.md`, `commands/agt-check.md` |
| Claude Code Skill(SKILL.md) | ❌ | 해당 형식 없음 |
| SDK 라이브러리 | ✅ | PyPI / npm / NuGet / crates.io / Go 모듈 |
| CLI | ✅ | `agt` |
| GitHub Action | ✅ | `action/action.yml`, `.github/actions/contributor-check/` |
| VS Code 확장 | ✅ | `agent-governance-typescript/agent-os-vscode/` |
| 컨테이너 | ✅ | `Dockerfile`, `docker-compose.yml` |

결론: **"MCP 서버를 포함한 Claude Code 플러그인"** 이 가장 정확한 표현이며,
실제 차단 기능의 핵심은 MCP가 아니라 **PreToolUse 훅**입니다.

---

## 7. API 토큰이 필요한가?

**기본 기능은 토큰이 전혀 필요 없습니다.**

Claude Code 플러그인 코드 전체를 확인한 결과:
- 외부 `fetch()` / HTTPS 요청: **0건**
- 사용 환경변수: `AGT_CLAUDE_POLICY_PATH`, `AGT_CLAUDE_AUDIT_PATH` **2개뿐** (둘 다 로컬 경로)
- 정책 판정은 정규식 + JSON 규칙 매칭으로 **완전 로컬·오프라인** 수행

장점: 비용 0원 / 오프라인 동작 / 코드 외부 유출 없음 / 지연시간 최소

**선택 기능에서만 인증 정보가 필요합니다.**

| 기능 | 필요 항목 |
|---|---|
| Azure 연동 기능 | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET` |
| 예제 속 실제 LLM 에이전트 | `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` (AGT가 아니라 예제 에이전트가 사용) |
| Azure 샌드박스 | Azure 구독 |

---

## 8. 왜 GitHub에서 주목받는가

레포 내부에서 확인 가능한 근거 기준 정리입니다. (별 개수 등 외부 지표는 미확인)

1. **Microsoft 공식 프로젝트** — `microsoft/` 오가니제이션, 공식 패키지 레지스트리 배포
2. **시점** — 에이전트 도입 급증 + EU AI Act 등 규제 압박이 겹친 시기에 "통제" 공백을 겨냥
3. **엔지니어링 완성도** — RFC 2119 정식 명세 10개, 적합성 테스트 992개, ADR 29개, CI 워크플로 43개
4. **폭넓은 커버리지** — 5개 언어 SDK + MAF/Semantic Kernel/AutoGen/LangGraph/CrewAI/OpenAI Agents SDK/LlamaIndex/Dify 등 어댑터
5. **논쟁적 메시지 + 학술 근거** — "프롬프트 방어는 통제가 아니다"를 arXiv·OWASP·NIST 인용으로 뒷받침
6. **표준 인증** — OWASP Agentic Top 10 / NIST AI RMF / EU AI Act / SOC 2 / OpenSSF Scorecard / AARM / ATF
7. **커뮤니티 및 거버넌스 문서** — Discord, GOVERNANCE.md, CHARTER.md, ANTITRUST.md, 기여자 사다리
8. **한계를 공개** — `docs/LIMITATIONS.md`에서 못 하는 것을 명시해 신뢰 확보

---

## 9. 로컬 에이전트 구축에 도움이 되는가

### 바로 활용 가능한 것
- Claude Code 플러그인으로 로컬 파일/자격증명 보호
- 내 에이전트 툴에 `govern()` 적용 → 호출 검사·로깅
- 감사 로그로 에이전트 행동 추적
- 킬 스위치로 폭주 정지
- 정책 YAML로 툴 화이트리스트/블랙리스트를 코드 밖에서 관리

### 학습 자료로서의 가치
- 특권 링(Ring) 기반 샌드박스 설계 (`agent-runtime`)
- Merkle 기반 변조 감지 감사 로그 (`docs/specs/AUDIT-COMPLIANCE-1.0.md`)
- 에이전트 신원 체계 (DID / SPIFFE / mTLS)
- 서킷 브레이커, SLO, 에러 버짓 (`agent-sre`)
- 프롬프트 인젝션 12벡터 평가기 (`prompt_defense.py`)
- MCP 보안 게이트웨이 (툴 포이즈닝, 드리프트, 타이포스쿼팅 탐지)
- `examples/` 41종 실전 레시피

### 공식 문서가 밝힌 한계 (`docs/LIMITATIONS.md`)
- **행동 거버넌스이지 사고(reasoning) 거버넌스가 아님** — 에이전트가 무엇을 하는지는 통제하나 무엇을 생각/말하는지는 통제하지 않음
- 허용된 툴에 전달되는 **내용이 환각인지**는 탐지하지 않음
- 간접 프롬프트 인젝션으로 인한 추론 오염은 탐지하지 않음
- 개별적으로 허용된 행동들이 조합되어 만드는 악성 워크플로는 상관 분석하지 않음
  (예: `read_database` + `send_slack_message` 모두 허용이면 고객 명단 유출 가능)
- 세션 경계를 넘는 지속 메모리/툴을 통한 공격 체인도 동일한 공백에 해당
- 정책 엔진과 에이전트가 **같은 프로세스 경계**를 공유 → 프로덕션에서는 에이전트별 컨테이너 격리 권장

---

## 10. React / PHP로 만들 수 있는가

### React — 적합
이 레포 자체가 이미 React를 대규모로 사용합니다. `agent-os-vscode/src/webviews/`에 `.tsx` **49개**:
`Sidebar.tsx`, `TopologyDetail.tsx`, `ForceGraph.tsx`, `SLOGauge.tsx`, `SLOSparkline.tsx`,
`AuditDetail.tsx`, `PolicyDetail.tsx`, `HubDetail.tsx` 등 + Tailwind 설정 포함.

```tsx
import { PolicyEngine } from "@microsoft/agent-governance-sdk";

const engine = new PolicyEngine([
  { action: "web_search", effect: "allow" },
  { action: "shell_exec", effect: "deny" },
]);
engine.evaluate("shell_exec"); // "deny"
```

React로 만들기 좋은 것: 정책 비주얼 에디터, 실시간 감사 로그 뷰어,
에이전트 토폴로지 그래프, SLO 대시보드, 정책 시뮬레이터.
추천 스택: **Next.js + TypeScript + Tailwind + shadcn/ui + Recharts**

### PHP — 가능하나 역할 분리 필요
공식 PHP SDK는 **없습니다**(Python / TS / .NET / Rust / Go만 제공). 세 가지 전략:

1. **사이드카 방식(권장)** — PHP(Laravel)는 UI·인증·결제·리포트, 정책 판정은 Python/Node AGT 서비스에 REST로 위임
2. **CLI 호출 방식** — `shell_exec('agt lint-policy policies/ --json')` 후 JSON 파싱 (간단하나 성능 제약)
3. **PHP 포팅** — `docs/specs/` 명세 기반으로 PHP 정책 엔진 직접 구현 → "PHP 생태계 최초 AGT 호환 SDK" 포지셔닝

### 권장 조합
```
프론트엔드: React (Next.js)
백엔드:     Node.js 또는 PHP(Laravel)
거버넌스:   Python AGT 또는 TypeScript SDK
저장소:     PostgreSQL + Redis
```

---

## 11. 수익화 아이디어

### 법적 전제
- **MIT 라이선스** → 상업적 이용 · 수정 · 재배포 · 판매 모두 허용
- 조건: **저작권 고지 + MIT 라이선스 사본 포함**
- **금지**: "Microsoft" 상표/로고 사용, Microsoft 후원·공식 인증 암시 (`TRADEMARKS.md`)
- 허용: "Agent Governance Toolkit 기반" 같은 사실 서술
- 주의: 현재 **Public Preview** — GA 이전 Breaking Change 가능 (`BREAKING_CHANGES.md` 참고)

### 티어 1 — 즉시 시작 가능 (자본 최소)

**1) AI 에이전트 보안 감사 컨설팅**

| 등급 | 내용 | 가격(제안) |
|---|---|---|
| Basic | `agt verify` + OWASP 리포트 | 100~200만원 |
| Standard | + 정책 설계 + 레드팀 스캔 | 300~500만원 |
| Premium | + 구축 대행 + 3개월 모니터링 | 1,000만원~ |

`agt verify --evidence`로 증빙 산출물이 자동 생성되는 점이 핵심 무기.

**2) 한국어 교육 콘텐츠 (가장 빠른 현금화)**

| 상품 | 가격 | 채널 |
|---|---|---|
| 온라인 강의 | 5~15만원 | 인프런 / 유데미 |
| 전자책 | 2~5만원 | 크몽 / 브런치 |
| 유튜브 시리즈 | 광고·협찬 | 유튜브 |
| 기업 출강 | 시간당 30~100만원 | B2B |
| 유료 뉴스레터 | 월 1만원 | 스티비 / 메일리 |

공식 문서에 한국어 번역은 있으나 **실전 강의 콘텐츠는 공백** → 키워드 선점 기회.

**3) 산업별 정책 템플릿 팩**

| 팩 | 대상 | 가격(제안) |
|---|---|---|
| 금융/핀테크 | 전자금융감독규정 대응 | 30만원 |
| 의료 | 개인정보보호법 / HIPAA | 30만원 |
| 이커머스 | 결제·주문 보호 | 20만원 |
| 스타트업 스타터 | 기본 안전장치 | 5만원 |

여기에 React 기반 비주얼 정책 에디터("Policy Studio")를 SaaS로 결합 가능.

### 티어 2 — 제품화 (3~6개월)

**4) AGT Cloud — 관리형 거버넌스 SaaS (최우선 추천)**

AGT는 강력하지만 러닝커브가 큼(명세 10개, 문서 17만 줄). "설치·설정 없이 5분 시작"이 핵심 가치 제안.

| 플랜 | 월 가격 | 내용 |
|---|---|---|
| Free | 0원 | 에이전트 1개, 로그 7일 |
| Starter | 4.9만원 | 에이전트 5개, 로그 30일, 대시보드 |
| Pro | 29만원 | 에이전트 50개, 로그 1년, 알림, SSO |
| Enterprise | 협의 | 온프레미스, SLA, 전담 지원 |

스택: Next.js + Tailwind / Node.js 또는 FastAPI / AGT SDK / PostgreSQL + Redis + S3 / 결제 연동

MVP 로드맵: 1개월차 정책 에디터 + 로그 뷰어 → 2개월차 SDK 연동 + 실시간 대시보드 → 3개월차 결제 + 알림 + 배포

**5) 업종 특화 버티컬 제품**

| 제품(예) | 타깃 | 킬러 기능 |
|---|---|---|
| FinGuard AI | 핀테크 | 금융 규제 정책팩 + 감사증적 자동 생성 |
| MediGuard AI | 헬스케어 | 환자정보 접근통제 + 개인정보보호법 리포트 |
| ShopGuard AI | 이커머스 | 주문·환불 에이전트 보호, 결제 조작 차단 |
| EduGuard AI | 교육 | 학생 데이터 보호, 부적절 콘텐츠 차단 |

**6) 개발자 도구 판매**

| 제품 | 형태 | 가격(제안) |
|---|---|---|
| AGT Studio (VS Code Pro 확장) | 확장 | 월 9,900원 |
| AGT for React | npm 라이브러리 | 무료 + Pro |
| AGT CI Action Pro | GitHub Action | 리포당 월 3만원 |
| AGT Policy Linter Pro | CLI | 연 12만원 |

### 티어 3 — 장기 (6개월 이상)

**7) AI 안전 인증 마크 사업** — 자동 검사 → 등급 부여 → 배지 발급, 연간 갱신료 수익. ISMS-P / ISO 27001 모델과 동일 구조.

**8) 사이버보험 연계** — AGT 거버넌스 적용 기업에 보험료 할인, 보험사 채널을 통한 수수료 수익.

**9) MSP / 관제 서비스** — 24시간 에이전트 모니터링, 이상 행동 알림, 긴급 킬스위치 대행, 월간 컴플라이언스 리포트. 에이전트당 월 10~50만원.

### 우선순위

| 순위 | 아이디어 | 자본 | 소요 시간 | 리스크 | 잠재 수익 |
|:--:|---|:--:|:--:|:--:|:--:|
| 1 | 교육 콘텐츠 | 최소 | 1개월 | 낮음 | 중 |
| 2 | 컨설팅/감사 | 최소 | 즉시 | 낮음 | 중상 |
| 3 | AGT Cloud SaaS | 중 | 3~6개월 | 중 | 최상 |
| 4 | 정책 템플릿 | 소 | 1~2개월 | 낮음 | 중 |
| 5 | 버티컬 제품 | 중상 | 6개월 | 중 | 상 |
| 6 | 개발자 도구 | 소 | 2~3개월 | 중 | 중 |
| 7 | 인증 사업 | 상 | 12개월+ | 높음 | 최상 |

### 실행 전략
```
1~2개월  : 교육 콘텐츠로 현금 흐름 + 인지도 확보
2~4개월  : 컨설팅으로 실제 고객 문제 학습
4~10개월 : 학습 내용을 반영한 AGT Cloud SaaS 제작
10개월~  : 버티컬 확장 또는 인증 사업 진출
```

### 차별화 포인트
1. 한국어 + 한국 규제(개인정보보호법, 전자금융감독규정) 대응
2. 쉬운 UI — AGT의 진입장벽이 가장 큰 공백
3. 초기 진입자 이점 — 국내 전문가 희소
4. React 역량으로 가장 부족한 영역(UI/UX)을 보완

---

## 12. 요약

| 질문 | 답변 |
|---|---|
| 뭐하는 거? | AI 에이전트의 행동을 실행 전에 가로채 정책으로 허용/거부하고 감사 기록을 남기는 거버넌스 툴킷 |
| 언제 써? | 에이전트가 툴/DB/외부 API를 자율 호출할 때, 규제·감사 대응이 필요할 때, Claude Code 로컬 안전장치가 필요할 때 |
| 플러그인/스킬/MCP? | MCP 서버 + Hooks + 슬래시 커맨드를 포함한 **플러그인** (Skill 형식은 없음) |
| 토큰 필요? | 기본 기능은 **불필요**. Azure 등 선택 기능만 인증 정보 필요 |
| React 가능? | 가능하며 이미 레포 내 `.tsx` 49개 사용 중 |
| PHP 가능? | 공식 SDK는 없음. 사이드카(REST) 방식 권장 |
| 수익화? | 교육 → 컨설팅 → SaaS 순서 추천, MIT라 상업적 활용 가능(상표 제외) |

---

*이 문서는 Claude Code 세션의 분석 결과를 정리한 것으로, 수치와 인용은 레포지토리 `v5.0.0` 기준 실측값입니다.*
