# 포트폴리오 사이트 구축·배포 전체 가이드 (Windows 기준)

> Magic Portfolio(Next.js) → GitHub Actions → GCP(GCE + HTTPS LB + Cloud DNS) 배포
> 작성일: 2026-08-09 / 정리: 2026-08-10
> 대상: 인턴 조카(2026-09-04 퇴사) 취업용 포트폴리오 사이트
> 기준 소스: https://github.com/once-ui-system/magic-portfolio
> **전제: 계정도 개발 환경도 아무것도 없는 상태에서 시작**

---

## 이 문서 사용법

- **「진행 순서」의 1번부터 13번까지 차례대로** 따라 하면 된다. 각 항목은 앞 항목이 끝나 있다는 것을 전제로 하므로 건너뛰지 말 것.
- 명령어는 전부 **Windows PowerShell** 기준이다. 코드 블록의 명령만 복사해서 붙여넣으면 된다.
- 각 항목 끝의 **[확인] 완료 조건**을 통과해야 다음으로 넘어간다. 통과 못 한 채로 진행하면 뒤에서 원인 찾기가 훨씬 어려워진다.
- ⚠️ 표시는 **실제로 자주 사고 나는 지점**이다. 여기만 조심해도 대부분의 시간 낭비를 막을 수 있다.
- 모든 파일과 폴더는 **`C:\recruit`** 안에서 관리한다.
- 막히면 **부록 B(Windows 특화 문제 모음)** 를 먼저 찾아볼 것.

**전체 소요 시간**: 실작업은 5~6시간이지만, DNS 전파와 SSL 인증서 발급 **대기 시간**이 끼어 있다. 여유 있게 **이틀**을 잡으면 된다. 시간 배분은 **부록 A**의 일정표를 참고.

---

## 최종 목표 아키텍처

```
[방문자]
   │  https://포트폴리오도메인.com
   ▼
[Cloud DNS]  ← 고대디에서 네임서버를 GCP로 위임
   │  A 레코드 → 고정 글로벌 IP
   ▼
[Global External HTTPS Load Balancer]
   ├─ Google 관리형 SSL 인증서 (무료·자동 갱신)
   ├─ HTTP(80) → HTTPS(443) 리다이렉트
   └─ Backend Service (+ Cloud CDN)
        │  health check :3000/
        ▼
[Unmanaged Instance Group]
   └─ GCE VM 1대 (asia-northeast3-a, Ubuntu 22.04 + Docker)
        └─ docker run  portfolio:latest  (Next.js standalone, :3000)
             ▲
             │ gcloud compute ssh --tunnel-through-iap → docker pull & run
        [GitHub Actions]
             ▲
             │ docker build & push
        [Artifact Registry]  asia-northeast3-docker.pkg.dev
```

**핵심 흐름**: 코드 수정 → `main`에 푸시 → GitHub Actions가 이미지 빌드·푸시 → VM에서 컨테이너 교체 → 사이트 반영 **(2~4분)**

> 💡 **이 그림은 포트폴리오의 프로젝트 소개에 그대로 써도 좋다** (12-4 참고). 신입 기준으로 인프라 구성 경험을 보여주는 가장 효과적인 자료다.

---

## 시작 전 알아둘 3가지

### 1. 라이선스 (중요)

Magic Portfolio는 **CC BY-NC 4.0** 라이선스다.

- **비상업적 사용만 허용** → 개인 취업용 포트폴리오는 해당됨. 문제 없음.
- **출처 표기(Attribution) 필수** → 푸터의 **Once UI 크레딧을 지우면 안 된다.**
- 상업적 이용(외주 제작, 회사 홍보 등)을 하려면 Once UI Pro 라이선스를 사서 확장해야 한다.

구체적인 표기 방법은 **2-8**에서 다룬다.

### 2. 무료 크레딧의 함정 ⚠️

GCP 무료 체험 크레딧 $300은 **90일 만료**다. **금액이 남아 있어도 90일이 지나면 소멸된다.**

| 항목 | 사양 | 월 예상 비용(서울 리전) |
|---|---|---|
| GCE VM | e2-small (2GB) | 약 $17 |
| 부팅 디스크 | pd-balanced 20GB | 약 $2.4 |
| Global HTTPS LB | 전달 규칙 1개 + 소량 트래픽 | 약 $19 |
| Cloud DNS | 존 1개 | 약 $0.3 |
| Artifact Registry | 이미지 수 GB | 약 $0.5 |
| **합계** | | **월 약 $39** |

- 월 $39이므로 크레딧으로는 **약 3개월** 무료 운영이 가능하다. 즉 **금액보다 90일 만료가 먼저 도달**한다.
- 2026년 11월경 만료 시점에 반드시 결정이 필요하다. 선택지는 **13-5** 참고.
- 프로젝트를 만들면 **예산 알림부터 걸어둘 것** (5-5). 이게 사실상 유일한 안전장치다.

### 3. 계정·비용 명의

- GCP 프로젝트, 결제 계정, GitHub 리포, 도메인 **전부 조카 본인 명의**로 만들 것. 나중에 인수인계할 게 없어진다.
- 카드는 무료 체험 등록에 필요하지만 크레딧 소진 전까진 실제 청구가 없다. ⚠️ **"유료 계정으로 업그레이드"는 누르지 말 것.**
- **이 작업은 인턴 종료(2026-09-04) 전에 끝내두는 게 좋다.** 이후엔 원격으로 봐주기가 번거로워진다.
- ⚠️ **써클 인턴 경력 내용은 재직 중에 정리해둘 것.** 담당 업무, 사용 기술, 성과 수치는 나가고 나면 세부가 흐려진다. (12-4 참고)

---

# 진행 순서

> 1번부터 13번까지 순서대로 진행한다. 각 항목은 앞 항목이 끝나 있다는 것을 전제로 한다.

**전체 흐름 한눈에 보기**

| # | 항목 | 대략 소요 |
|---|---|---|
| 1 | 개발 환경 구성 (`C:\recruit`) | 90분 (재부팅 2회) |
| 2 | GitHub 계정 + 리포지토리 이관 | 40분 |
| 3 | 고대디 가입 + 도메인 구매 | 20분 |
| 4 | GCP Console 가입 | 20분 |
| 5 | GCP 프로젝트 + gcloud 설정 | 30분 |
| 6 | Cloud DNS + 네임서버 위임 | 20분 + **대기 10분~1시간** |
| 7 | GCE 인스턴스 생성 | 30분 |
| 8 | HTTPS LB + SSL 인증서 | 20분 + **대기 15분~24시간** |
| 9 | 앱 컨테이너화 (Dockerfile) | 40분 |
| 10 | Artifact Registry + 서비스 계정 | 30분 |
| 11 | GitHub Actions 첫 배포 | 40분 |
| 12 | 콘텐츠 커스터마이징 | **2~6시간** |
| 13 | 최종 검증 및 운영 | 30분 |

⚠️ **6번과 8번의 대기 시간이 유일한 병목**이다. 기다리는 동안 9~12번을 진행하도록 순서를 짰다. 자세한 일정 배분은 **부록 A** 참고.

## 1. 개발 환경 구성 (`C:\recruit`)

조카 PC(Windows)에 개발 환경을 구축한다. 이후 모든 작업은 `C:\recruit` 폴더 아래에서 진행한다.

> **설치 목록**: Windows Terminal → Git → Node.js → Python → Java(JDK) → VS Code → Claude Code → WSL2 → Docker Desktop → Google Cloud SDK
>
> Python과 Java는 이 포트폴리오 사이트(Next.js) 구동에 필요하진 않지만, **개발자로서의 기본 환경**이라 함께 설치한다. 시간이 부족하면 Python·Java는 뒤로 미뤄도 나머지 작업에 지장은 없다.

⚠️ **이 장 전체를 관통하는 원칙**: **프로그램을 설치할 때마다 터미널을 닫았다가 새로 열 것.** PATH 환경변수가 갱신되지 않아 "명령을 찾을 수 없습니다"가 뜨는 것이 이 단계 사고의 90%다.

### 1-1. 작업 폴더 만들기

`Win + X` → **터미널(관리자)** 실행 후:

```powershell
New-Item -ItemType Directory -Force -Path "C:\recruit"
Set-Location "C:\recruit"
```

⚠️ **`C:\` 바로 아래에 폴더를 만들려면 관리자 권한이 필요해.** 일반 권한 PowerShell에서 실행하면 "액세스가 거부되었습니다"가 뜬다. 관리자 터미널에서 한 번만 만들어두면 이후 작업은 일반 권한으로 해도 된다.

만든 뒤 폴더 구조는 이렇게 갈 예정이야:

```
C:\recruit\
  ├─ portfolio\        ← 포트폴리오 소스코드 (1-12에서 생성)
  └─ set-vars.ps1      ← GCP 작업용 변수 파일 (GCP 인프라 단계에서 생성)
```

### 1-2. PowerShell 실행 정책 풀기

이후 `.ps1` 스크립트와 Claude Code 설치 스크립트를 실행해야 하는데, Windows 기본 설정에서는 차단된다. 한 번만 풀어주면 된다.

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

`Y` 입력 후 엔터.

### 1-3. winget 확인

Windows 10 1809 이상이면 기본 설치돼 있다.

```powershell
winget --version
```

버전이 안 나오면 Microsoft Store에서 **"앱 설치 관리자"**를 설치하면 된다.

> ⚠️ **winget 패키지 ID는 시간이 지나면서 바뀔 수 있어.** 아래 명령이 "패키지를 찾을 수 없습니다"로 실패하면 `winget search <이름>` 으로 정확한 ID를 확인한 뒤 그 ID로 설치할 것. (예: `winget search python`, `winget search openjdk`)

### 1-4. 기본 도구 설치

**[관리자] PowerShell**에서 순서대로:

```powershell
# 터미널 (선택이지만 권장 — 복사·붙여넣기와 탭 관리가 훨씬 편함)
winget install --id Microsoft.WindowsTerminal -e

# Git
winget install --id Git.Git -e

# Node.js LTS
winget install --id OpenJS.NodeJS.LTS -e

# VS Code
winget install --id Microsoft.VisualStudioCode -e
```

**터미널을 닫고 새로 열어서** 확인:

```powershell
git --version     # git version 2.4x.x
node -v           # v22.x.x 이상
npm -v            # 10.x.x 이상
code --version
```

> Magic Portfolio의 최소 요구는 Node 18.17+ 이지만, Node 20은 2026년 4월에 지원이 끝났어. **LTS(22.x 이상)** 로 간다.

### 1-5. Git 최초 설정 ⚠️ 개행 설정 주의

```powershell
git config --global user.name "홍길동"
git config --global user.email "취업용Gmail주소@gmail.com"
git config --global init.defaultBranch main
git config --global core.autocrlf input
```

⚠️ **`core.autocrlf input`이 핵심이야.** Windows Git은 기본적으로 파일 개행을 CRLF로 바꿔 저장하는데, 리눅스 서버에서 실행될 셸 스크립트(`.sh`)나 Dockerfile에 CRLF가 들어가면 **`$'\r': command not found` 에러로 배포가 통째로 실패해.**

설정 확인:

```powershell
git config --global --list
```

> `user.email`은 2번에서 만들 취업용 Gmail 주소로 넣는다. 아직 계정이 없으면 2번을 먼저 하고 돌아와도 된다.

### 1-6. Python 개발 환경

```powershell
winget install --id Python.Python.3.13 -e
```

> ⚠️ 위 ID가 실패하면 `winget search Python.Python.3` 로 설치 가능한 최신 3.x 버전을 확인해서 그 ID를 쓸 것.

터미널 새로 열고 확인:

```powershell
python --version
pip --version
```

⚠️ **`python`을 쳤을 때 Microsoft Store가 열리면** Windows 기본 앱 별칭이 실제 Python을 가리고 있는 것이다.
**설정 → 앱 → 고급 앱 설정 → 앱 실행 별칭**에서 `python.exe`, `python3.exe` 항목을 **끄면** 해결된다.

프로젝트별 가상환경 사용법(파이썬 작업할 때 습관화할 것):

```powershell
python -m venv .venv           # 가상환경 생성
.\.venv\Scripts\Activate.ps1   # 활성화 (프롬프트 앞에 (.venv) 표시됨)
pip install <패키지>
deactivate                     # 비활성화
```

VS Code 확장: **Python** (Microsoft 제공) 설치.

### 1-7. Java 개발 환경

```powershell
winget install --id Microsoft.OpenJDK.21 -e
```

> ⚠️ 위 ID가 실패하거나 더 최신 LTS를 원하면 `winget search openjdk` 또는 `winget search Temurin` 으로 확인할 것.
> 대안: `winget install --id EclipseAdoptium.Temurin.21.JDK -e`

터미널 새로 열고 확인:

```powershell
java -version
javac -version     # JDK가 맞게 깔렸는지 확인 (JRE만 깔리면 javac이 없음)
```

`JAVA_HOME` 확인 (일부 빌드 도구가 이 값을 참조한다):

```powershell
$env:JAVA_HOME
```

비어 있으면 **[관리자]** PowerShell에서 설정 (경로는 실제 설치 경로로 교체):

```powershell
[Environment]::SetEnvironmentVariable("JAVA_HOME", "C:\Program Files\Microsoft\jdk-21.0.x", "Machine")
```

VS Code 확장: **Extension Pack for Java** (Microsoft 제공) 설치.

### 1-8. WSL2 설치 (Docker Desktop 전제 조건)

**[관리자]** PowerShell에서:

```powershell
wsl --install
```

⚠️ 설치 후 **반드시 재부팅.** 재부팅하면 Ubuntu 초기 설정 창이 뜨는데, 사용자명·비밀번호를 만들어두면 된다 (이 계정은 이번 작업에서 직접 쓰진 않는다).

재부팅 후 확인:

```powershell
wsl --status
# 기본 버전: 2  로 나오면 성공
```

> 안 될 경우: BIOS에서 가상화(Virtualization / Intel VT-x / AMD-V / SVM)가 꺼져 있을 수 있다. 제조사별 BIOS 진입키가 다르다(보통 F2/F10/Del). "가상화" 항목을 Enabled로 바꾸면 된다.

### 1-9. Docker Desktop 설치

```powershell
winget install --id Docker.DockerDesktop -e
```

설치 후:

1. **재부팅**
2. 시작 메뉴에서 **Docker Desktop** 실행
3. 약관 동의 → 로그인은 건너뛰어도 됨(Skip)
4. 우측 하단 트레이의 고래 아이콘이 **멈추면** 준비 완료 (움직이면 아직 시작 중)

확인:

```powershell
docker --version
docker run --rm hello-world
# "Hello from Docker!" 가 나오면 성공
```

⚠️ **Docker Desktop이 실행 중이어야 `docker` 명령이 동작해.** PC를 껐다 켠 뒤에는 Docker Desktop을 먼저 실행하고 작업할 것.

### 1-10. Google Cloud SDK(gcloud CLI) 설치

```powershell
winget install --id Google.CloudSDK -e
```

터미널 새로 열고 확인:

```powershell
gcloud --version
```

⚠️ winget 설치가 실패하거나 명령을 못 찾으면 설치 파일로 하면 된다:
https://dl.google.com/dl/cloudsdk/channels/rapid/GoogleCloudSDKInstaller.exe
→ 실행 → "Bundled Python" 체크 유지 → 설치 → 터미널 재시작

> **로그인(`gcloud auth login`)과 프로젝트 설정은 4번(GCP 가입) 이후에 한다.** 여기서는 설치와 버전 확인까지만.

### 1-11. VS Code + Claude Code 세팅

**(1) Claude Code 설치**

PowerShell에서 (1-2의 실행 정책을 먼저 풀어둬야 한다):

```powershell
irm https://claude.ai/install.ps1 | iex
```

또는 npm으로 (Node.js가 이미 깔려 있으므로 이쪽도 가능):

```powershell
npm install -g @anthropic-ai/claude-code
```

터미널 새로 열고 확인:

```powershell
claude --version
```

**(2) VS Code 확장 설치**

- VS Code 좌측 **확장(Extensions)** 아이콘 → `Claude Code` 검색 → 설치
- 또는 VS Code 통합 터미널(`Ctrl + \``)에서 `claude` 를 실행하면 확장 설치를 안내해준다

