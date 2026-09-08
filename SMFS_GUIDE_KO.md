# 📚 smfs 완전 정리 노트 (한국어)

> Supermemory 컨테이너를 **로컬 폴더처럼** 다루는 도구, `smfs`를 처음부터 끝까지 정리한 문서입니다.
> 개요 → 설치/사용법 → 수익화 아이디어 → 기술 스택(React/PHP) 순서로 구성했습니다.

---

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| 📦 **이 저장소 (내 포크)** | https://github.com/bmshin94/smfs |
| ⭐ **원본 저장소** | https://github.com/supermemoryai/smfs |
| 🌐 Supermemory 서비스 | https://supermemory.ai |
| ⚡ 설치 스크립트 | https://smfs.ai/install |
| 📘 npm 패키지 (TypeScript) | https://www.npmjs.com/package/@supermemory/bash |
| 🐍 PyPI 패키지 (Python) | https://pypi.org/project/supermemory-bash |
| 📄 라이선스 | MIT (상업적 이용 자유) · vendored `just-bash` 는 Apache-2.0 |

---

## 목차

1. [smfs가 뭔가요?](#1-smfs가-뭔가요)
2. [핵심 기능 3가지](#2-핵심-기능-3가지)
3. [폴더 구조](#3-폴더-구조)
4. [설치 가이드](#4-설치-가이드)
5. [사용법](#5-사용법)
6. [명령어 레퍼런스](#6-명령어-레퍼런스)
7. [문제 해결](#7-문제-해결)
8. [수익화 아이디어](#8-수익화-아이디어)
9. [React / PHP 로 만들기](#9-react--php-로-만들기)
10. [REST API 레퍼런스](#10-rest-api-레퍼런스)

---

## 1. smfs가 뭔가요?

한 문장 요약:

> **AI의 기억 창고를, 내 컴퓨터의 평범한 폴더처럼 보이게 해주는 도구**

### 구글 드라이브 비유

구글 드라이브 데스크톱 앱을 깔면, 파일은 인터넷 어딘가에 있지만 내 컴퓨터에선 그냥 폴더로 보입니다.
smfs가 정확히 그것입니다. 다만 저장되는 곳이 구글이 아니라 **AI 전용 기억 창고(Supermemory)** 라는 점만 다릅니다.

```
[내 컴퓨터의 폴더]   ←→   [AI의 기억 창고]
     똑같이 보임          실제로는 클라우드에 있음
```

### 왜 필요한가?

- **문제**: AI는 대화창을 닫으면 다 잊어버립니다. 어제 얘기한 걸 오늘 또 설명해야 합니다.
- **해결**: AI에게 "메모장 폴더"를 쥐여줍니다. AI가 스스로 파일로 적어두고, 나중에 찾아보고, 세션이 바뀌어도 기억합니다.

### 접근 방식 2가지

| 방식 | 설명 | 사용 환경 |
|---|---|---|
| 🗂️ **디렉터리 마운트** | 실제 로컬 폴더로 마운트 | macOS, Linux, devcontainer, Docker |
| 🧰 **가상 bash 툴** | 파일시스템 자체를 흉내내는 라이브러리 | Cloudflare Workers, 서버리스, 엣지, 브라우저 에이전트 |

---

## 2. 핵심 기능 3가지

### ① `grep` 이 의미 검색으로 바뀜

`smfs init` 을 한 번 실행하면, 마운트 폴더 안에서 `grep` 의 동작이 갈라집니다.

```sh
grep "OAuth 리프레시 토큰"        # 🧠 시맨틱(의미) 검색 — 플래그 없음
grep "디자인 회의록" work/         # 🧠 폴더 범위로 의미 검색
grep -F "정확한문자열" notes.md    # 📝 진짜 grep — 플래그 있음
grep -rn "TODO" .                 # 📝 진짜 grep
```

**규칙은 단 하나:**

> 마운트 폴더 **안** + 플래그 **없음** = 의미 검색 / 그 외 전부 = 원래 grep

폴더 안에 숨겨진 `.smfs` 마커 파일을 위로 거슬러 올라가며 찾아 판단합니다.
마운트 밖에서는 `grep` 이 100% 원래대로 동작합니다.

> 💡 핵심 설계 사상: **새 명령어를 배울 필요도, AI 에이전트에게 새 툴을 가르칠 필요도 없다.**

### ② 백그라운드 자동 동기화 (4개 루프)

`crates/smfs-core/src/sync/mod.rs` 기준:

| 루프 | 역할 | 주기 |
|---|---|---|
| **A. delta pull** | 서버에서 변경된 문서 가져오기 | 30초 |
| **C. deletion scan** | 서버에서 삭제된 파일을 로컬에서도 제거 | 5분 |
| **D. push worker** | 로컬 쓰기를 서버로 업로드 (빠른 연속 수정은 파일당 최대 2요청으로 합침) | 즉시 |
| **E. inflight poller** | 서버 측 처리 완료 여부 폴링 | 상시 |

로컬 캐시는 SQLite 로 유지되어 빠르고, 업로드 큐도 SQLite 에 남아 재마운트 시 이어서 처리됩니다.

### ③ 파일시스템 없는 환경용 "가상 bash"

`bash/` (TypeScript) 와 `bash-py/` (Python) 가 이 역할을 합니다.

```ts
import { createBash } from "@supermemory/bash";

const { bash, toolDescription } = await createBash({
  apiKey: process.env.SUPERMEMORY_API_KEY!,
  containerTag: "user_42",
});

await bash.exec("echo '메모' > /notes/a.md");
await bash.exec("sgrep '인증 토큰'");   // 의미 검색 전용 명령어
```

LLM 에게 `run_bash` 툴 하나만 넘기면, LLM 이 이미 아는 `ls` / `cat` / `grep` / 파이프 / 리다이렉트가 전부 클라우드 메모리 위에서 동작합니다.
`toolDescription` 이라는 잘 작성된 툴 설명문도 함께 제공됩니다.

---

## 3. 폴더 구조

```
smfs/
├── crates/                    🦀 Rust — CLI 본체
│   ├── smfs/src/cmd/          → mount, grep, sync, login, daemon 등 서브커맨드
│   └── smfs-core/src/
│       ├── vfs/               → 가상 파일시스템 (POSIX 시맨틱)
│       ├── mount/             → fuse.rs (Linux) / nfs.rs (macOS)
│       ├── cache/             → SQLite 로컬 캐시
│       ├── sync/              → push / pull 동기화 엔진
│       ├── api/               → Supermemory REST 클라이언트 (5회 재시도 + 백오프)
│       └── agent_hint.rs      → ⭐ 에이전트 지시문 자동 주입 (아래 참고)
├── bash/                      📘 TypeScript 가상 bash 패키지
├── bash-py/                   🐍 Python 버전 (동일 기능)
├── install.sh                 설치 스크립트 (체크섬 검증 포함)
├── Dockerfile, docker/        도커 지원
└── .devcontainer/             VS Code / Cursor 데브컨테이너
```

### 🔍 `agent_hint.rs` 가 흥미로운 이유

`smfs mount` 실행 시 아래 파일들에 **경로 범위가 지정된 규칙 블록을 자동 주입**합니다.

- `~/.claude/CLAUDE.md`
- `~/.codex/AGENTS.md`
- `~/.gemini/GEMINI.md`

내용은 "이 마운트 경로 안에서 검색할 땐 `smfs grep` 을 써라" 이고, `smfs unmount` 시 깔끔하게 제거됩니다.
해당 에이전트의 디렉터리가 **이미 존재할 때만** 건드리며, 없으면 새로 만들지 않습니다.

> 소스 주석에 "Anthropic 문서상 CLAUDE.md 준수 보장은 없으니, 계약이 아니라 유도(steer)로 취급하라"고 솔직하게 적혀 있습니다.

---

## 4. 설치 가이드

### 0단계 · 준비물

| 필요한 것 | 설명 |
|---|---|
| 🖥️ macOS 또는 Linux | ❌ **Windows 미지원** (`install.sh` 에서 명시적으로 거부) → WSL 사용 |
| 🔑 Supermemory API 키 | https://supermemory.ai 에서 발급 |
| 🐚 zsh 셸 | 시맨틱 grep 래퍼가 **zsh 전용** (아래 주의사항 참고) |

### 방법 A · 한 줄 설치 (권장)

```sh
curl -fsSL https://smfs.ai/install | bash
```

내부 동작:
1. OS / CPU 자동 감지 (`darwin-arm64`, `linux-x64` 등)
2. GitHub 릴리스에서 최신 정식 버전 탐색 (draft·prerelease 제외)
3. 다운로드 후 **SHA256 체크섬 검증** — 불일치 시 즉시 중단
4. `~/.local/bin/smfs` 에 설치

> Rosetta 로 실행 중인 Mac 은 자동으로 arm64 빌드를 선택합니다.

확인:
```sh
smfs --version
```

`command not found` 가 뜨면 PATH 문제입니다:
```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 방법 B · 소스 빌드 (Rust 1.80+)

```sh
git clone https://github.com/bmshin94/smfs.git
cd smfs
cargo build --release
./target/release/smfs install     # ~/.local/bin 으로 복사
```

### 방법 C · Docker

```sh
docker build -t smfs:dev .

docker run --rm -it \
  --device /dev/fuse --cap-add SYS_ADMIN \
  -e SUPERMEMORY_API_KEY="$SUPERMEMORY_API_KEY" \
  smfs:dev mount agent_memory --path /mnt/memory
```

> ⚠️ `--device /dev/fuse --cap-add SYS_ADMIN` 이 없으면 마운트에 실패합니다.

---

## 5. 사용법

### 5-1. 로그인

```sh
smfs login                        # 대화형으로 키 입력
smfs login --key sm_xxxxxxxxxxxx  # 한 줄로
```

- 저장 **전에** 서버에 키 유효성을 먼저 확인합니다. 틀리면 `Invalid API key.` 로 거부합니다.
- 통과 시 `credentials.json` 에 저장하며, 파일 권한을 **0600**(소유자만 읽기)으로 설정합니다.

| OS | 저장 위치 |
|---|---|
| macOS | `~/Library/Application Support/ai.supermemory.supermemoryfs/credentials.json` |
| Linux | `~/.config/supermemoryfs/credentials.json` |

확인:
```sh
smfs whoami          # 계정 / 조직 / 플랜
smfs whoami --json   # JSON 출력 (키는 마스킹됨)
```

### 5-2. 마운트

```sh
smfs mount agent_memory                       # ./agent_memory/ 생성
smfs mount agent_memory --path ~/Documents/내기억   # 위치 지정
```

```sh
ls agent_memory/
cat agent_memory/profile.md                    # AI가 정리한 요약 (읽기 전용)
echo "오늘 회의 내용..." > agent_memory/notes/meeting.md   # 쓰면 자동 업로드
```

#### mount 옵션 전체

| 옵션 | 기본값 | 설명 |
|---|---|---|
| `--path <DIR>` | `./<태그>/` | 마운트 경로 |
| `--backend fuse\|nfs` | Linux=fuse, macOS=nfs | 마운트 백엔드 |
| `--foreground` | off | 백그라운드 detach 없이 포그라운드 실행 (디버깅) |
| `--clean` | off | 로컬 캐시 삭제 후 서버에서 새로 받기 |
| `--ephemeral` | off | 인메모리 캐시, 언마운트 시 아무것도 남지 않음 |
| `--sync-interval <초>` | 30 | 델타 pull 주기 |
| `--deletion-scan-interval <초>` | 300 | 삭제 스캔 주기 |
| `--no-sync` | off | pull 만 중단 (로컬 쓰기는 계속 push 됨) |
| `--drain-timeout <초>` | 30 | 언마운트 시 push 큐 배출 최대 대기 |
| `--memory-paths "<csv>"` | 서버 설정 유지 | 아래 참고 |
| `--key <KEY>` | 저장된 키 | API 키 직접 지정 |
| `--api-url <URL>` | 프로덕션 | API 주소 재정의 |

#### `--memory-paths` — 반드시 이해할 것

폴더에 넣은 파일이 **전부 "AI 기억"이 되지는 않습니다.**

```
📁 마운트 전체       = 내구성 있는 저장소 (파일은 안전하게 보관)
   └─ 🧠 memory paths = 여기 있는 것만 의미 분석되어 검색 대상이 됨
```

```sh
# notes 폴더 전체 + journal.md 파일만 기억으로 처리
smfs mount agent_memory --memory-paths "/notes/,/journal.md,/work/"

# 기억 처리 완전 끄기 → 순수 저장소로만 사용
smfs mount agent_memory --memory-paths ""

# 플래그 자체를 생략하면 → 서버의 기존 설정 그대로 유지
smfs mount agent_memory
```

- 끝에 `/` 있음 → 해당 폴더 안 전부 (재귀)
- 끝에 `/` 없음 → 정확히 그 파일 하나

이 설정은 컨테이너 태그에 저장되므로, 다음 마운트 때도 동일하게 적용됩니다.

### 5-3. 시맨틱 grep 활성화

```sh
smfs init
source ~/.zshrc     # 반드시 실행해야 적용됨
```

#### ⚠️ 셸 호환성 주의 (중요)

`crates/smfs/src/cmd/init.rs` 확인 결과, 래퍼는 **`~/.zshrc` 에만** 설치됩니다.

| 셸 | 동작 |
|---|---|
| **zsh** (macOS 기본) | ✅ 정상 작동 |
| **bash** | ❌ 자동 설치 안 됨 |
| **fish** | ❌ 자동 설치 안 됨 |

bash 사용자는 `~/.zshrc` 에 추가된 `grep()` 함수를 복사해 `~/.bashrc` 에 붙여넣으면 됩니다.
(문법이 POSIX 호환이라 그대로 동작합니다.)

> 💡 smfs 버전을 올린 뒤에는 `smfs init` 을 다시 실행하세요. 구버전 래퍼를 제거하고 새로 설치합니다.

마운트 밖에서 검색하려면:
```sh
smfs grep "인증 관련 메모" --tag agent_memory
```

---

## 6. 명령어 레퍼런스

```sh
smfs login                      # API 키 저장 (검증 후 0600 권한으로 저장)
smfs whoami [--json]            # 현재 계정 / 조직 / 플랜
smfs mount <tag> [옵션]         # 컨테이너 태그 마운트
smfs unmount <tag> [--force]    # 언마운트 (push 큐 배출 후 종료)
smfs list                       # 실행 중인 모든 마운트
smfs status [tag] [--json]      # 데몬 상태 + 큐 깊이
smfs logs [tag] [-f] [-n 200]   # 데몬 로그 (tail -f 처럼)
smfs sync [tag]                 # 즉시 동기화 강제 실행
smfs grep "질의" [경로]          # 시맨틱 검색
smfs init                       # grep 셸 래퍼 (재)설치
smfs install                    # 바이너리를 ~/.local/bin 으로 자체 설치
smfs logout [--project]         # 저장된 자격증명 삭제
```

> 💡 마운트 폴더 **안**에 있으면 태그를 생략할 수 있습니다. `.smfs` 마커를 위로 탐색해 자동 인식합니다.
> ```sh
> cd agent_memory/ && smfs status
> ```

### 일상 워크플로 요약

```sh
# ① 설치
curl -fsSL https://smfs.ai/install | bash

# ② PATH (필요 시)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc

# ③ 로그인
smfs login

# ④ 시맨틱 grep 활성화 (최초 1회)
smfs init && source ~/.zshrc

# ⑤ 마운트
smfs mount my_memory

# ⑥ 사용
cd my_memory
echo "메모 내용" > notes/hello.md
grep "메모 관련 질문"

# ⑦ 종료
cd .. && smfs unmount my_memory
```

---

## 7. 문제 해결

| 증상 | 원인 / 해결 |
|---|---|
| `command not found: smfs` | PATH 누락 → `export PATH="$HOME/.local/bin:$PATH"` |
| `grep` 이 의미 검색으로 안 바뀜 | ① `source ~/.zshrc` 했는지 ② bash 사용 중인지 ③ 마운트 폴더 **안**인지 확인 |
| 마운트 실패 (Docker/Linux) | FUSE 권한 → `--device /dev/fuse --cap-add SYS_ADMIN` |
| 방금 쓴 파일이 검색되지 않음 | **정상 동작**. 서버 인덱싱에 보통 5~30초 소요 (메모리 추출은 더 걸릴 수 있음) |
| 상태가 꼬임 | `smfs logs -f` 확인 → `smfs unmount --force` → `smfs mount <태그> --clean` |
| 상세 로그가 필요함 | `RUST_LOG=smfs=debug,smfs_core=debug smfs mount ... --foreground` |

로그 파일: `~/.config/supermemoryfs/logs/<태그>.log` (macOS 는 Application Support 하위)

### 지원하지 않는 것

- `chmod`, `utimes`, 심볼릭 링크 (`ln -s`, `readlink`) → Supermemory 에 권한/심링크 모델이 없어 `ENOSYS`
- `/dev/null` 리다이렉트 → 디렉터리 마커로만 존재. 대신 `2>/tmp/discard.log` 사용
- 순수 바이너리 업로드 → 서버에서 텍스트 추출 처리되므로 현재 버전에서는 미지원

### 꼭 기억할 5가지

1. 🚫 Windows 미지원 — WSL / macOS / Linux 에서 사용
2. 🐚 zsh 가 아니면 시맨틱 grep 래퍼가 자동 설치되지 않음
3. 🧠 `memory-paths` 안에 있는 파일만 의미 검색 대상
4. ⏳ 방금 쓴 내용은 5~30초 후에 검색됨 (최종적 일관성)
5. 🔌 종료 시 반드시 `smfs unmount` — 업로드가 남아 있을 수 있음

---

## 8. 수익화 아이디어

### 먼저 냉정한 현실 3가지

1. **이건 제품이 아니라 부품입니다.** smfs 는 Supermemory API 가 있어야 동작합니다. 무엇을 만들든 **남의 인프라 위에 짓는 것**이며, 원가가 그쪽 가격표에 묶입니다.
2. **플랫폼 리스크가 있습니다.** 잘 되면 Supermemory 가 직접 그 기능을 만들 수 있습니다. → **그들이 하지 않을 영역**(업종 특화, 국내 규제 대응, 온프레미스 구축)을 골라야 안전합니다.
3. **개인용 AI 메모장 앱은 레드오션입니다.** 이미 수백 개가 있고, 획득 비용 대비 이탈률이 높습니다. B2C 는 피하는 편이 좋습니다.

### Tier 1 — 자본 0원, 즉시 시작 가능

#### ① AI 기억 구축 대행 (컨설팅) ⭐ 최우선 추천

| 항목 | 내용 |
|---|---|
| 타겟 | 직원 20~200명 중소기업, 로펌, 병원, 학원, 대행사 |
| 모델 | 구축비 300~1,000만원 + 월 유지보수 30~80만원 |
| 난이도 | ⭐⭐ (기술보다 영업이 관건) |

**근거**: 회사에는 수년치 문서가 쌓여 있지만 아무도 찾지 못합니다. 그걸 마운트해 AI 가 답하게 만들면 "우리 회사 전용 AI"가 됩니다.

**차별화 포인트**: smfs 는 그냥 **폴더**입니다. 담당자에게 "이 폴더에 넣으세요" 한 마디면 끝입니다. 별도 업로드 UI 나 교육이 필요 없다는 점이 강력한 세일즈 포인트입니다.

#### ② 콘텐츠 → 신뢰 → 수주

유튜브 / 블로그 / 뉴스레터로 "AI 에게 기억을 주는 법" 시리즈 제작.
직접 수익은 작지만, **①번 고객이 알아서 찾아오게 만드는 것**이 진짜 목적입니다.
한국어로 이 주제를 다루는 사람이 거의 없어 선점 기회가 있습니다.

### Tier 2 — 제품화 (3~6개월)

#### ③ 개발팀 "장애 기억" 서비스

| 항목 | 내용 |
|---|---|
| 타겟 | 개발팀 5~50명 규모 스타트업 |
| 모델 | 시트당 월 1~2만원 |
| 난이도 | ⭐⭐⭐ |

**시나리오**: 새벽 장애 발생 → Cursor / Claude Code 가 팀 메모리를 `sgrep` → "8개월 전 동일 에러. 원인: 커넥션 풀. 해결: ..."

**근거**: 팀 지식은 슬랙에 묻히고 슬랙 검색은 형편없습니다. 반면 AI 코딩 도구는 이미 bash 를 쓰므로 **툴 하나만 꽂으면 되어 통합 비용이 거의 0** 입니다.

#### ④ Obsidian / Notion 브릿지 플러그인

| 항목 | 내용 |
|---|---|
| 타겟 | 이미 노트앱을 적극적으로 쓰는 파워유저 |
| 모델 | 유료 플러그인 $5~9/월 또는 평생 $49 |
| 난이도 | ⭐⭐ |

**근거**: Obsidian 은 원래 **로컬 마크다운 폴더**이고, smfs 마운트도 **폴더**입니다. 궁합이 매우 좋아 개발이 빠릅니다. 또한 노트앱 유저는 이미 유료 결제 습관이 있어, B2C 중 유일하게 시도해볼 만합니다.

#### ⑤ 버티컬 특화 — 상담 기록 메모리

| 항목 | 내용 |
|---|---|
| 타겟 | 세무사 / 변호사 / 심리상담사 / 코치 / 학원 |
| 모델 | 월 10~30만원 |
| 난이도 | ⭐⭐⭐⭐ (개인정보 규제 대응 필요) |

**근거**: "3년 전 그 고객이 뭐라고 했더라?"가 매일의 고통입니다. 단가가 높고 한번 도입하면 잘 바꾸지 않습니다.

### Tier 3 — 큰 승부 (6개월+)

#### ⑥ 셀프호스팅(온프레미스) 버전

smfs 의 **파일시스템 인터페이스는 그대로 두고**, 백엔드를 Supermemory 대신 자체 구축(PostgreSQL + pgvector 등)으로 교체합니다.

| 항목 | 내용 |
|---|---|
| 타겟 | 금융 / 공공 / 의료 — 데이터를 외부로 보낼 수 없는 조직 |
| 모델 | 라이선스 연 2,000만원~ 또는 구축비 |
| 난이도 | ⭐⭐⭐⭐⭐ |

**근거**: 이 시장에는 SaaS 회사인 Supermemory 가 들어오지 않으므로 플랫폼 리스크가 없습니다. 국내 대기업·공공은 "클라우드 불가"가 기본값이라 수요가 확실합니다.
`crates/smfs-core/src/api/` 만 교체하면 되도록 코드가 잘 분리되어 있어 기술적으로 실현 가능합니다.

### 추천 루트

```
1개월차   ② 콘텐츠 (직접 써보고 기록)      → 신뢰 확보
   ↓
2~4개월   ① 구축 대행 1~2건 수주           → 현금 확보 + 진짜 니즈 파악
   ↓
5개월~    반복 작업을 제품화               → ③ / ④ / ⑤
   ↓
1년~      ⑥ 온프레미스로 큰 계약
```

> **"서비스로 돈을 벌면서, 그 과정에서 발견한 반복 작업을 제품으로 만든다."**
> 자본 없이 시작하는 가장 안전한 경로입니다.

### 시작 전 반드시 확인할 것

| 체크 | 이유 |
|---|---|
| 💵 Supermemory 가격표 확인 | 문서 1만개면 월 얼마인가? 이것이 **원가**입니다. 계산하지 않으면 팔수록 손해입니다 |
| 📜 재판매 가능 여부 확인 | 남의 API 를 고객에게 재판매하는 것이 약관상 가능한지 문의 (보통 파트너/리셀러 프로그램 존재) |
| 🔒 개인정보 처리방침 | 타사 문서를 다루면 **수탁자**가 됩니다. 계약서가 필수입니다 |
| 🧪 본인이 먼저 3주 사용 | 직접 써보지 않은 것을 팔 수는 없습니다 |

---

## 9. React / PHP 로 만들기

### 결론

| 스택 | 가능? | 평가 |
|---|---|---|
| **React 단독** | ❌ 불가 | API 키가 브라우저에 노출됨 |
| **React + Next.js/Node** | ✅✅ | 공식 패키지를 그대로 사용. 가장 빠름 |
| **PHP** | ✅ | 전용 SDK 는 없지만 REST 직접 호출로 충분 |

### 🚨 React 단독이 안 되는 이유

```jsx
// ❌ 절대 금지
const res = await fetch("https://api.supermemory.ai/v4/search", {
  headers: { Authorization: `Bearer ${API_KEY}` }   // 브라우저에 그대로 노출
});
```

React 는 브라우저에서 실행됩니다. 개발자도구를 열면 키가 그대로 보입니다.
`.env` 에 넣어도 소용없습니다 — `REACT_APP_` / `NEXT_PUBLIC_` 접두사가 붙은 변수는 **빌드 시 번들에 박힙니다.**

> **철칙: API 키는 반드시 서버에만 둡니다.**

### 올바른 구조

```
👤 사용자
   ↓
🎨 React (화면)              ← 키 없음. 화면만 담당
   ↓  fetch("/api/search")   ← 내 서버로만 요청
🖥️ 내 서버 (PHP or Node)     ← 🔑 여기에만 키 보관
   ↓  Authorization: Bearer sm_xxx
☁️ Supermemory API
```

### PHP 구현

의존성 없는 순수 cURL 클래스입니다.

```php
<?php
// Supermemory.php

class Supermemory
{
    private string $key;
    private string $base = 'https://api.supermemory.ai';

    public function __construct(string $key) { $this->key = $key; }

    private function req(string $method, string $path, ?array $body = null): array
    {
        $ch = curl_init($this->base . $path);
        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST  => $method,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 30,
            CURLOPT_HTTPHEADER     => [
                'Authorization: Bearer ' . $this->key,
                'Content-Type: application/json',
            ],
        ]);
        if ($body !== null) {
            curl_setopt($ch, CURLOPT_POSTFIELDS,
                json_encode($body, JSON_UNESCAPED_UNICODE));
        }

        $raw  = curl_exec($ch);
        $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($code >= 400) {
            throw new RuntimeException("Supermemory {$code}: {$raw}");
        }
        return json_decode($raw, true) ?? [];
    }

    /** 의미 검색 */
    public function search(string $q, string $tag, ?string $filepath = null): array
    {
        $body = [
            'q'            => $q,
            'containerTag' => $tag,
            'searchMode'   => 'hybrid',
            'include'      => ['documents' => true],
        ];
        if ($filepath !== null) $body['filepath'] = $filepath;

        return $this->req('POST', '/v4/search', $body);
    }

    /** 문서 저장 */
    public function save(string $tag, string $filepath, string $content): array
    {
        return $this->req('POST', '/v3/documents', [
            'content'      => $content,
            'filepath'     => $filepath,        // 예: "/notes/meeting.md"
            'containerTag' => $tag,
        ]);
    }

    /** 문서 목록 */
    public function listDocs(string $tag, int $page = 1, int $limit = 50): array
    {
        return $this->req('POST', '/v3/documents/list', [
            'containerTags'  => [$tag],
            'limit'          => $limit,
            'page'           => $page,
            'includeContent' => false,
            'sort'           => 'updatedAt',
            'order'          => 'desc',
        ]);
    }

    /** 키 검증 */
    public function whoami(): array { return $this->req('GET', '/v3/session'); }
}
```

React 가 호출할 엔드포인트:

```php
<?php
// api/search.php
header('Content-Type: application/json; charset=utf-8');
require __DIR__ . '/../Supermemory.php';

$sm = new Supermemory(getenv('SUPERMEMORY_API_KEY'));   // 서버 환경변수에서만

$q = trim($_GET['q'] ?? '');
if ($q === '') {
    http_response_code(400);
    echo json_encode(['error' => '검색어를 입력해주세요']);
    exit;
}

try {
    // ⚠️ containerTag 는 반드시 서버 세션 기준으로 생성
    $tag = 'user_' . $_SESSION['user_id'];

    $result = $sm->search($q, $tag);
    echo json_encode($result['results'] ?? [], JSON_UNESCAPED_UNICODE);
} catch (Throwable $e) {
    http_response_code(500);
    echo json_encode(['error' => '검색 실패']);   // 내부 에러는 노출하지 않음
    error_log($e->getMessage());
}
```

> 🚨 `containerTag` 를 프론트엔드에서 받으면 **다른 사용자의 데이터를 조회할 수 있습니다.** 반드시 서버 세션에서 생성하세요.

### React 구현

```jsx
import { useState } from "react";

export default function MemorySearch() {
  const [q, setQ] = useState("");
  const [hits, setHits] = useState([]);
  const [loading, setLoading] = useState(false);

  async function handleSearch(e) {
    e.preventDefault();
    setLoading(true);
    try {
      // 키 없음. 내 서버로만 요청
      const res = await fetch(`/api/search.php?q=${encodeURIComponent(q)}`);
      setHits(await res.json());
    } finally {
      setLoading(false);
    }
  }

  return (
    <form onSubmit={handleSearch}>
      <input
        value={q}
        onChange={(e) => setQ(e.target.value)}
        placeholder="예: 작년 계약 조건이 뭐였지?"
      />
      <button disabled={loading}>{loading ? "찾는 중…" : "검색"}</button>

      <ul>
        {hits.map((h) => (
          <li key={h.id}>
            <b>{h.filepath}</b>
            <p>{h.memory ?? h.chunk}</p>
            <small>유사도 {(h.similarity * 100).toFixed(0)}%</small>
          </li>
        ))}
      </ul>
    </form>
  );
}
```

### Next.js 구현 (가장 간단)

```sh
npm install @supermemory/bash just-bash supermemory
```

> `just-bash` 와 `supermemory` 는 **peerDependency** 이므로 함께 설치해야 합니다.

```ts
// app/api/search/route.ts   ← 서버에서만 실행되므로 키가 안전
import { createBash } from "@supermemory/bash";

export async function POST(req: Request) {
  const { q } = await req.json();

  const { bash } = await createBash({
    apiKey: process.env.SUPERMEMORY_API_KEY!,   // NEXT_PUBLIC_ 붙이면 안 됨
    containerTag: `user_${await getUserId()}`,
  });

  const r = await bash.exec(`sgrep '${q.replace(/'/g, "'\\''")}'`);
  return Response.json({ output: r.stdout });
}
```

#### `createBash` 옵션

```ts
createBash({
  apiKey: string,
  containerTag: string,        // 사용자 / 프로젝트당 하나
  baseURL?: string,
  eagerLoad?: boolean,         // 기본 true — 시작 시 경로 인덱스 워밍업
  eagerContent?: boolean,      // 기본 true — 문서 1만개 이상이면 false 권장
  cacheTtlMs?: number | null,  // 기본 150_000 (2.5분). null=무한(단일 writer), 0=캐시 없음
  cwd?: string,                // 기본 "/home/user"
  env?: Record<string, string>,
});
```

### 스택 선택 가이드

| 상황 | 추천 | 이유 |
|---|---|---|
| 새로 시작 + 빠르게 만들고 싶음 | **Next.js** | 공식 패키지 그대로, `sgrep` 내장, 단일 배포 |
| 이미 PHP 서버 / 워드프레스 보유 | **PHP + React** | 위 코드를 붙이면 바로 동작 |
| AI 기능을 많이 붙일 예정 | **Python(FastAPI) + React** | LangChain 등 생태계가 파이썬 중심 |
| AI 에이전트가 주인공 | **Next.js** | `toolDescription` 을 그대로 사용 |

수익화 아이디어별 추천:

| 아이디어 | 추천 스택 |
|---|---|
| ① 구축 대행 | 고객사 환경에 맞춤 (PHP 비중 높음) |
| ③ 개발팀 지식 두뇌 | Next.js |
| ④ Obsidian 플러그인 | TypeScript (선택지 없음) |
| ⑤ 전문직 상담 메모리 | PHP 또는 Next.js |

---

## 10. REST API 레퍼런스

> ⚠️ 아래 정보는 **이 저장소의 소스코드(v0.0.5 / `@supermemory/bash` 0.0.32)를 직접 읽어 정리한 것**입니다.
> 실제 개발 전에는 Supermemory 공식 API 문서와 대조하세요. 특히 `/v4/search` 응답 필드명은 버전에 따라 달라질 수 있습니다.

**Base URL**: `https://api.supermemory.ai`
**공통 헤더**: `Authorization: Bearer <API_KEY>` · `Content-Type: application/json`

| 기능 | 메서드 | 경로 |
|---|---|---|
| 🔍 의미 검색 | POST | `/v4/search` |
| 📋 문서 목록 | POST | `/v3/documents/list` |
| 📄 문서 단건 조회 | GET | `/v3/documents/{id}` |
| ⏳ 처리중 문서 조회 | GET | `/v3/documents/processing?containerTag=<tag>` |
| 💾 문서 생성 | POST | `/v3/documents` |
| ✏️ 문서 수정 | PATCH | `/v3/documents/{id}` |
| 🗑️ 문서 일괄 삭제 | DELETE | `/v3/documents/bulk` |
| 📎 파일 업로드 | POST | `/v3/documents/file` (multipart) |
| 👤 세션 / 키 검증 | GET | `/v3/session` |
| ⚙️ 컨테이너 설정 | PATCH | `/v3/container-tags/{tag}` |

> 🤓 검색만 `v4`, 나머지는 `v3` 입니다. 검색 엔진이 별도로 개편된 것으로 보입니다.

### 요청 바디 형태

**문서 생성** (`POST /v3/documents`)
```json
{
  "content": "본문 텍스트",
  "filepath": "/notes/meeting.md",
  "containerTag": "user_42",
  "metadata": {}
}
```

**문서 수정** (`PATCH /v3/documents/{id}`) — 모든 필드 선택적
```json
{ "filepath": "...", "content": "...", "metadata": {} }
```

**문서 목록** (`POST /v3/documents/list`)
```json
{
  "containerTags": ["user_42"],
  "filepath": "/notes/",
  "limit": 50,
  "page": 1,
  "includeContent": false,
  "sort": "updatedAt",
  "order": "desc"
}
```

**의미 검색** (`POST /v4/search`)
```json
{
  "q": "인증 토큰 갱신 방식",
  "containerTag": "user_42",
  "searchMode": "hybrid",
  "include": { "documents": true },
  "filepath": "/work/"
}
```

응답의 `results[]` 각 항목은 대략 다음 형태입니다:

```json
{
  "id": "doc_xxx",
  "memory": "추출된 기억 텍스트",
  "chunk": "원문 청크",
  "similarity": 0.87,
  "filepath": "/work/auth.md"
}
```

### 클라이언트 동작 참고

- **재시도**: 네트워크 오류와 5xx 는 최대 **5회** 재시도, 초기 백오프 **100ms** 에서 지수 증가 (최대 10초). 4xx 는 재시도하지 않고 즉시 오류.
- **문서 상태**: `done` / `failed` / `processing` 세 가지.
- **최종적 일관성**: 쓰기는 즉시 반환되고 로컬 캐시로 자기 읽기는 보장되지만, 다른 세션과 `sgrep` 은 서버 인덱싱 완료 후(보통 5~30초) 반영됩니다.

---

## 📄 라이선스

- 본 저장소: **MIT** (Copyright (c) 2026 Supermemory) — 상업적 이용, 수정, 재배포 자유. 저작권 고지만 유지하면 됩니다.
- `bash-py` 에 vendoring 된 [`just-bash-py`](https://github.com/dbreunig/just-bash-py) 0.1.16: **Apache License 2.0**

---

*이 문서는 https://github.com/bmshin94/smfs 저장소의 소스코드를 직접 분석하여 작성되었습니다.*