**(3) 로그인**

터미널에서 `claude` 실행 후 `/login` 입력 → 브라우저가 열리면 계정 인증.

⚠️ **Claude Code는 유료 구독(Claude Pro/Max) 또는 Anthropic Console의 API 크레딧이 있어야 사용할 수 있어.** 무료 계정으로는 동작하지 않는다. 취업 준비 기간 동안만 쓸 거라면 구독 비용을 미리 확인하고 결정할 것. **없어도 이 문서의 나머지 작업은 전부 진행 가능하다.**

**(4) 그 외 권장 VS Code 확장**

| 확장 | 용도 |
|---|---|
| **MDX** | 포트폴리오 콘텐츠(`.mdx`) 편집 |
| **Docker** | Dockerfile 문법 강조·컨테이너 관리 |
| **ESLint** | 코드 검사 |
| **Python** | 파이썬 (1-6) |
| **Extension Pack for Java** | 자바 (1-7) |

### 1-12. Magic Portfolio 로컬 실행 확인 ⚠️

환경이 제대로 갖춰졌는지 실제로 돌려서 확인한다. **여기가 이 장의 진짜 완료 조건이야.**

```powershell
Set-Location "C:\recruit"
git clone --depth 1 https://github.com/once-ui-system/magic-portfolio.git portfolio
Set-Location portfolio
Remove-Item -Recurse -Force .git
npm install
npm run dev
```

브라우저에서 http://localhost:3000 접속 → 포트폴리오 화면이 나오면 성공. 확인했으면 터미널에서 `Ctrl + C`로 종료.

> `Remove-Item -Recurse -Force .git` 은 원본 템플릿의 깃 히스토리를 지우는 것이다. **2번에서** 조카 계정의 새 리포지토리로 다시 초기화한다.
> `npm install`에서 에러가 나면 `node -v` 로 Node 버전부터 확인할 것.

### [확인] 1번 완료 조건

아래를 한 번에 실행해서 전부 버전이 출력되면 통과:

```powershell
Write-Host "=== 설치 확인 ===" -ForegroundColor Cyan
git --version
node -v
npm -v
python --version
java -version
docker --version
gcloud --version
claude --version
wsl --status
```

- [ ] `C:\recruit` 폴더 생성됨
- [ ] git, node, npm, python, java, docker, gcloud 전부 버전 출력됨
- [ ] `git config --global core.autocrlf` 값이 `input`
- [ ] `docker run --rm hello-world` 성공
- [ ] VS Code에 Claude Code 확장 설치 + 로그인 완료 (구독이 있는 경우)
- [ ] `npm run dev` 로 localhost:3000에 포트폴리오 화면이 정상 표시됨

## 2. GitHub 계정 만들고 포트폴리오 리포지토리 이관

취업용 Gmail로 GitHub 계정을 만든 뒤, 원본 템플릿 리포지토리를 **조카 계정의 새 리포지토리로 이관**한다.

### 2-1. 취업용 Gmail 계정 만들기

취업 관련 계정을 **하나의 Gmail로 통일**한다. 앞으로 GitHub·GCP·고대디를 전부 이 계정으로 만든다.

**이미 개인 Gmail이 있어도 새로 하나 만드는 것을 권장한다.**

1. https://accounts.google.com/signup 접속
2. 이름 → 아이디 → 비밀번호 입력
3. 복구용 전화번호·이메일 등록 (**계정이 잠겼을 때 유일한 복구 수단이니 꼭 등록**)
4. 약관 동의 → 완료

⚠️ **아이디는 단정하게.** 이력서와 지원 메일에 그대로 노출된다. `xxgamer1234@gmail.com` 같은 주소는 새로 만드는 게 낫다. 본인 이름 기반을 권장.

**이렇게 통일하는 이유**

- 계정이 흩어져 있으면 인증 메일·알림이 여러 곳으로 분산돼서 놓치기 쉽다. 특히 **도메인 소유권 확인 메일(3-4)** 이나 **GCP 예산 알림(5-5)** 은 놓치면 사고로 이어진다.
- 기존 개인 계정에는 학습용 리포지토리나 실습 커밋이 섞여 있는 경우가 많다. 채용 담당자가 보는 프로필은 정돈된 상태가 유리하다.
- 이력서에 적을 이메일 주소와 GitHub 계정 주소가 하나로 묶여서 일관성 있게 보인다.

### 2-2. Google 계정 2단계 인증(2SV) ⚠️ 필수

⚠️ **2026년 10월 20일부터 GCP 콘솔 로그인에 2단계 인증이 의무화된다.** 나중에 갑자기 막히면 곤란하니 지금 켜둔다.

1. https://myaccount.google.com/security 접속
2. "Google에 로그인" → **2단계 인증** 클릭
3. "시작하기" → 비밀번호 재입력
4. 휴대전화 번호 입력 → SMS로 코드 수신 → 입력 → **사용 설정**
5. 다시 2단계 인증 화면으로 들어가 아래로 스크롤 → **백업 코드** → "백업 코드 받기"
6. 10개 코드가 나오면 **다운로드하거나 인쇄해서 안전한 곳에 보관**

> ⚠️ **백업 코드는 진짜로 저장해둘 것.** 휴대폰을 잃어버리거나 기기를 바꾸면 이게 유일한 진입 수단이다. GCP 프로젝트에 접근하지 못하면 사이트 관리 자체가 불가능해진다.

### 2-3. GitHub 계정 만들기

1. https://github.com/signup 접속
2. **2-1의 취업용 Gmail 주소** → 비밀번호 → **사용자 이름(username)** 입력
3. 이메일로 온 인증 코드 입력
4. 설문은 건너뛰어도 됨 → **Free** 플랜 선택

⚠️ **username은 신중하게 정할 것.** 이력서에 `github.com/사용자이름` 으로 적히고, 나중에 바꾸면 **기존 링크가 전부 깨진다.** 본인 이름 기반(`minsu-kim`, `kimminsu-dev` 등)을 권장한다. 숫자 나열이나 게임 닉네임은 피할 것.

### 2-4. GitHub 2단계 인증(2FA) ⚠️ 필수

GitHub는 2023년부터 코드를 올리는 계정에 2FA를 의무화했다. 안 켜면 어느 시점에 강제로 막힌다.

1. https://github.com/settings/security 접속
2. **Two-factor authentication** → "Enable two-factor authentication"
3. 방식 선택:
   - **인증 앱(권장)**: 휴대폰에 Google Authenticator 또는 Microsoft Authenticator 설치 → QR 스캔 → 6자리 코드 입력
   - **SMS**: 인증 앱을 못 쓰면 차선책
4. **복구 코드(recovery codes)가 나오면 반드시 다운로드해서 보관**

> 2-2의 백업 코드와 마찬가지로, 복구 코드가 없으면 계정을 잃었을 때 되찾을 방법이 없다.

### 2-5. 원본 리포지토리 이관 — 포크가 아니라 "새 리포"로

이관 대상 원본: **https://github.com/once-ui-system/magic-portfolio**

⚠️ **GitHub의 Fork 버튼을 쓰지 말 것.** 새 리포지토리로 시작하는 걸 권장한다.

| | 포크(Fork) | 새 리포 (권장) |
|---|---|---|
| 프로필 표시 | `forked from once-ui-system/magic-portfolio` 꼬리표가 붙음 | 조카 본인의 리포로 보임 |
| 커밋 히스토리 | 원본 커밋과 섞임 | 조카 커밋만 남음 |
| 이력서 관점 | 남의 프로젝트를 복사한 것으로 보이기 쉬움 | 본인이 만든 프로젝트로 읽힘 |
| 업스트림 업데이트 | `git pull`로 가져올 수 있음 | 못 가져옴 (아래 참고) |

> **트레이드오프 하나**: 히스토리를 버리기 때문에 나중에 원본 템플릿이 업데이트돼도 `git pull`로 받아올 수 없다(unrelated histories). 필요해지면 `upstream` 리모트를 따로 추가해서 특정 커밋만 cherry-pick 하면 된다. 어차피 콘텐츠를 전면 교체할 거라 실질적인 제약은 아니다.

**소스는 이미 준비돼 있다.** 1-12에서 `C:\recruit\portfolio`로 clone하고 `.git`을 지운 상태다. 아직 안 했으면 1-12를 먼저 하고 올 것.

### 2-6. GitHub에 빈 리포지토리 만들기

1. https://github.com/new 접속
2. **Repository name**: `portfolio`
3. **Public** 선택
   → 이력서에 리포 주소를 같이 적을 수 있어서 권장. 코드를 보여주는 것 자체가 포트폴리오다.
4. ⚠️ **"Add a README file", "Add .gitignore", "Choose a license" 세 개는 전부 체크 해제.**
   로컬에 이미 같은 파일들이 있어서, 체크하면 첫 푸시에서 충돌이 난다.
5. **Create repository** 클릭
6. 다음 화면에 나오는 리포 주소(`https://github.com/조카아이디/portfolio.git`)를 메모

### 2-7. 로컬 소스를 새 리포지토리로 푸시

```powershell
Set-Location "C:\recruit\portfolio"

# 혹시 .git이 남아 있으면 제거 (1-12에서 이미 지웠다면 아무 일도 안 일어남)
if (Test-Path .git) { Remove-Item -Recurse -Force .git }

git init -b main
git add .
git commit -m "chore: initialize portfolio from Magic Portfolio template"
git remote add origin https://github.com/조카아이디/portfolio.git
git push -u origin main
```

⚠️ **첫 푸시 때 Git Credential Manager 창이 뜨면서 브라우저 로그인**을 요구한다. 2-3에서 만든 GitHub 계정으로 로그인하면 이후로는 자동 인증된다.

확인:

```powershell
git remote -v          # origin이 조카 리포 주소로 나오는지
git log --oneline -1   # 커밋 1개가 보이는지
git status -sb         # "## main...origin/main" — 동기화 상태
```

브라우저에서 `https://github.com/조카아이디/portfolio` 에 들어가 파일이 올라와 있는지도 눈으로 확인할 것.

> ⚠️ **`git push`가 인증 오류로 막히면** GitHub는 2021년부터 비밀번호 인증을 막았다. Credential Manager 창의 브라우저 로그인을 쓰거나, 그래도 안 되면 **Settings → Developer settings → Personal access tokens**에서 토큰을 발급받아 비밀번호 자리에 넣으면 된다.

### 2-8. 라이선스 의무 — 출처 표기 ⚠️

Magic Portfolio는 **CC BY-NC 4.0** 라이선스다.

- **비상업적 사용만 허용** → 개인 취업용 포트폴리오는 해당됨. 문제 없음.
- **출처 표기(Attribution) 필수** → 푸터의 **Once UI 크레딧을 지우면 안 된다.**
- 상업적 이용(외주 제작, 회사 홍보 등)을 하려면 Once UI Pro 라이선스를 별도로 구매해야 한다.

`README.md` 최상단에 아래 한 줄을 추가한다:

```markdown
Based on [Magic Portfolio](https://github.com/once-ui-system/magic-portfolio) by Once UI (CC BY-NC 4.0).
```

`LICENSE` 파일은 **삭제하지 말고 그대로 둘 것.**

커밋:

```powershell
git add README.md
git commit -m "docs: credit Magic Portfolio template"
git push
```

> **이력서에는 정직하게 쓰는 게 유리하다.** "Once UI Magic Portfolio 템플릿 기반으로 커스터마이징 및 GCP 인프라 구축"이라고 적으면, 면접에서 오히려 인프라 역량으로 어필된다. 템플릿을 감추려다 들키는 게 훨씬 손해다.

### 2-9. GitHub 프로필 정리

https://github.com/settings/profile 에서:

- **프로필 사진 등록** (기본 아이콘이면 성의 없어 보인다)
- **이름(본명), 한 줄 소개, 위치** 입력
- **Website 칸**은 3번에서 도메인을 사고 배포까지 끝난 뒤에 등록한다

### [확인] 2번 완료 조건

- [ ] 취업용 Gmail 계정 생성됨
- [ ] **Google 계정 2단계 인증 켜짐 + 백업 코드 저장함**
- [ ] 그 Gmail로 GitHub 계정 생성됨 (username 신중히 결정)
- [ ] **GitHub 2FA 켜짐 + 복구 코드 저장함**
- [ ] `github.com/조카아이디/portfolio` 리포지토리가 Public으로 생성됨
- [ ] 로컬 `C:\recruit\portfolio`가 그 리포로 푸시됨 (`git status -sb`가 `## main...origin/main`)
- [ ] 브라우저에서 리포에 파일이 보이고, **"forked from" 꼬리표가 없음**
- [ ] README 최상단에 출처 표기 추가 + 푸시됨
- [ ] `LICENSE` 파일이 그대로 남아 있음
- [ ] GitHub 프로필에 사진·이름·소개 입력됨

## 3. 취업용 Gmail로 고대디 가입 후 도메인 구매

2번에서 만든 취업용 Gmail 계정으로 고대디(GoDaddy)에 가입하고, 포트폴리오에서 쓸 도메인을 구매한다.

### 3-1. 결제 수단 확인 ⚠️ 먼저 할 것

도메인 구매(고대디)와 GCP 가입(4번) 모두 **해외 결제**다.

- **해외 결제가 가능한** 신용카드 또는 체크카드가 필요하다. 체크카드도 되지만 잔액이 있어야 한다.
- ⚠️ **카드사 앱에서 "해외 결제 차단"이 걸려 있지 않은지 미리 확인할 것.** 기본으로 차단된 카드가 많고, 여기서 막히면 진행이 통째로 멈춘다. 차단돼 있으면 카드사 앱·고객센터에서 해제한다.
- GCP는 카드 등록 시 **약 $1 승인 테스트**가 잡혔다가 취소된다. 실제 청구가 아니다.

### 3-2. 고대디 가입

1. https://www.godaddy.com 접속 → 우측 상단 로그인 → **계정 만들기**
2. **취업용 Gmail 주소**와 비밀번호 입력 (GCP·GitHub와 동일한 계정으로 통일)
3. 수신함에서 이메일 인증 완료

### 3-3. 도메인명 정하기 (구매 전에 먼저 결정)

취업용이니까 **본인 이름 기반**이 제일 무난해.

- 추천 형태: `이름성.com`, `이름-dev.com`, `이름.dev`
- 예시: `kimminsu.com`, `minsu-kim.dev`, `minsudev.com`
- 피할 것: 하이픈 2개 이상, 숫자 혼용, 지나치게 긴 이름
  → 판단 기준은 **"전화로 불러줄 수 있는가"**. 면접에서 말로 알려줄 일이 실제로 생겨.

| TLD | 연 비용 | 비고 |
|---|---|---|
| `.com` | 2만원 내외 | 가장 무난, 신뢰감 |
| `.dev` | 2만원 내외 | 개발자 이미지. **HTTPS 강제(HSTS preload)** 라 이 구조와 잘 맞음 |
| `.io`, `.me` | 4~6만원 | 갱신비가 비쌈. 비추 |

⚠️ **1순위가 이미 팔렸을 때를 대비해 후보를 2~3개 미리 정해두면** 현장에서 고민하는 시간을 줄일 수 있어.

### 3-4. 구매 절차

1. 고대디 로그인 상태에서 검색창에 원하는 도메인 입력
2. 사용 가능하면 장바구니에 담기
3. ⚠️ **부가 옵션은 전부 해제할 것.** 여기서 안 빼면 불필요한 비용이 몇 배로 붙어:

   | 옵션 | 처리 | 이유 |
   |---|---|---|
   | 웹 호스팅 | **해제** | GCP에 올릴 거라 불필요 |
   | 이메일(Microsoft 365 등) | **해제** | 불필요 |
   | SSL 인증서 | **해제** | GCP에서 무료로 자동 발급받음 (8-3) |
   | 개인정보 보호(WHOIS) | 무료 기본 제공이면 유지, 유료면 판단 | |

4. 결제 기간은 **1년**으로 (여러 해 결제는 나중에 판단)
5. 결제 완료
6. ⚠️ **고대디에서 오는 "도메인 소유권 확인" 메일의 링크를 반드시 클릭할 것.**
   미인증 상태로 15일이 지나면 도메인이 **정지**돼. 결제만 하고 넘어가는 실수가 잦은 지점이야.
7. **자동 갱신은 켜두는 걸 권장.** 갱신을 놓치면 도메인이 풀려서 이력서에 적은 주소가 죽어.

### 3-5. 구매 정보 메모

```
도메인명: ______________________
구매일:   ______________________
만료일:   ______________________
자동갱신: 켬 / 끔        ← 켜두는 걸 권장
```

### [확인] 3번 완료 조건

- [ ] 고대디 "내 제품"에 도메인이 보임
- [ ] 소유권 확인 메일 인증 완료 (도메인에 경고 표시 없음)
- [ ] 도메인명을 위 메모란에 기록함 (이후 `set-vars.ps1`의 `$DOMAIN` 값으로 씀 — 5-4)

> 참고: 이 도메인은 **6-6에서 네임서버를 GCP로 위임**하게 된다. 고대디에서는 도메인만 사고, DNS 관리는 Cloud DNS에서 하는 구조다.

## 4. 취업용 Gmail로 GCP Console 가입

2번에서 만든 취업용 Gmail 계정으로 Google Cloud Console에 가입하고 무료 체험을 등록한다.

### 4-1. 가입 전 확인

- [ ] **Google 계정 2단계 인증이 켜져 있을 것** (2-2에서 완료)
      ⚠️ **2026년 10월 20일부터 GCP 콘솔 로그인에 2단계 인증이 의무화**돼. 지금 켜두지 않으면 그 시점에 갑자기 로그인이 막혀. 백업 코드도 함께 저장해둘 것.
- [ ] **해외 결제가 가능한 신용/체크카드** 준비 (3번에서 확인한 그 카드)
      카드사 앱에서 **해외 결제 차단이 걸려 있지 않은지** 미리 확인. 여기서 막히는 경우가 은근히 많아.

### 4-2. 무료 체험 등록

1. 취업용 Gmail 계정으로 https://console.cloud.google.com 접속
2. **무료로 시작하기** 클릭
3. 1단계: 국가 **대한민국** 선택 → 약관 동의
4. 2단계: 계정 유형 **개인** → 이름·주소·카드 정보 입력
5. 완료되면 콘솔 상단에 **$300 크레딧과 남은 일수**가 표시됨

### 4-3. 결제 관련 주의사항 ⚠️

- 카드 등록 시 **약 $1 승인 테스트**가 잡혔다가 취소돼. 실제 청구가 아니니 놀라지 않아도 돼.
- ⚠️ **"유료 계정으로 업그레이드(Upgrade)" 버튼은 절대 누르지 말 것.** 누르는 순간 크레딧 소진과 무관하게 실제 과금이 시작돼.
- ⚠️ **크레딧 $300은 90일 만료야.** 금액이 남아 있어도 90일이 지나면 소멸돼. 이 구조의 월 비용이 약 $39라서, **금액보다 90일 만료가 먼저 도달**해.
  - 오늘 가입하면 만료 시점은 **2026년 11월 초**. 그때의 선택지는 13-5 참고.

### 4-4. 크레딧 만료일 메모

```
GCP 가입일:     ______________________
크레딧 만료일:  ______________________   ← 캘린더에 알림 걸어둘 것
```

만료일은 콘솔 상단 또는 **결제 → 개요**에서 확인할 수 있어.

### [확인] 4번 완료 조건

- [ ] 취업용 Gmail로 GCP 콘솔 로그인됨
- [ ] Google 계정 2단계 인증 켜짐 + 백업 코드 저장함
- [ ] 콘솔 상단에 $300 크레딧과 남은 일수가 표시됨
- [ ] 크레딧 만료일을 메모하고 캘린더 알림 등록함
- [ ] "유료 계정으로 업그레이드"를 누르지 않은 상태


---

## 5. GCP 프로젝트 생성 및 gcloud 초기 설정

여기부터가 실제 인프라 구성이다. 이 단계에서는 **아직 아무 리소스도 만들지 않는다.** 이후 모든 `gcloud` 명령이 올바른 프로젝트를 향하도록 준비만 한다.

### 5-1. 프로젝트 생성

콘솔 상단의 프로젝트 선택 드롭다운 → **새 프로젝트**

```
프로젝트 이름: portfolio
프로젝트 ID:   portfolio-XXXXXX   ← 전역에서 유일해야 함. 자동 생성값 그대로 써도 됨
위치:          조직 없음
```

⚠️ **프로젝트 ID는 만든 뒤 절대 바꿀 수 없다.** 화면에 나온 ID를 정확히 메모할 것. 이름(`portfolio`)과 ID(`portfolio-XXXXXX`)는 다른 값이고, 명령어에 쓰는 건 **ID**다.

```
프로젝트 ID: ______________________
```

생성 후 **콘솔 상단에서 이 프로젝트가 선택되어 있는지** 확인한다. 무료 체험 가입 시 자동 생성된 "My First Project"가 선택된 채로 작업하는 실수가 잦다.

**결제 계정 연결 확인**: 콘솔 → **결제** → 이 프로젝트에 결제 계정이 연결되어 있어야 한다. 4번에서 무료 체험을 등록했다면 자동 연결되지만, 새로 만든 프로젝트는 연결이 빠져 있을 수 있다. 연결 안 되어 있으면 **결제 계정 연결**을 눌러 붙인다.

⚠️ **결제 계정이 안 붙어 있으면** 이후 `gcloud services enable`부터 실패한다.

### 5-2. gcloud 로그인 및 프로젝트 지정

```powershell
gcloud auth login
```

브라우저가 열리면 **취업용 Gmail 계정**으로 로그인 → 권한 허용.

```powershell
gcloud config set project portfolio-XXXXXX      # 본인 프로젝트 ID로 교체
gcloud config list
```

`account`와 `project`가 의도한 값으로 나오면 성공.

⚠️ **PC에 다른 구글 계정이 로그인되어 있던 경우** 엉뚱한 계정이 잡힐 수 있다. 확인·교체:

```powershell
gcloud auth list                               # * 표시가 현재 활성 계정
gcloud config set account 취업용Gmail주소@gmail.com
```

### 5-3. 필요한 API 활성화

```powershell
gcloud services enable `
  compute.googleapis.com `
  artifactregistry.googleapis.com `
  dns.googleapis.com `
  iap.googleapis.com `
  iamcredentials.googleapis.com `
  cloudresourcemanager.googleapis.com
```

> PowerShell에서 백틱(`` ` ``)은 줄바꿈 이어쓰기 기호다. 한 줄로 붙여 써도 된다.

1~2분 걸린다. 완료 확인:

```powershell
gcloud services list --enabled --filter="name:(compute OR artifactregistry OR dns OR iap)"
```

| API | 어디서 쓰나 |
|---|---|
| `compute` | 고정 IP, VM, 방화벽, 로드밸런서 |
| `dns` | **6번의 Cloud DNS 영역** |
| `artifactregistry` | 도커 이미지 저장소 |
| `iap` | VM에 안전하게 SSH 접속 |
| `iamcredentials` | GitHub Actions 배포용 서비스 계정 |
| `cloudresourcemanager` | 프로젝트 단위 권한 부여 |

### 5-4. 작업 변수 파일 만들기 ⚠️ 이후 계속 씀

PowerShell 변수는 창을 닫으면 사라진다. 매번 다시 입력하지 않도록 파일로 만들어 둔다.

VS Code로 **`C:\recruit\set-vars.ps1`** 파일을 만들고 아래 내용을 넣는다:

```powershell
$PROJECT_ID = "portfolio-XXXXXX"         # 5-1에서 메모한 프로젝트 ID로 교체
$REGION     = "asia-northeast3"          # 서울
$ZONE       = "asia-northeast3-a"
$DOMAIN     = "example.com"              # 3번에서 구매한 도메인으로 교체
$REPO       = "portfolio-docker"
$REGISTRY   = "asia-northeast3-docker.pkg.dev"
$SA_NAME    = "github-actions-deployer"
$SA_EMAIL   = "$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"

Write-Host "변수 로드 완료: $PROJECT_ID / $DOMAIN" -ForegroundColor Green
```

**새 터미널을 열 때마다 아래를 먼저 실행**한다:

```powershell
. C:\recruit\set-vars.ps1
```

⚠️ **맨 앞의 점(`.`)과 공백을 빠뜨리지 말 것.** 이건 "닷 소싱(dot sourcing)"이라고 해서, 스크립트의 변수를 **현재 터미널에 남기는** 문법이다. 점 없이 `C:\recruit\set-vars.ps1` 로 실행하면 초록색 메시지는 뜨지만 변수는 전부 사라진다. 이후 명령에서 `$DOMAIN`이 비어 있어 이상하게 실패하는 원인 1순위다.

제대로 로드됐는지 확인:

```powershell
Write-Host "$PROJECT_ID / $DOMAIN"
```

### 5-5. 예산 알림 설정 ⚠️ 건너뛰지 말 것

콘솔 → 좌측 메뉴 **결제** → **예산 및 알림** → **예산 만들기**

```
이름:   portfolio-budget
범위:   프로젝트 = portfolio
금액:   월 $50
알림:   실제 지출 50% / 80% / 100% → 이메일 수신
```

크레딧이 만료된 뒤 방치되어 과금이 계속되는 사고를 막는 **유일한 안전장치**다. 크레딧 만료일(4-4에 메모)과 이 알림 두 가지가 비용 안전망 전부다.

### [확인] 5번 완료 조건

```powershell
gcloud config list
gcloud services list --enabled --filter="name:(compute OR dns)"
. C:\recruit\set-vars.ps1
```

- [ ] 프로젝트가 생성되고 **프로젝트 ID를 메모**함
- [ ] 그 프로젝트에 **결제 계정이 연결**됨
- [ ] `gcloud config list`의 `account`·`project`가 의도한 값
- [ ] API 6개 활성화 완료 (특히 `dns.googleapis.com`)
- [ ] `C:\recruit\set-vars.ps1` 작성 완료, 닷 소싱 시 초록색 메시지 + `$DOMAIN` 값 출력
- [ ] 예산 알림 생성됨

---

## 6. Cloud DNS 영역 생성 및 고대디 네임서버 위임

3번에서 산 도메인이 **GCP를 바라보도록** 만드는 단계다. 이 단계가 이 문서 전체에서 **가장 오래 기다려야 하는 구간**이라, 인프라 작업 중 가장 먼저 한다.

### 6-1. 먼저 구조를 이해하고 시작하자

```
[고대디]  도메인 소유권 등록기관 (Registrar)
   │      "이 도메인의 DNS는 누구에게 물어봐라" 만 알려주는 역할로 축소된다
   │  ── 네임서버 위임 (6-6에서 설정) ──▶
   ▼
[Cloud DNS]  실제 DNS 응답 담당
   └─ A 레코드: 도메인 → 고정 IP
```

- **고대디** = 도메인을 "소유"하는 곳. 갱신·이전은 계속 여기서 한다.
- **Cloud DNS** = 도메인이 어떤 IP를 가리키는지 "대답"하는 곳. 앞으로 모든 레코드는 여기서 관리한다.

⚠️ **위임을 마치고 나면 고대디의 "DNS 레코드" 화면은 아무 효력이 없어진다.** 거기서 A 레코드를 고쳐도 반영되지 않는다. 나중에 "분명히 바꿨는데 왜 안 되지?" 하는 혼동이 여기서 가장 많이 나온다. **레코드 수정은 무조건 Cloud DNS에서.**

> 도메인으로 이메일을 받고 싶어지면(예: `me@내도메인.com`) MX 레코드도 **Cloud DNS에** 추가해야 한다. 이번 구성에는 포함하지 않는다.

### 6-2. 고정 글로벌 IP 예약

A 레코드에 넣을 IP가 먼저 있어야 하므로, 로드밸런서보다 IP를 먼저 만든다.

```powershell
. C:\recruit\set-vars.ps1

gcloud compute addresses create portfolio-ip --global

$LB_IP = gcloud compute addresses describe portfolio-ip --global --format="value(address)"
Write-Host "LB IP: $LB_IP" -ForegroundColor Yellow
```

```
LB IP 메모: ______________________
```

> 이 IP는 지금은 아무 데도 연결돼 있지 않다. 나중에 HTTPS 로드밸런서가 이 주소를 물려받는다. **지금 미리 만드는 이유는 DNS 전파 시계를 최대한 일찍 돌려두기 위해서**다.

⚠️ **`--global`을 빠뜨리지 말 것.** 리전 IP를 만들면 나중에 글로벌 로드밸런서에 붙지 않아 다시 만들어야 한다.

### 6-3. Cloud DNS 영역(zone) 생성

```powershell
gcloud dns managed-zones create portfolio-zone `
  --dns-name="$DOMAIN." `
  --description="Portfolio DNS zone"
```

⚠️ **`$DOMAIN` 뒤의 점(`.`)을 반드시 붙일 것.** DNS에서 맨 뒤의 점은 "루트까지 포함한 완전한 이름(FQDN)"을 뜻한다. `example.com.` 이 되어야 한다.

확인:

```powershell
gcloud dns managed-zones describe portfolio-zone
```

`dnsName`이 `example.com.` 형태로 나오면 성공.

### 6-4. 네임서버 4개 확인 ⚠️ 영역마다 다름

```powershell
(gcloud dns managed-zones describe portfolio-zone --format="value(nameServers)") -split ';'
```

> `value(nameServers)`는 4개를 세미콜론으로 이어 한 줄로 출력한다. 위처럼 `-split ';'` 를 붙이면 한 줄에 하나씩 보기 좋게 나온다.

출력된 4개를 메모한다:

```
1: ns-cloud-__.googledomains.com.
2: ns-cloud-__.googledomains.com.
3: ns-cloud-__.googledomains.com.
4: ns-cloud-__.googledomains.com.
```

⚠️ **이 값은 영역마다 다르다.** 이 문서나 인터넷의 예시를 그대로 복사해 넣으면 도메인이 죽는다. 반드시 위 명령의 실제 출력을 쓸 것.

### 6-5. A 레코드 등록

```powershell
# 새 터미널이라면 변수부터 다시 로드
. C:\recruit\set-vars.ps1
$LB_IP = gcloud compute addresses describe portfolio-ip --global --format="value(address)"

gcloud dns record-sets create "$DOMAIN." `
  --zone=portfolio-zone --type=A --ttl=300 --rrdatas=$LB_IP

gcloud dns record-sets create "www.$DOMAIN." `
  --zone=portfolio-zone --type=A --ttl=300 --rrdatas=$LB_IP
```

- 루트 도메인(`example.com`)과 `www` 둘 다 등록한다. 방문자가 `www`를 붙여 들어와도 열려야 한다.
- **TTL 300초(5분)** 로 짧게 잡는 이유: 구축 중에는 값을 바꿀 일이 생기는데, TTL이 길면 옛 값이 오래 캐시된다. 운영이 안정되면 3600으로 올려도 된다.

확인:

```powershell
gcloud dns record-sets list --zone=portfolio-zone
```

`NS`, `SOA`(자동 생성) + 방금 만든 `A` 2개, 총 4줄이 보이면 정상이다.

### 6-6. 고대디에서 네임서버 변경 ⚠️ 이 단계가 실제 "위임"

1. https://www.godaddy.com 로그인 → 우측 상단 계정 → **내 제품(My Products)**
2. **도메인** 목록에서 해당 도메인의 **DNS 관리(Manage DNS)** 클릭
3. 페이지 아래로 스크롤 → **네임서버(Nameservers)** 섹션 → **변경(Change)** 클릭
4. **"내 고유 네임서버 사용"** (I'll use my own nameservers) 선택
5. 6-4에서 얻은 **4개**를 입력
   ⚠️ **끝의 점(`.`)은 빼고 입력할 것.** `ns-cloud-a1.googledomains.com` 형태.
   ⚠️ 입력란이 2개만 보이면 **"네임서버 추가"** 를 눌러 4개를 모두 채운다.
6. **저장** → "네임서버를 변경하면 현재 DNS 설정이 적용되지 않습니다" 류의 경고가 뜨면 **계속 진행**
7. 고대디에서 **네임서버 변경 확인 메일**이 오면 링크를 클릭한다 (계정 보안 설정에 따라 올 수도, 안 올 수도 있다). 메일을 무시하면 변경이 적용되지 않는 경우가 있다.

저장 직후 고대디 화면의 네임서버가 `ns-cloud-...googledomains.com` 4개로 바뀌어 보이면 설정은 끝난 것이다. **반영은 지금부터 시간이 걸린다.**

### 6-7. 전파 확인 ⚠️ 여기서 기다려야 한다

전파에 **보통 10분~1시간, 최대 48시간**이 걸린다.

**(1) 권위 서버에 직접 물어보기 — 즉시 확인 가능**

```powershell
Resolve-DnsName $DOMAIN -Type A -Server ns-cloud-a1.googledomains.com
```

> 서버 주소는 6-4에서 받은 것 중 아무거나 하나(끝의 점 빼고). 여기서 LB IP가 나오면 **Cloud DNS 설정 자체는 완벽하다.** 이 확인은 전파와 무관하게 바로 된다.

**(2) 공용 DNS로 전파 상태 확인 — 시간이 걸리는 쪽**

```powershell
Clear-DnsClientCache
Resolve-DnsName $DOMAIN -Type NS -Server 8.8.8.8
Resolve-DnsName $DOMAIN -Type A  -Server 8.8.8.8
```

- NS 결과에 `googledomains.com`이 나오면 → **위임 성공**
- A 결과에 6-2의 LB IP가 나오면 → **전파 완료**

> **(1)은 되는데 (2)가 안 되면 설정은 맞고 전파를 기다리는 중일 뿐이다.** 이 구분을 못 하면 멀쩡한 설정을 계속 뜯어고치게 된다. 이 지점에서 가장 흔한 시간 낭비다.

**(3) 자동 대기 스크립트**

```powershell
while ($true) {
  $ns = (Resolve-DnsName $DOMAIN -Type NS -Server 8.8.8.8 -ErrorAction SilentlyContinue).NameHost
  Write-Host "$(Get-Date -Format 'HH:mm:ss')  NS: $ns"
  if ($ns -match "googledomains") {
    Write-Host "네임서버 위임 완료!" -ForegroundColor Green
    break
  }
  Start-Sleep -Seconds 60
}
```

중단하려면 `Ctrl + C`.

⚠️ **여기서 넘어가면 안 되는 지점이다.** DNS가 LB IP를 가리키기 전에는 이후 단계의 **Google 관리형 SSL 인증서가 절대 발급되지 않는다** (`FAILED_NOT_VISIBLE` 상태로 멈춘다).

> **기다리는 시간을 활용할 것.** 전파를 기다리는 동안 포트폴리오 콘텐츠 작성(이름·경력·프로젝트)을 진행하면 전체 일정이 크게 줄어든다.

### 6-8. 문제 해결

| 증상 | 원인 | 조치 |
|---|---|---|
| `$DOMAIN`이 비어서 이상한 영역이 생성됨 | `set-vars.ps1` 미로드 | `. C:\recruit\set-vars.ps1` 후 재실행. 잘못 만든 영역은 `gcloud dns managed-zones delete` |
| `PERMISSION_DENIED` / API 오류 | `dns.googleapis.com` 미활성 | 5-3 다시 실행 |
| NS 조회에 여전히 고대디 네임서버 | 저장 안 됨 / 전파 전 | 고대디 화면에서 네임서버 4개를 다시 확인, 그 후 대기 |
| 4개 중 1~2개만 입력함 | 입력란 부족 | 고대디에서 "네임서버 추가"로 4개 모두 입력 |
| `Resolve-DnsName` 명령 없음 | 구버전 PowerShell | `nslookup -type=NS 내도메인.com 8.8.8.8` 로 대체 |
| A 조회에 엉뚱한 IP | 고대디 파킹 페이지 캐시 | 위임 완료 후엔 사라진다. `gcloud dns record-sets list`로 Cloud DNS 값부터 확인 |
| 브라우저로 접속하면 아직 안 열림 | **정상** | 지금은 IP만 예약된 상태. 로드밸런서·서버는 아직 없다 |

### [확인] 6번 완료 조건

```powershell
. C:\recruit\set-vars.ps1
gcloud compute addresses describe portfolio-ip --global --format="value(address)"
gcloud dns record-sets list --zone=portfolio-zone
Resolve-DnsName $DOMAIN -Type NS -Server 8.8.8.8
Resolve-DnsName $DOMAIN -Type A  -Server 8.8.8.8
```

- [ ] 고정 글로벌 IP 예약됨 + **IP 주소 메모**
- [ ] Cloud DNS 영역 `portfolio-zone` 생성됨 (`dnsName`이 `내도메인.com.`)
- [ ] 네임서버 4개를 메모함
- [ ] A 레코드 2개(루트 + `www`) 등록됨
- [ ] 고대디 네임서버가 `ns-cloud-*.googledomains.com` 4개로 바뀜
- [ ] `Resolve-DnsName ... -Type NS -Server 8.8.8.8` 결과에 `googledomains.com`이 나옴
- [ ] `Resolve-DnsName ... -Type A -Server 8.8.8.8` 결과가 **예약한 LB IP와 일치**

> 마지막 두 항목이 통과해야 다음 단계(SSL 인증서)로 넘어갈 수 있다. 통과 못 했으면 6-7의 대기 스크립트를 돌려두고, 그동안 콘텐츠 작업을 진행할 것.

---

## 7. GCE 인스턴스 생성 (포트폴리오 실행 서버)

실제로 사이트를 돌릴 서버를 만든다. VM 생성 자체는 3분이면 끝나고 앱과는 무관하므로, **8번의 SSL 인증서 발급 시계를 빨리 돌리기 위해 먼저 한다.**

### 7-1. 이 단계에서 만드는 것

```
[Unmanaged Instance Group]  portfolio-ig   ← 8번의 로드밸런서가 여기를 바라본다
   └─ named port: http:3000
        │
[GCE VM]  portfolio-vm  (e2-small, Ubuntu 22.04, asia-northeast3-a)
   ├─ 시작 스크립트가 부팅 시 Docker 자동 설치
   ├─ 태그: portfolio-web
   └─ 방화벽
        ├─ tcp:3000 ← Google LB 대역만 (헬스체크 + 트래픽)
        └─ tcp:22   ← IAP 터널 대역만 (SSH)
```

### 7-2. `.gitattributes` 추가 ⚠️ Windows 필수

다음 단계에서 만들 시작 스크립트(`.sh`)는 **리눅스에서 실행**된다. Windows Git이 개행을 CRLF로 바꿔 커밋하면 서버에서 `$'\r': command not found` 로 배포가 통째로 실패한다. 1-5의 `core.autocrlf input` 과 짝을 이루는 안전장치를 리포에 넣는다.

VS Code로 **`C:\recruit\portfolio\.gitattributes`** 파일을 만들고:

```
* text=auto eol=lf
*.sh text eol=lf
Dockerfile text eol=lf
*.png binary
*.jpg binary
*.jpeg binary
*.webp binary
*.ico binary
```

### 7-3. 시작 스크립트 생성 ⚠️ 반드시 LF 개행

VM이 부팅될 때 Docker를 자동 설치하는 스크립트다. PowerShell로 **LF 개행 + BOM 없는 UTF-8** 로 안전하게 만든다.

```powershell
Set-Location "C:\recruit\portfolio"
New-Item -ItemType Directory -Force -Path "infra"

$startupScript = @'
#!/bin/bash
set -e

apt-get update
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io

gcloud auth configure-docker asia-northeast3-docker.pkg.dev --quiet

systemctl enable docker
systemctl start docker
'@

$lf = $startupScript -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\infra\startup-script.sh", $lf, (New-Object System.Text.UTF8Encoding $false))

Write-Host "생성 완료: $PWD\infra\startup-script.sh" -ForegroundColor Green
```

⚠️ **`@' ... '@` 는 PowerShell의 "변수를 해석하지 않는" 문자열 블록이다.** 스크립트 안의 `$(dpkg ...)`, `$VERSION_CODENAME` 같은 것들이 PowerShell 변수로 잡히면 안 되기 때문에 작은따옴표 버전을 썼다. **큰따옴표(`@" "@`)로 바꾸면 스크립트가 망가진다.**

커밋해 둔다:

```powershell
git add .gitattributes infra/startup-script.sh
git commit -m "chore: add VM startup script and gitattributes"
git push
```

### 7-4. VM 생성

```powershell
. C:\recruit\set-vars.ps1

gcloud compute instances create portfolio-vm `
  --zone=$ZONE `
  --machine-type=e2-small `
  --image-family=ubuntu-2204-lts `
  --image-project=ubuntu-os-cloud `
  --boot-disk-size=20GB `
  --boot-disk-type=pd-balanced `
  --tags=portfolio-web `
  --scopes=https://www.googleapis.com/auth/cloud-platform `
  --metadata=enable-oslogin=TRUE `
  --metadata-from-file=startup-script=infra/startup-script.sh
```

> 명령은 `C:\recruit\portfolio` 에서 실행해야 한다(`infra/startup-script.sh` 상대 경로 때문).

| 옵션 | 의미 |
|---|---|
| `e2-small` | vCPU 2 / RAM 2GB. Next.js standalone 이미지에 적당한 최소 사양 |
| `--tags=portfolio-web` | 방화벽 규칙을 이 VM에만 적용하기 위한 꼬리표 |
| `--scopes=cloud-platform` | VM이 Artifact Registry에서 이미지를 당겨올 수 있게 함 |
| `enable-oslogin=TRUE` | SSH 키를 IAM으로 관리(안전). GitHub Actions 배포에도 필요 |
| `--metadata-from-file=startup-script` | 부팅 시 7-3 스크립트 자동 실행 |

확인:

```powershell
gcloud compute instances list
```

`STATUS`가 `RUNNING`이면 된다. **단, 이 시점에 Docker 설치는 아직 진행 중**이다(2~3분).

### 7-5. VM에 이미지 읽기 권한 부여

VM이 Artifact Registry에서 이미지를 당겨오려면 기본 컴퓨트 서비스 계정에 읽기 권한이 필요하다.

```powershell
$PROJECT_NUM = gcloud projects describe $PROJECT_ID --format="value(projectNumber)"
Write-Host "프로젝트 번호: $PROJECT_NUM"

gcloud projects add-iam-policy-binding $PROJECT_ID `
  --member="serviceAccount:$PROJECT_NUM-compute@developer.gserviceaccount.com" `
  --role="roles/artifactregistry.reader"
```

⚠️ **프로젝트 번호(숫자)와 프로젝트 ID(문자열)는 다른 값이다.** 위처럼 명령으로 받아서 쓰면 헷갈릴 일이 없다.

### 7-6. 방화벽 규칙

```powershell
# (1) LB 헬스체크 + 트래픽 유입 — Google 전용 대역만 허용
gcloud compute firewall-rules create allow-lb-to-portfolio `
  --allow=tcp:3000 `
  --source-ranges=130.211.0.0/22,35.191.0.0/16 `
  --target-tags=portfolio-web `
  --description="Allow GCLB health check and traffic"

# (2) IAP 터널을 통한 SSH — Google IAP 대역만 허용
gcloud compute firewall-rules create allow-iap-ssh `
  --allow=tcp:22 `
  --source-ranges=35.235.240.0/20 `
  --target-tags=portfolio-web `
  --description="Allow SSH via IAP tunnel"
```

- `130.211.0.0/22`, `35.191.0.0/16` = Google 로드밸런서·헬스체크 대역
- `35.235.240.0/20` = Google IAP 대역

⚠️ **22번 포트를 `0.0.0.0/0`으로 열지 말 것.** 몇 분 안에 전 세계에서 무차별 로그인 시도가 들어온다. IAP 터널로만 접속하는 지금 구조가 훨씬 안전하다.

**⚠️ 기본 방화벽 규칙 확인 — 이게 은근한 함정이다.**

GCP는 프로젝트를 만들 때 `default` 네트워크에 **`default-allow-ssh` (tcp:22, `0.0.0.0/0`)** 규칙을 자동으로 만들어 둔다. 위에서 IAP 전용으로 열어봤자 이게 살아 있으면 22번은 여전히 전 세계에 열려 있다.

```powershell
gcloud compute firewall-rules list --format="table(name,sourceRanges.list(),allowed[].map().firewall_rule().list())"
```

목록에 `default-allow-ssh` 가 `0.0.0.0/0`로 보이면, **7-7의 IAP SSH 접속이 성공하는 것을 먼저 확인한 뒤** 삭제한다:

```powershell
gcloud compute firewall-rules delete default-allow-ssh --quiet
```

⚠️ **순서를 지킬 것.** IAP 접속이 안 되는 상태에서 먼저 지우면 VM에 들어갈 방법이 없어진다(콘솔의 시리얼 콘솔로 복구는 가능하지만 번거롭다).

### 7-7. SSH 접속 테스트 (시작 스크립트 완료 확인)

VM 생성 후 **2~3분** 기다린 뒤:

```powershell
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap --command="docker --version"
```

⚠️ **첫 실행 시 SSH 키 생성 안내가 나오면서 passphrase를 물어본다. 그냥 엔터를 두 번** 치면 된다(비밀번호 없이 생성).

`Docker version 2x.x.x` 가 나오면 성공이다.

안 나오면 시작 스크립트가 아직 안 끝났거나 실패한 것이다. 로그 확인:

```powershell
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap `
  --command="sudo journalctl -u google-startup-scripts --no-pager | tail -30"
```

마지막에 `Finished running startup scripts` 가 보이면 완료된 것이다.

### 7-8. 인스턴스 그룹 생성 (LB 백엔드용)

로드밸런서는 VM을 직접 바라보지 않고 **인스턴스 그룹**을 바라본다.

```powershell
gcloud compute instance-groups unmanaged create portfolio-ig --zone=$ZONE

gcloud compute instance-groups unmanaged add-instances portfolio-ig `
  --zone=$ZONE --instances=portfolio-vm

# LB가 3000 포트로 보내도록 named port 지정
gcloud compute instance-groups unmanaged set-named-ports portfolio-ig `
  --zone=$ZONE --named-ports=http:3000
```

⚠️ **`set-named-ports`를 빼먹으면** 8번에서 백엔드가 영원히 UNHEALTHY로 남는다. 원인 찾기 어려운 대표적인 실수다.

확인:

```powershell
gcloud compute instance-groups unmanaged describe portfolio-ig --zone=$ZONE
```

`namedPorts`에 `name: http, port: 3000` 이 보이면 정상.

### 7-9. 문제 해결

| 증상 | 원인 | 조치 |
|---|---|---|
| `docker: command not found` | 시작 스크립트 미완료/실패 | 2~3분 더 대기 후 재시도, 안 되면 7-7의 로그 확인 |
| SSH 접속이 타임아웃 | IAP 방화벽 규칙 없음 | 7-6의 `allow-iap-ssh` 생성 확인 |
| `Permission denied (publickey)` | OS Login 권한 부족 | 프로젝트 소유자 계정으로 로그인했는지 `gcloud auth list` 확인 |
| `$'\r': command not found` | 시작 스크립트가 CRLF | 7-3을 다시 실행해 파일을 재생성 후 VM 재생성 |
| VM은 RUNNING인데 아무것도 안 됨 | 정상 | 아직 앱 컨테이너가 없다. 11번에서 배포한다 |

### [확인] 7번 완료 조건

```powershell
. C:\recruit\set-vars.ps1
gcloud compute instances list
gcloud compute firewall-rules list --format="table(name,sourceRanges.list())"
gcloud compute instance-groups unmanaged describe portfolio-ig --zone=$ZONE
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap --command="docker --version"
```

- [ ] `portfolio-vm` 이 `RUNNING`
- [ ] SSH 접속되고 `docker --version` 출력됨
- [ ] 방화벽 규칙 2개 생성됨 (`allow-lb-to-portfolio`, `allow-iap-ssh`)
- [ ] `default-allow-ssh` 삭제됨 (IAP 접속 확인 후)
- [ ] 인스턴스 그룹에 VM 등록 + **named port가 `http:3000`**
- [ ] `.gitattributes`, `infra/startup-script.sh` 커밋·푸시됨

---

## 8. HTTPS 로드밸런서 구성 및 도메인 연결

6번의 고정 IP·DNS와 7번의 VM을 연결해서, **도메인으로 들어온 HTTPS 요청이 VM의 3000번 포트까지 도달**하게 만든다.

⚠️ **이 단계를 마치면 요금이 발생하기 시작한다.** 전달 규칙이 만들어지는 순간부터 로드밸런서 과금(월 약 $19)이 시작된다. 크레딧으로 커버되지만, 크레딧 만료일(4-4 메모)을 잊지 말 것.

### 8-1. 전체 흐름

```
https://내도메인.com
   ▼
[Cloud DNS] A 레코드 → 고정 IP (6번에서 완료)
   ▼
[전달 규칙 :443] ──▶ [HTTPS 프록시] ──▶ [URL 맵] ──▶ [백엔드 서비스 + CDN]
        │                   ▲                                  │  헬스체크 :3000/
[전달 규칙 :80]             │                                  ▼
        └──▶ [HTTP 프록시] ─┘                        [인스턴스 그룹] → [VM :3000]
             (301 → HTTPS)         [관리형 SSL 인증서]
```

만드는 순서가 중요하다: **헬스체크 → 백엔드 → 인증서 → URL 맵 → 프록시 → 전달 규칙.** 앞의 것이 없으면 뒤의 것을 못 만든다.

### 8-2. 헬스체크와 백엔드 서비스

```powershell
. C:\recruit\set-vars.ps1

# 헬스체크: 3000번 포트의 / 경로가 200을 주는지 15초마다 확인
gcloud compute health-checks create http portfolio-hc `
  --port=3000 `
  --request-path=/ `
  --check-interval=15s `
  --timeout=5s `
  --healthy-threshold=2 `
  --unhealthy-threshold=3

# 백엔드 서비스 (+ Cloud CDN)
gcloud compute backend-services create portfolio-backend `
  --protocol=HTTP `
  --port-name=http `
  --health-checks=portfolio-hc `
  --global `
  --enable-cdn `
  --cache-mode=CACHE_ALL_STATIC

# 인스턴스 그룹을 백엔드로 연결
gcloud compute backend-services add-backend portfolio-backend `
  --instance-group=portfolio-ig `
  --instance-group-zone=$ZONE `
  --global
```

- `--port-name=http` 는 7-8에서 지정한 named port(`http:3000`)와 짝을 이룬다. **이름이 다르면 트래픽이 안 간다.**
- `--enable-cdn` 은 이미지·JS·CSS 같은 정적 자산을 구글 엣지에 캐싱해 응답을 빠르게 하고 VM 부하도 줄여준다. 콘텐츠 수정 후 반영이 늦어 보이면 13-3의 캐시 무효화를 쓴다.

### 8-3. 관리형 SSL 인증서 생성

⚠️ **전제: 6번의 DNS 전파가 완료되어 있어야 한다.** `Resolve-DnsName $DOMAIN -Type A -Server 8.8.8.8` 결과가 고정 IP와 일치하지 않으면, 인증서는 `FAILED_NOT_VISIBLE` 로 끝난다.

```powershell
gcloud compute ssl-certificates create portfolio-cert `
  --domains="$DOMAIN,www.$DOMAIN" `
  --global
```

- Google이 발급·갱신까지 자동으로 해주는 무료 인증서다. 만료 걱정이 없다.
- 루트 도메인과 `www` 둘 다 넣어야 양쪽 다 자물쇠가 뜬다.

⚠️ **이 명령은 즉시 끝나지만 인증서는 아직 발급되지 않았다.** 아래 8-4~8-5에서 프록시·전달 규칙까지 만들어야 Google이 검증을 시작한다.

### 8-4. URL 맵 + HTTPS 프록시 + 전달 규칙 (443)

```powershell
gcloud compute url-maps create portfolio-urlmap `
  --default-service=portfolio-backend

gcloud compute target-https-proxies create portfolio-https-proxy `
  --url-map=portfolio-urlmap `
  --ssl-certificates=portfolio-cert

gcloud compute forwarding-rules create portfolio-https-rule `
  --global `
  --target-https-proxy=portfolio-https-proxy `
  --address=portfolio-ip `
  --ports=443
```

`--address=portfolio-ip` 로 6-2에서 예약한 고정 IP를 물린다. **여기서 DNS와 로드밸런서가 비로소 이어진다.**

### 8-5. HTTP → HTTPS 리다이렉트 (80)

`http://`로 들어온 요청을 `https://`로 301 넘겨준다. 리다이렉트용 URL 맵은 gcloud 플래그로 못 만들고 YAML을 넣어야 한다.

```powershell
$redirectYaml = @'
name: portfolio-redirect-urlmap
defaultUrlRedirect:
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
  httpsRedirect: true
'@

$lf = $redirectYaml -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\redirect-urlmap.yaml", $lf, (New-Object System.Text.UTF8Encoding $false))

gcloud compute url-maps import portfolio-redirect-urlmap `
  --source=redirect-urlmap.yaml --global --quiet

gcloud compute target-http-proxies create portfolio-http-proxy `
  --url-map=portfolio-redirect-urlmap

gcloud compute forwarding-rules create portfolio-http-rule `
  --global `
  --target-http-proxy=portfolio-http-proxy `
  --address=portfolio-ip `
  --ports=80
```

> `redirect-urlmap.yaml` 은 한 번 쓰고 마는 파일이라 커밋하지 않아도 된다.

### 8-6. 인증서 발급 대기

이제 Google이 도메인 소유를 검증하고 인증서를 발급한다. **보통 15분~1시간, 최대 24시간**이 걸린다.

```powershell
while ($true) {
  $status = gcloud compute ssl-certificates describe portfolio-cert --global --format="value(managed.status)"
  $domainStatus = gcloud compute ssl-certificates describe portfolio-cert --global --format="value(managed.domainStatus)"
  Write-Host "$(Get-Date -Format 'HH:mm:ss')  상태: $status  |  $domainStatus"
  if ($status -eq "ACTIVE") { Write-Host "인증서 발급 완료!" -ForegroundColor Green; break }
  Start-Sleep -Seconds 30
}
```

중단은 `Ctrl + C`.

| 상태 | 의미 | 조치 |
|---|---|---|
| `PROVISIONING` | 발급 진행 중 | 대기 |
| `ACTIVE` | 완료 | 다음 단계로 |
| `FAILED_NOT_VISIBLE` | **DNS가 이 LB IP를 안 가리킴** | 6-6·6-7 다시 확인. DNS를 고친 뒤 인증서를 삭제하고 8-3부터 재생성 |
| `PROVISIONING_FAILED` | 발급 실패 | 인증서 삭제 후 8-3부터 재생성 |

### 8-7. ⚠️ 지금 도메인에 접속하면 502가 뜬다 — 정상이다

인증서가 `ACTIVE`가 돼도, **VM에 아직 앱 컨테이너가 없어서** 백엔드는 UNHEALTHY다.

```powershell
gcloud compute backend-services get-health portfolio-backend --global
```

지금은 `UNHEALTHY` 로 나오는 게 **맞다.** 브라우저로 도메인에 들어가면 `502 Server Error` 가 뜬다.

> 이 상태는 **11번에서 첫 배포가 끝나면 자동으로 해소**된다. 여기서 뭔가 잘못됐다고 판단해 LB를 뜯어고치지 말 것. 이 구간에서 시간을 가장 많이 낭비한다.
>
> 다만 **자물쇠 아이콘(HTTPS)은 지금도 정상**이어야 한다. 502 페이지가 `https://`로 뜨고 인증서 경고가 없으면 8번은 성공이다.

### 8-8. 문제 해결

| 증상 | 원인 | 조치 |
|---|---|---|
| 인증서가 계속 `FAILED_NOT_VISIBLE` | DNS가 LB IP를 안 가리킴 | 6-7의 `-Server 8.8.8.8` 조회부터 다시. 고친 뒤 인증서 재생성 |
| `forwarding-rules create`에서 주소 오류 | 리전 IP를 만들었음 | 6-2를 `--global`로 다시 |
| 백엔드가 계속 UNHEALTHY (11번 이후에도) | named port 누락 | 7-8의 `set-named-ports` 재실행 |
| 백엔드 UNHEALTHY + 방화벽 의심 | LB 대역 미허용 | 7-6의 `allow-lb-to-portfolio` 확인 |
| `www`만 인증서 경고 | 인증서에 `www` 미포함 | 8-3에서 `--domains`에 둘 다 넣었는지 확인 |
| 브라우저에서 502 | **정상** (앱 미배포) | 8-7 참고. 11번까지 진행 |

### [확인] 8번 완료 조건

```powershell
. C:\recruit\set-vars.ps1
gcloud compute ssl-certificates describe portfolio-cert --global --format="value(managed.status)"
gcloud compute forwarding-rules list --global
curl.exe -I "https://$DOMAIN"
curl.exe -I "http://$DOMAIN"
```

⚠️ PowerShell에서 `curl`은 다른 명령의 별칭이라 **반드시 `curl.exe`** 로 입력해야 한다.

- [ ] SSL 인증서 상태가 `ACTIVE`
- [ ] 전달 규칙 2개(80, 443) 생성됨
- [ ] `https://도메인` 접속 시 **자물쇠 아이콘 정상** (내용은 502여도 됨)
- [ ] `http://도메인` 이 301로 https에 리다이렉트됨
- [ ] 백엔드가 UNHEALTHY인 것을 **정상으로 인지**하고 넘어감

---

## 9. 앱 컨테이너화 (Dockerfile)

여기부터는 코드 작업이다. **8번의 인증서 발급을 기다리는 동안 병행하면 시간을 크게 아낄 수 있다.**

### 9-1. `next.config.mjs`에 standalone 출력 추가

`C:\recruit\portfolio\next.config.mjs` 를 열고 `nextConfig` 객체 안에 한 줄 추가:

```js
const nextConfig = {
  output: "standalone",              // ← 이 줄 추가
  pageExtensions: ["ts", "tsx", "md", "mdx"],
  // ... 기존 설정은 그대로 유지
};
```

⚠️ **이게 있어야 런타임 이미지가 1GB대 → 200MB대로 줄어든다.** RAM 2GB짜리 e2-small에서 돌리려면 사실상 필수다. 빌드에 필요한 파일만 추린 `.next/standalone` 디렉터리가 만들어진다.

### 9-2. `Dockerfile` 생성 (프로젝트 루트)

`C:\recruit\portfolio\Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1

# 1) 의존성 설치
FROM node:22-alpine AS deps
WORKDIR /app
RUN apk add --no-cache libc6-compat
COPY package.json package-lock.json* ./
RUN npm ci

# 2) 빌드
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# 3) 실행
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME=0.0.0.0

RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

- **3단계 멀티스테이지**: 의존성 → 빌드 → 실행. 최종 이미지에는 빌드 도구가 안 들어간다.
- `HOSTNAME=0.0.0.0` 이 없으면 컨테이너 내부에서만 접속되고 **헬스체크가 실패한다.**
- `npm ci` 는 `package-lock.json` 이 리포에 커밋돼 있어야 동작한다(템플릿에 이미 포함돼 있다).

### 9-3. `.dockerignore` 생성

`C:\recruit\portfolio\.dockerignore`:

```
node_modules
.next
.git
.github
README.md
*.md
.env*.local
Dockerfile
.dockerignore
infra
docs
```

빌드 컨텍스트를 줄여서 빌드 속도를 올리고, 불필요한 파일이 이미지에 들어가는 것을 막는다.

### 9-4. 로컬에서 컨테이너 검증 ⚠️ 이 단계가 핵심

Docker Desktop이 실행 중인지(고래 아이콘) 확인하고:

```powershell
Set-Location "C:\recruit\portfolio"
docker build -t portfolio:local .
docker run --rm -p 3000:3000 portfolio:local
```

브라우저에서 http://localhost:3000 접속 → 포트폴리오 화면이 뜨면 성공. `Ctrl + C`로 종료.

> ⚠️ **여기까지 통과해야 다음으로 넘어갈 것.** 로컬 컨테이너에서 안 뜨는 이미지는 GCE에서도 안 뜬다. 원인을 여기서 잡는 게 GitHub Actions 로그를 뒤지는 것보다 10배 빠르다.

| 증상 | 조치 |
|---|---|
| `docker: error during connect` | Docker Desktop 미실행 — 실행 후 고래 아이콘이 멈출 때까지 대기 |
| `npm ci` 실패 | `package-lock.json` 이 있는지 확인 |
| 빌드는 되는데 화면이 안 뜸 | `output: "standalone"` 과 `HOSTNAME=0.0.0.0` 을 넣었는지 확인 |
| 빌드가 메모리 부족으로 죽음 | Docker Desktop → Settings → Resources에서 메모리를 4GB 이상으로 |

### 9-5. 커밋·푸시

```powershell
git add next.config.mjs Dockerfile .dockerignore
git commit -m "feat: add Dockerfile and standalone output for GCE deployment"
git push
```

### [확인] 9번 완료 조건

- [ ] `next.config.mjs` 에 `output: "standalone"` 추가됨
- [ ] `Dockerfile`, `.dockerignore` 생성됨
- [ ] `docker build` 성공
- [ ] `docker run` 으로 localhost:3000에 화면이 정상 표시됨
- [ ] GitHub에 푸시됨

---

## 10. Artifact Registry + 배포용 서비스 계정 + GitHub Secrets

GitHub Actions가 이미지를 올려둘 **창고**와, GCP에 접근할 **로봇 계정**을 만든다. 11번 워크플로우가 쓸 재료 준비 단계다.

### 10-1. Artifact Registry 저장소 생성

```powershell
. C:\recruit\set-vars.ps1

gcloud artifacts repositories create $REPO `
  --repository-format=docker `
  --location=$REGION `
  --description="Portfolio container images"
```

확인:

```powershell
gcloud artifacts repositories list --location=$REGION
```

이미지 경로 형식은 이렇게 된다:

```
asia-northeast3-docker.pkg.dev/<프로젝트ID>/portfolio-docker/portfolio:<태그>
```

### 10-2. 서비스 계정 생성 및 권한 부여

```powershell
gcloud iam service-accounts create $SA_NAME --display-name="GitHub Actions Deployer"
```

권한 5개를 순서대로 부여한다. **하나라도 빠지면 배포가 그 지점에서 멈춘다.**

```powershell
# 이미지 푸시
gcloud projects add-iam-policy-binding $PROJECT_ID `
  --member="serviceAccount:$SA_EMAIL" --role="roles/artifactregistry.writer"

# VM 조회
gcloud projects add-iam-policy-binding $PROJECT_ID `
  --member="serviceAccount:$SA_EMAIL" --role="roles/compute.viewer"

# SSH 접속 (OS Login)
gcloud projects add-iam-policy-binding $PROJECT_ID `
  --member="serviceAccount:$SA_EMAIL" --role="roles/compute.osAdminLogin"

# IAP 터널을 통한 SSH
gcloud projects add-iam-policy-binding $PROJECT_ID `
  --member="serviceAccount:$SA_EMAIL" --role="roles/iap.tunnelResourceAccessor"

# VM의 서비스 계정 대행 (SSH 시 필요)
gcloud projects add-iam-policy-binding $PROJECT_ID `
  --member="serviceAccount:$SA_EMAIL" --role="roles/iam.serviceAccountUser"
```

| 권한 | 없으면 생기는 증상 |
|---|---|
| `artifactregistry.writer` | 이미지 푸시 단계에서 403 |
| `compute.viewer` | VM을 못 찾음 |
| `compute.osAdminLogin` | SSH `Permission denied` |
| `iap.tunnelResourceAccessor` | SSH 단계에서 타임아웃 |
| `iam.serviceAccountUser` | SSH 시작 자체가 거부됨 |

확인:

```powershell
gcloud projects get-iam-policy $PROJECT_ID `
  --flatten="bindings[].members" `
  --filter="bindings.members:$SA_EMAIL" `
  --format="value(bindings.role)"
```

5줄이 나오면 정상.

### 10-3. 키 발급 후 GitHub Secret 등록

```powershell
gcloud iam service-accounts keys create "C:\recruit\gha-key.json" --iam-account=$SA_EMAIL

# 클립보드로 복사
Get-Content "C:\recruit\gha-key.json" -Raw | Set-Clipboard
```

⚠️ **경로를 반드시 `C:\recruit\` 바로 아래로 할 것. `C:\recruit\portfolio\` 안은 절대 안 된다.**

| 경로 | 가능 여부 |
|---|---|
| `C:\recruit\gha-key.json` | ✅ 깃 리포 **바깥**이라 커밋될 수 없음 |
| `C:\recruit\portfolio\gha-key.json` | ❌ **깃 리포 안.** `git add .` 한 번에 Public 리포로 유출됨 |

`C:\recruit\portfolio` 만 깃 저장소이고 그 부모인 `C:\recruit` 는 깃과 무관하다. 이 한 칸 차이가 유출 여부를 가른다.

GitHub 리포 → **Settings** → 좌측 **Secrets and variables** → **Actions** → **New repository secret**

| Secret 이름 | 값 |
|---|---|
| `GCP_SA_KEY` | 방금 복사한 JSON **전체** (`{` 부터 `}` 까지 그대로 붙여넣기) |
| `GCP_PROJECT_ID` | 프로젝트 ID (예: `portfolio-XXXXXX`) |

⚠️ **JSON은 줄바꿈 포함 전체를 넣어야 한다.** 일부만 붙여넣으면 인증 단계에서 파싱 오류가 난다.

### 10-4. 로컬 키 파일 즉시 삭제 ⚠️

```powershell
Remove-Item "C:\recruit\gha-key.json"

# 삭제 확인 (아무것도 안 나와야 정상)
Get-ChildItem "C:\recruit" -Filter "*.json"
```

이 파일은 **GCP 프로젝트 전체에 대한 열쇠**다. GitHub Secret에 등록했으면 로컬 파일은 더 이상 필요 없으니 즉시 지운다.

> ⚠️ 이 키를 **리포지토리에 커밋하는 일은 절대 없어야 한다.** Public 리포라 커밋되는 순간 자동 스캐너에 걸려 몇 분 만에 악용될 수 있다. `.gitignore`에 `*.json` 을 넣는 건 해법이 아니다(프로젝트 설정 파일 중에 `.json`이 많아서 부작용이 크다). **애초에 리포 폴더 바깥(`C:\recruit\`)에 만들었다가 바로 지우는** 지금 방식이 가장 안전하다.
>
> **만약 실수로 커밋·푸시했다면**, 파일을 지우는 것만으로는 부족하다(히스토리에 남는다). 즉시 키를 무효화할 것:
>
> ```powershell
> gcloud iam service-accounts keys list --iam-account=$SA_EMAIL
> gcloud iam service-accounts keys delete <KEY_ID> --iam-account=$SA_EMAIL
> ```
>
> 그 후 10-3부터 새 키를 발급받아 Secret을 다시 등록한다.

> 더 안전한 방식은 키가 아예 없는 **Workload Identity Federation**이다. 이번엔 단순화를 위해 키 방식으로 가고, 여유가 생기면 **부록 C**로 전환하면 된다.

### [확인] 10번 완료 조건

- [ ] Artifact Registry 저장소 `portfolio-docker` 생성됨
- [ ] 서비스 계정 생성 + **권한 5개** 부여 확인
- [ ] GitHub Secret 2개(`GCP_SA_KEY`, `GCP_PROJECT_ID`) 등록됨
- [ ] 로컬 키 파일 삭제 완료
- [ ] 키 파일이 리포지토리에 커밋되지 않았음

---

## 11. GitHub Actions 워크플로우 작성 및 첫 배포

`main`에 푸시하면 자동으로 빌드·배포되는 파이프라인을 만든다. **이 단계가 끝나면 8-7의 502가 해소되고 실제 사이트가 뜬다.**

### 11-1. 워크플로우 파일 생성

```powershell
Set-Location "C:\recruit\portfolio"
New-Item -ItemType Directory -Force -Path ".github\workflows"
```

VS Code로 `.github/workflows/deploy.yml` 을 만들고:

```yaml
name: Deploy Portfolio to GCE

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  GCP_ZONE: asia-northeast3-a
  ARTIFACT_REGISTRY: asia-northeast3-docker.pkg.dev
  REPO_NAME: portfolio-docker
  IMAGE_NAME: portfolio
  VM_NAME: portfolio-vm

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v3
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v3

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker ${{ env.ARTIFACT_REGISTRY }} --quiet

      - name: Build and push image
        run: |
          IMAGE_BASE="${{ env.ARTIFACT_REGISTRY }}/${{ env.GCP_PROJECT_ID }}/${{ env.REPO_NAME }}/${{ env.IMAGE_NAME }}"
          docker build -t "$IMAGE_BASE:${{ github.sha }}" -t "$IMAGE_BASE:latest" .
          docker push "$IMAGE_BASE:${{ github.sha }}"
          docker push "$IMAGE_BASE:latest"

      - name: Deploy to GCE VM
        run: |
          IMAGE_BASE="${{ env.ARTIFACT_REGISTRY }}/${{ env.GCP_PROJECT_ID }}/${{ env.REPO_NAME }}/${{ env.IMAGE_NAME }}"
          gcloud compute ssh ${{ env.VM_NAME }} \
            --zone=${{ env.GCP_ZONE }} \
            --tunnel-through-iap \
            --quiet \
            --command="
              set -e
              sudo gcloud auth configure-docker ${{ env.ARTIFACT_REGISTRY }} --quiet
              sudo docker pull $IMAGE_BASE:${{ github.sha }}
              sudo docker stop portfolio || true
              sudo docker rm portfolio || true
              sudo docker run -d --name portfolio --restart always -p 3000:3000 $IMAGE_BASE:${{ github.sha }}
              sudo docker image prune -af --filter 'until=168h'
            "

      - name: Verify deployment
        run: |
          gcloud compute ssh ${{ env.VM_NAME }} \
            --zone=${{ env.GCP_ZONE }} \
            --tunnel-through-iap \
            --quiet \
            --command="sleep 10 && curl -sf -o /dev/null -w '%{http_code}' http://localhost:3000 && echo ' OK'"
```

**동작 요약**

1. 코드 체크아웃
2. `GCP_SA_KEY`로 GCP 인증
3. 이미지를 빌드해 Artifact Registry에 푸시 (커밋 해시 + `latest` 두 태그)
4. IAP 터널로 VM에 SSH → 새 이미지 pull → 기존 컨테이너 교체
5. VM 안에서 `localhost:3000`이 200을 주는지 확인

> `--restart always` 덕분에 VM이 재부팅돼도 컨테이너가 자동으로 다시 뜬다.
> `docker image prune` 은 7일 지난 옛 이미지를 정리해 디스크가 차는 것을 막는다.

⚠️ **YAML은 들여쓰기가 문법이다.** 복사할 때 들여쓰기가 깨지지 않았는지 확인할 것. VS Code에서 파일 우하단 인코딩이 `UTF-8`, 개행이 `LF`인지도 확인(7-2의 `.gitattributes`가 처리해준다).

### 11-2. 커밋·푸시 → 자동 배포 시작

```powershell
git add .github/workflows/deploy.yml
git commit -m "ci: add GitHub Actions deployment to GCE"
git push
```

GitHub 리포 → **Actions** 탭에서 진행 상황을 볼 수 있다. **첫 배포는 5~8분** 걸린다(이미지 빌드 때문). 두 번째부터는 캐시가 있어 더 빠르다.

각 단계 왼쪽에 초록색 체크가 차례로 켜지고, 마지막 `Verify deployment` 에서 `200 OK` 가 나오면 성공이다.

### 11-3. 배포 확인 — 502가 사라지는 순간

**(1) 컨테이너가 떴는지**

```powershell
. C:\recruit\set-vars.ps1
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap --command="sudo docker ps"
```

`portfolio` 컨테이너가 `Up` 상태로 보이면 된다.

**(2) LB 백엔드가 HEALTHY로 바뀌었는지** ⚠️ 여기가 8-7의 해소 지점

```powershell
gcloud compute backend-services get-health portfolio-backend --global
```

`healthState: HEALTHY` 가 나오면 성공이다. 컨테이너가 뜬 뒤 **헬스체크 2회 통과(약 30초)** 가 필요하므로, 바로 안 바뀌면 30초쯤 기다렸다 다시 조회한다.

**(3) 도메인으로 실제 접속**

```powershell
curl.exe -I "https://$DOMAIN"
```

`HTTP/2 200` 이 나오면 끝이다. 브라우저에서 `https://내도메인.com` 을 열면 포트폴리오 화면이 뜬다.

### 11-4. 첫 실행에서 자주 나는 오류

| 증상 | 원인 | 해결 |
|---|---|---|
| `Permission denied on resource project` | 서비스 계정 권한 누락 | 10-2의 권한 5개 다시 확인 |
| 인증 단계에서 JSON 파싱 오류 | Secret에 JSON 일부만 붙여넣음 | `GCP_SA_KEY` 를 전체 JSON으로 다시 등록 |
| SSH 단계에서 타임아웃 | IAP 방화벽 규칙 없음 | 7-6의 `allow-iap-ssh` 확인 |
| `docker: not found` (VM 쪽) | 시작 스크립트 미완료/실패 | 7-7의 로그 확인 명령 실행 |
| 이미지 pull 권한 오류 | VM 서비스 계정 권한 없음 | 7-5 다시 실행 |
| `npm ci` 실패 | `package-lock.json` 미커밋 | `git add package-lock.json` 후 푸시 |
| `$'\r': command not found` | 개행이 CRLF | 1-5의 `core.autocrlf input` + 7-2의 `.gitattributes` 확인 후 파일 재생성·재커밋 |
| 배포는 성공인데 여전히 502 | 헬스체크 대기 중 | 30초~1분 후 `get-health` 재조회. 계속 UNHEALTHY면 7-8의 named port 확인 |

### [확인] 11번 완료 조건

- [ ] Actions 워크플로우가 **초록색 체크**로 끝남
- [ ] `Verify deployment` 단계에서 `200 OK` 출력
- [ ] `docker ps` 에 `portfolio` 컨테이너가 `Up`
- [ ] `get-health` 결과가 **`HEALTHY`**
- [ ] **`https://내도메인.com` 에서 포트폴리오 화면이 실제로 보임** ← 배포 성공의 최종 기준

---

## 12. 콘텐츠 커스터마이징

인프라가 다 돌아가는 상태이므로, 이제부터는 **파일 수정 → 푸시 → 2~4분 후 자동 반영**이다. 조카가 시간을 가장 많이 써야 하는 단계이자, 실제 취업 성과를 좌우하는 부분이다.

```powershell
Set-Location "C:\recruit\portfolio"
code .          # VS Code로 프로젝트 열기
```

### 12-1. `baseURL`을 가장 먼저 바꿀 것 ⚠️

`src/resources/once-ui.config.ts` 17번째 줄 근처:

```ts
const baseURL: string = "https://demo.magic-portfolio.com";   // ← 기본값
```

를 구매한 도메인으로 바꾼다:

```ts
const baseURL: string = "https://내도메인.com";
```

⚠️ **메타태그, OG 이미지, sitemap, robots.txt가 전부 이 값을 기준으로 생성된다.** 기본값이 남아 있으면 구글 검색 노출이 깨지고, 카톡·슬랙에 링크를 붙였을 때 미리보기가 **남의 사이트 이미지로 뜬다.** 다른 어떤 수정보다 먼저 할 것.

### 12-2. 수정 대상

| 대상 | 경로 | 내용 |
|---|---|---|
| 사이트 설정 | `src/resources/once-ui.config.ts` | **`baseURL`**, 테마·컬러 |
| 개인 정보·경력 | `src/resources/content.tsx` | 이름, 직함, 한 줄 소개, 이메일, 소셜 링크, 학력, 경력, 기술 스택 |
| 프로젝트 | `src/app/work/projects/*.mdx` | 프로젝트별 MDX 파일. **샘플 삭제 후 본인 것으로 교체** |
| 블로그 | `src/app/blog/posts/*.mdx` | 선택. 안 쓸 거면 `content.tsx`에서 메뉴 숨김 |
| 이미지 | `public/images/` | `avatar.jpg`(프로필), `projects/`(스크린샷), `og/`(링크 미리보기) |

⚠️ **템플릿 샘플 글이 하나라도 남아 있으면 안 된다.** `src/app/blog/posts/` 에 있는 `quick-start.mdx`, `styling.mdx` 같은 Once UI 사용법 글들이 그대로 배포되면 완성도가 크게 떨어져 보인다. 안 쓸 파일은 삭제할 것.

### 12-3. 로컬에서 실시간 확인하며 작업

```powershell
npm run dev
```

http://localhost:3000 을 켜둔 채 파일을 저장하면 브라우저가 자동 갱신된다. **콘텐츠 작업 내내 켜두는 게 편하다.** 매번 푸시해서 확인하면 한 번에 3~4분씩 버린다.

### 12-4. 이력서 관점 작성 팁

- **써클 인턴 경험을 구체적으로**: 기간(2026-XX ~ 2026-09-04), 담당 업무, 사용 기술을 **성과 중심**으로.
  ⚠️ **재직 중에 정리해두는 게 정확하다.** 나가고 나면 세부 내용과 수치가 흐려진다.
- **프로젝트는 3~5개**가 적정. 개수보다 각각의 **"문제 → 해결 → 결과"** 가 명확한 게 중요하다.
- **이 포트폴리오 사이트 자체를 프로젝트로 등록할 것.** Next.js + Docker + GitHub Actions + GCP(GCE/LB/Cloud DNS) 구성도를 넣으면 신입 기준으로 강한 차별점이 된다. 문서 앞의 「최종 목표 아키텍처」 그림을 그대로 써도 좋다.
- 연락 가능한 **이메일과 GitHub 링크는 첫 화면에서 바로 보이게.**
- 라이선스상 **푸터의 Once UI 크레딧은 지우지 말 것** (2-5 참고).

### 12-5. 배포

```powershell
git add .
git commit -m "feat: customize portfolio content"
git push
```

Actions 탭에서 초록불을 확인하고, 2~4분 뒤 도메인에 접속해 반영을 확인한다.

> 수정이 반영 안 된 것처럼 보이면 **CDN 캐시** 때문일 수 있다. 13-3의 무효화 명령을 쓴다.

### [확인] 12번 완료 조건

- [ ] `baseURL` 이 구매한 도메인으로 변경됨
- [ ] 이름·소개·연락처가 본인 것으로 교체됨
- [ ] 템플릿 샘플 프로젝트/블로그 글이 **전부 정리**됨
- [ ] 프로필 사진·프로젝트 이미지 교체됨
- [ ] 푸터의 Once UI 크레딧이 남아 있음
- [ ] 푸시 후 도메인에서 수정 내용이 보임

---

## 13. 최종 검증 및 운영

### 13-1. 인프라 상태 일괄 점검

```powershell
. C:\recruit\set-vars.ps1

# 1) 컨테이너 동작
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap --command="sudo docker ps"

# 2) LB 백엔드 헬스 (HEALTHY 여야 함)
gcloud compute backend-services get-health portfolio-backend --global

# 3) 인증서 (ACTIVE 여야 함)
gcloud compute ssl-certificates describe portfolio-cert --global --format="value(managed.status)"

# 4) HTTP 응답
curl.exe -I "https://$DOMAIN"
curl.exe -I "http://$DOMAIN"     # 301 리다이렉트 확인
```

### 13-2. 브라우저 체크리스트

- [ ] `https://도메인` 정상 표시, 주소창에 **자물쇠 아이콘**
- [ ] `http://도메인` → https로 자동 전환
- [ ] `www.도메인` 도 접속됨
- [ ] **휴대폰에서 접속** 시 레이아웃 정상
- [ ] **카카오톡·슬랙에 링크 붙여넣기** → 썸네일 미리보기 정상 (`baseURL` 확인)
- [ ] `https://도메인/sitemap.xml` 정상
- [ ] `https://도메인/robots.txt` 정상
- [ ] 다크/라이트 모드 전환 정상

### 13-3. 수정이 반영 안 될 때 (CDN 캐시)

```powershell
gcloud compute url-maps invalidate-cdn-cache portfolio-urlmap --path="/*" --global
```

1~2분 뒤 반영된다. 자주 쓸 일은 아니고, 배포는 성공했는데 화면이 그대로일 때만 쓴다.

### 13-4. 일상 운영

**콘텐츠 수정 → 반영** (앞으로 이것만 반복하면 된다):

```powershell
Set-Location "C:\recruit\portfolio"
# 파일 수정 후
git add .
git commit -m "update: 내용 수정"
git push
# → GitHub Actions가 자동 배포 (2~4분)
```

**로그 확인:**

```powershell
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap --command="sudo docker logs --tail 50 portfolio"
```

**VM 재시작:**

```powershell
gcloud compute instances reset portfolio-vm --zone=$ZONE
```

**정기 점검:**
- 도메인 자동갱신 여부 (연 1회)
- 예산 알림 메일 확인 (월 1회)
- 크레딧 잔액·만료일 (4-4 메모)

### 13-5. 크레딧 만료(약 2026년 11월) 시 선택지 ⚠️

현재 구성의 월 비용은 **약 $39**다. 항목별 내역은 문서 앞의 **「시작 전 알아둘 3가지 → 2. 무료 크레딧의 함정」** 표를 참고할 것. 크레딧이 만료되면 이 금액이 실제로 청구되기 시작하므로, 만료 전에 아래 중 하나를 선택해야 한다.

| 선택지 | 월 비용 | 작업량 | 비고 |
|---|---|---|---|
| **A. 그대로 유지** | 약 $39 | 없음 | 취업 확정까지 짧게 쓸 거면 무난 |
| **B. LB 제거 + VM에 Caddy** | 약 $10 | 반나절 | VM에 Caddy를 얹어 Let's Encrypt 자동 발급. DNS를 VM 고정 IP로 변경 |
| **C. Cloud Run 전환** | 약 $0~2 | 반나절 | 트래픽 없으면 0으로 스케일. **개인 포폴엔 사실상 최적** |
| **D. Vercel 무료 플랜** | $0 | 1시간 | Next.js 원저작 플랫폼. 다만 인프라 학습 요소가 사라짐 |

**권장**: 취업 활동이 끝날 때까지는 A로 두고, 장기 보관용으로 넘어갈 때 C(Cloud Run)로 전환. 이력서에 "GCE+LB로 구성 후 비용 최적화를 위해 Cloud Run으로 마이그레이션"이라고 쓸 수 있어서 오히려 스토리가 하나 더 생긴다.

### 13-6. 전체 정리(삭제)

더 이상 안 쓸 때 과금을 끊는 순서다. **의존 관계가 있어서 이 순서를 지켜야 한다.**

```powershell
. C:\recruit\set-vars.ps1

gcloud compute forwarding-rules delete portfolio-https-rule --global --quiet
gcloud compute forwarding-rules delete portfolio-http-rule --global --quiet
gcloud compute target-https-proxies delete portfolio-https-proxy --quiet
gcloud compute target-http-proxies delete portfolio-http-proxy --quiet
gcloud compute url-maps delete portfolio-urlmap --global --quiet
gcloud compute url-maps delete portfolio-redirect-urlmap --global --quiet
gcloud compute ssl-certificates delete portfolio-cert --global --quiet
gcloud compute backend-services delete portfolio-backend --global --quiet
gcloud compute health-checks delete portfolio-hc --quiet
gcloud compute instance-groups unmanaged delete portfolio-ig --zone=$ZONE --quiet
gcloud compute instances delete portfolio-vm --zone=$ZONE --quiet
gcloud compute addresses delete portfolio-ip --global --quiet
gcloud dns record-sets delete "$DOMAIN." --zone=portfolio-zone --type=A --quiet
gcloud dns record-sets delete "www.$DOMAIN." --zone=portfolio-zone --type=A --quiet
gcloud dns managed-zones delete portfolio-zone --quiet
gcloud artifacts repositories delete $REPO --location=$REGION --quiet
```

가장 확실한 방법은 **프로젝트 자체 종료**다. 콘솔 → IAM 및 관리자 → 설정 → **종료**. 30일 후 완전 삭제되고 그 전엔 복구할 수 있다.

### 13-7. 마무리 작업

- [ ] **GitHub 프로필**(https://github.com/settings/profile)의 **Website 칸에 도메인 등록** (2-6에서 미뤄둔 것)
- [ ] 이력서·구직 사이트 프로필에 도메인 기재
- [ ] Google Search Console(https://search.google.com/search-console)에 도메인 등록 → 검색 노출용 (선택)
- [ ] 크레딧 만료일 캘린더 알림 확인 (4-4)

### [확인] 13번 완료 조건 — 전체 완료

- [ ] 13-1의 4개 명령이 모두 정상 (컨테이너 Up / 백엔드 HEALTHY / 인증서 ACTIVE / 200 응답)
- [ ] 13-2 브라우저 체크리스트 전부 통과
- [ ] 수정 → 푸시 → 자동 반영 사이클을 **한 번 이상 직접 성공**시켜 봄
- [ ] GitHub 프로필과 이력서에 도메인 등록됨
- [ ] 크레딧 만료일과 예산 알림이 캘린더/메일에 걸려 있음


---

# 부록

## 부록 A. 이틀 일정표

**병목은 6번(DNS 전파)과 8번(SSL 인증서 발급)뿐이다.** 둘 다 그냥 기다리는 시간이므로, 그 사이에 코드 작업을 끼워 넣는 것이 이 순서의 핵심이다.

### 1일차 — 계정·인프라 뼈대 (약 6시간 + 대기)

| 시간 | 항목 | 비고 |
|---|---|---|
| 0:00~1:30 | **1. 개발 환경 구성** | ⚠️ **재부팅 2회** (WSL2, Docker Desktop). 시간이 가장 많이 드는 구간 |
| 1:30~2:10 | **2. GitHub 계정 + 리포 이관** | 2FA 복구 코드 저장 잊지 말 것 |
| 2:10~2:30 | **3. 도메인 구매** | 후보 2~3개를 미리 정해두면 빠르다 |
| 2:30~2:50 | **4. GCP 가입** | 카드 해외결제 차단 여부가 관건 |
| 2:50~3:20 | **5. 프로젝트 + gcloud** | `set-vars.ps1` 작성 |
| 3:20~3:40 | **6. Cloud DNS + 네임서버 위임** | ⏳ **여기서 DNS 시계 시작** |
| 3:40~4:10 | **7. GCE 인스턴스** | DNS 기다리는 동안 진행 |
| 4:10~4:30 | **8. HTTPS LB + 인증서 생성** | ⏳ **여기서 인증서 시계 시작.** DNS 전파가 끝나 있어야 함 |
| 4:30~5:10 | **9. 앱 컨테이너화** | 인증서 기다리는 동안 진행 |
| 5:10~5:40 | **10. AR + 서비스 계정** | 〃 |
| 5:40~6:20 | **11. GitHub Actions 첫 배포** | 〃. **여기서 502가 사라진다** |

> ⚠️ **3:20의 네임서버 위임을 최대한 앞당기는 것이 전체 일정을 좌우한다.** 도메인을 미리 사둘 수 있다면 3번을 전날 끝내두는 것도 방법이다.

### 2일차 — 콘텐츠와 마무리

| 시간 | 항목 |
|---|---|
| 0:00~4:00 | **12. 콘텐츠 커스터마이징** (실제로는 며칠 걸릴 수 있다) |
| 4:00~4:30 | **13. 최종 검증 및 운영 설정** |

### 반나절만 있을 때

1일차의 1~11번을 다 하기는 어렵다. 우선순위는 이렇다:

1. **1~6번을 무조건 먼저** — 특히 6번(네임서버 위임)까지 끝내면 대기가 백그라운드로 돌아간다
2. 7~8번으로 인증서 시계까지 돌려둔다
3. 9~11번은 다음 기회에 (사이트는 502지만 인프라는 살아 있다)

---

## 부록 B. Windows 특화 문제 모음

여러 항목에 걸쳐 나타나는 문제를 한 곳에 모았다. **막히면 여기부터 찾아볼 것.**

| 증상 | 원인 | 해결 |
|---|---|---|
| `'gcloud'은(는) 내부 또는 외부 명령... 이 아닙니다` | PATH 미갱신 | **터미널을 닫고 새로 열기.** 그래도 안 되면 PC 재부팅 (1-4) |
| `docker: error during connect` | Docker Desktop 미실행 | 시작 메뉴에서 Docker Desktop 실행 → 고래 아이콘이 멈출 때까지 대기 (1-9) |
| `이 시스템에서 스크립트를 실행할 수 없으므로...` | 실행 정책 | `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` (1-2) |
| `$'\r': command not found` (배포 시) | 개행이 CRLF | `core.autocrlf input`(1-5) + `.gitattributes`(7-2) 설정 후 **파일 재생성·재커밋** |
| `curl -I` 결과가 이상함 | PowerShell 별칭 | **`curl.exe -I`** 로 입력 (`curl`은 `Invoke-WebRequest` 별칭) |
| `python`을 쳤는데 Microsoft Store가 열림 | 앱 실행 별칭 | 설정 → 앱 → 고급 앱 설정 → 앱 실행 별칭에서 `python.exe` 끄기 (1-6) |
| `$DOMAIN` 등 변수가 비어 있음 | 닷 소싱 안 함 | **`. C:\recruit\set-vars.ps1`** — 맨 앞의 점과 공백 필수 (5-4) |
| WSL 설치 실패 | BIOS 가상화 비활성 | BIOS에서 Virtualization / Intel VT-x / AMD-V(SVM) 활성화 (1-8) |
| `npm install` 권한 오류 | 백신 프로그램 간섭 | `C:\recruit` 폴더를 백신 실시간 검사 예외로 추가 |
| 콘솔에 한글이 깨짐 | 인코딩 | `chcp 65001` 실행 후 재시도 |
| `C:\recruit` 폴더 생성 시 액세스 거부 | C 드라이브 루트 권한 | **관리자 권한 PowerShell**로 실행 (1-1) |
| 탐색기에서 `.github` 폴더가 안 만들어짐 | 구버전 탐색기 제약 | 이름을 `.github.` 처럼 **뒤에도 점**을 찍으면 만들어진다. PowerShell `New-Item`을 쓰면 애초에 문제없다 |
| 도커 빌드가 메모리 부족으로 죽음 | Docker Desktop 메모리 할당 | Settings → Resources에서 4GB 이상으로 (9-4) |

---

## 부록 C. Workload Identity Federation 전환 (선택)

10-3의 서비스 계정 키(JSON)를 GitHub Secret에 두는 방식은 **키가 유출되면 그대로 뚫린다.** 여유가 생기면 키가 아예 없는 방식으로 바꾸는 것을 권장한다.

### 설정

```powershell
. C:\recruit\set-vars.ps1

gcloud iam workload-identity-pools create github-pool `
  --location=global --display-name="GitHub Pool"

gcloud iam workload-identity-pools providers create-oidc github-provider `
  --location=global `
  --workload-identity-pool=github-pool `
  --issuer-uri="https://token.actions.githubusercontent.com" `
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" `
  --attribute-condition="assertion.repository=='조카아이디/portfolio'"

$PROJECT_NUM = gcloud projects describe $PROJECT_ID --format="value(projectNumber)"

gcloud iam service-accounts add-iam-policy-binding $SA_EMAIL `
  --role=roles/iam.workloadIdentityUser `
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUM/locations/global/workloadIdentityPools/github-pool/attribute.repository/조카아이디/portfolio"
```

⚠️ `attribute-condition`의 `조카아이디/portfolio` 를 **실제 리포 경로로 정확히 교체할 것.** 이 조건이 "이 리포지토리에서 실행된 워크플로우만 허용"을 보장한다.

### 워크플로우 수정

`.github/workflows/deploy.yml` 의 `jobs.deploy` 아래에 권한을 추가하고:

```yaml
permissions:
  contents: read
  id-token: write
```

인증 스텝에서 `credentials_json` 대신:

```yaml
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: projects/<PROJECT_NUM>/locations/global/workloadIdentityPools/github-pool/providers/github-provider
          service_account: <SA_EMAIL>
```

### 전환 확인 후 기존 키 삭제

워크플로우가 정상 동작하는 것을 확인한 뒤:

```powershell
gcloud iam service-accounts keys list --iam-account=$SA_EMAIL
gcloud iam service-accounts keys delete <KEY_ID> --iam-account=$SA_EMAIL
```

GitHub Secret의 `GCP_SA_KEY` 도 삭제한다.

---

## 부록 D. 참고 링크

**템플릿**
- Magic Portfolio: https://github.com/once-ui-system/magic-portfolio
- Magic Portfolio 문서: https://magic-portfolio.com/

**Next.js**
- Docker 배포: https://nextjs.org/docs/app/building-your-application/deploying#docker-image

**GCP**
- 외부 애플리케이션 로드밸런서: https://cloud.google.com/load-balancing/docs/https
- Google 관리형 SSL 인증서: https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs
- Cloud DNS 빠른 시작: https://cloud.google.com/dns/docs/quickstart
- IAP TCP 전달: https://cloud.google.com/iap/docs/using-tcp-forwarding
- 가격 계산기: https://cloud.google.com/products/calculator

**GitHub Actions**
- google-github-actions/auth: https://github.com/google-github-actions/auth
- Workload Identity Federation 가이드: https://github.com/google-github-actions/auth#preferred-direct-workload-identity-federation
