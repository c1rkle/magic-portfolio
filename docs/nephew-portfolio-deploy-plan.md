# 포트폴리오 사이트 구축·배포 전체 가이드 (Windows 기준)

> Magic Portfolio(Next.js) → GitHub Actions → GCP(GCE + HTTPS LB + Cloud DNS) 배포
> 작성일: 2026-08-09 / 대상: 인턴 조카 취업용 포트폴리오 사이트
> 기준 소스: https://github.com/once-ui-system/magic-portfolio
> **전제: 계정도 개발 환경도 아무것도 없는 상태에서 시작**

---

## 이 문서 사용법

- 명령어는 전부 **Windows PowerShell** 기준이야. `>` 표시는 빼고 명령만 복사해서 붙여넣으면 돼.
- 각 장 끝의 **[확인]** 블록을 통과해야 다음 장으로 넘어가. 통과 못 한 채로 진행하면 뒤에서 원인 찾기가 훨씬 어려워져.
- ⚠️ 표시는 실제로 자주 사고 나는 지점이야. 건너뛰지 말 것.

**전체 소요 시간**: 사전 준비(1~3장) 약 2시간, 나머지 하루. 여유 있게 이틀 잡으면 돼.

---

## 0. 최종 목표 아키텍처

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

**핵심 흐름**: 코드 수정 → `main`에 푸시 → GHA가 이미지 빌드·푸시 → VM에서 컨테이너 교체 → 사이트 반영 (2~4분)

---

## 1. 시작 전 반드시 알아둘 3가지

### 1-1. 라이선스 (중요)

Magic Portfolio는 **CC BY-NC 4.0** 라이선스야.

- **비상업적 사용만 허용** → 개인 취업용 포트폴리오는 해당됨. 문제 없음.
- **출처 표기(Attribution) 필수** → 푸터의 Once UI 크레딧을 **지우면 안 됨**.
- 상업적 이용(외주 제작, 회사 홍보 등)을 하려면 Once UI Pro 라이선스를 사서 확장해야 해.
- 이력서에는 "Once UI Magic Portfolio 템플릿 기반으로 커스터마이징 및 GCP 인프라 구축"이라고 정직하게 쓰는 게 좋아. 면접에서 오히려 인프라 역량으로 어필됨.

### 1-2. 무료 크레딧의 함정 ⚠️

GCP 무료 체험 크레딧 $300은 **90일 만료**야. 금액이 남아도 90일이 지나면 소멸돼.

| 항목 | 사양 | 월 예상 비용(서울 리전) |
|---|---|---|
| GCE VM | e2-small (2GB) | 약 $17 |
| 부팅 디스크 | pd-balanced 20GB | 약 $2.4 |
| Global HTTPS LB | 전달 규칙 1개 + 소량 트래픽 | 약 $19 |
| Cloud DNS | 존 1개 | 약 $0.3 |
| Artifact Registry | 이미지 수 GB | 약 $0.5 |
| **합계** | | **월 약 $39** |

- 크레딧으로는 **약 3개월** 무료 운영 (금액보다 90일 만료가 먼저 도달)
- 2026년 11월경 만료 시점에 반드시 결정이 필요해. 선택지는 §15-2 참고.
- 프로젝트 생성 직후 **예산 알림**부터 걸어둘 것 (§5-5). 이게 유일한 안전장치야.

### 1-3. 계정·비용 명의

- GCP 프로젝트, 결제 계정, GitHub 리포, 도메인 **전부 조카 본인 명의**로 만들 것. 나중에 인수인계할 게 없어져.
- 카드는 무료 체험 등록에 필요하지만 크레딧 소진 전까진 실제 청구가 없어. **"유료 계정으로 업그레이드"는 누르지 말 것.**

---

## 2. 사전 준비 (1) — 계정 만들기

여기서 만들 계정은 4개야: Google, GitHub, 고대디, 그리고 결제 수단 준비.

### 2-1. Google 계정

이미 개인 Gmail이 있으면 그걸 써도 돼. 다만 **취업용으로 쓸 계정이니 주소가 단정한지** 확인할 것. (`xxgamer1234@gmail.com` 같은 주소면 새로 만드는 게 나아)

없으면:

1. https://accounts.google.com/signup 접속
2. 이름 → 아이디 → 비밀번호 입력
3. 복구용 전화번호·이메일 등록 (**계정 잠겼을 때 유일한 복구 수단이니 꼭 등록**)
4. 약관 동의 → 완료

### 2-2. Google 계정 2단계 인증(2SV) 설정 ⚠️ 필수

**2026년 10월 20일부터 GCP 콘솔 로그인에 2단계 인증이 의무화돼.** 나중에 갑자기 막히면 곤란하니 지금 켜두는 게 맞아.

1. https://myaccount.google.com/security 접속
2. "Google에 로그인" → **2단계 인증** 클릭
3. "시작하기" → 비밀번호 재입력
4. 휴대전화 번호 입력 → SMS로 코드 수신 → 입력 → **사용 설정**
5. 다시 2단계 인증 화면으로 들어가서 아래로 스크롤 → **백업 코드** → "백업 코드 받기"
6. 10개 코드가 나오면 **다운로드하거나 인쇄해서 안전한 곳에 보관**

> 휴대폰을 잃어버리거나 기기를 바꾸면 백업 코드가 유일한 진입 수단이야. 이건 진짜로 저장해둘 것.

### 2-3. GitHub 계정

1. https://github.com/signup 접속
2. 이메일 → 비밀번호 → **사용자 이름(username)** 입력

   ⚠️ **username은 신중하게.** 이력서에 `github.com/사용자이름`으로 적히고, 나중에 바꾸면 기존 링크가 다 깨져. 본인 이름 기반(`minsu-kim`, `kimminsu-dev` 등)을 권장.
3. 이메일로 온 인증 코드 입력
4. 설문은 건너뛰어도 됨 → Free 플랜 선택

### 2-4. GitHub 2단계 인증 ⚠️ 필수

GitHub는 2023년부터 코드를 올리는 계정에 2FA를 의무화했어. 안 켜면 어느 시점에 강제로 막혀.

1. https://github.com/settings/security 접속
2. **Two-factor authentication** → "Enable two-factor authentication"
3. 방식 선택:
   - **인증 앱(권장)**: 휴대폰에 Google Authenticator 또는 Microsoft Authenticator 설치 → QR 스캔 → 6자리 코드 입력
   - SMS: 인증 앱을 못 쓰면 차선책
4. **복구 코드(recovery codes)가 나오면 반드시 다운로드해서 보관**

### 2-5. 프로필 정리 (취업용이니까)

https://github.com/settings/profile 에서:

- 프로필 사진 등록 (기본 아이콘이면 성의 없어 보임)
- 이름(본명), 한 줄 소개, 위치 입력
- 나중에 포트폴리오 도메인이 생기면 **Website 칸에 등록**

### 2-6. 고대디 계정

1. https://www.godaddy.com 접속 → 우측 상단 로그인 → "계정 만들기"
2. 이메일·비밀번호 입력 (**GCP와 같은 이메일 쓰면 관리 편함**)
3. 이메일 인증 완료

### 2-7. 결제 수단 확인

- **해외 결제가 가능한** 신용카드 또는 체크카드가 필요해 (GCP·고대디 둘 다 해외 결제)
- 체크카드도 되지만 잔액이 있어야 함. GCP는 등록 시 **약 $1 승인 테스트**가 잡혔다가 취소돼
- 카드사 앱에서 **해외 결제 차단이 걸려 있지 않은지** 미리 확인할 것 (은근히 여기서 막혀)

### [확인] 2장 완료 조건

- [ ] Google 계정 로그인됨 + 2단계 인증 켜짐 + 백업 코드 저장함
- [ ] GitHub 계정 로그인됨 + 2FA 켜짐 + 복구 코드 저장함
- [ ] 고대디 계정 이메일 인증 완료
- [ ] 해외 결제 가능한 카드 준비됨

---

## 3. 사전 준비 (2) — Windows 개발 환경 구축

설치할 것: Windows Terminal → Git → Node.js → VS Code → WSL2 → Docker Desktop → Google Cloud SDK

⚠️ **각 프로그램 설치 후에는 터미널을 반드시 닫았다가 새로 열어야 해.** PATH 환경변수가 갱신되지 않아서 "명령을 찾을 수 없습니다"가 뜨는 게 이 단계 사고의 90%야.

### 3-1. PowerShell 열기

`Win + X` → **터미널(관리자)** 또는 **Windows PowerShell(관리자)** 선택

이후 이 문서에서:
- **[관리자]** 표시가 있으면 관리자 권한 PowerShell에서 실행
- 표시가 없으면 일반 PowerShell에서 실행

### 3-2. winget 확인

Windows 10 1809 이상이면 기본 설치돼 있어.

```powershell
winget --version
```

버전이 안 나오면 Microsoft Store에서 **"앱 설치 관리자"**를 설치하면 돼.

### 3-3. Windows Terminal 설치 (선택이지만 권장)

```powershell
winget install --id Microsoft.WindowsTerminal -e
```

기본 PowerShell 창보다 복사·붙여넣기와 탭 관리가 훨씬 편해.

### 3-4. Git 설치

```powershell
winget install --id Git.Git -e
```

설치 후 **터미널을 닫고 새로 열어서** 확인:

```powershell
git --version
# git version 2.4x.x  형태로 나오면 성공
```

### 3-5. Git 최초 설정 ⚠️ 개행 설정 주의

```powershell
git config --global user.name "홍길동"
git config --global user.email "조카의github이메일@example.com"
git config --global init.defaultBranch main
```

⚠️ **여기가 Windows 특유의 함정이야.** Windows Git은 기본적으로 파일 개행을 CRLF로 바꿔서 저장해. 그런데 리눅스 서버에서 실행될 셸 스크립트(`.sh`)나 Dockerfile에 CRLF가 들어가면 **`$'\r': command not found` 같은 에러로 배포가 통째로 실패해.**

그래서 이렇게 설정해:

```powershell
git config --global core.autocrlf input
```

추가로 프로젝트에 `.gitattributes`도 넣을 건데, 그건 §7-4에서 다뤄.

설정 확인:

```powershell
git config --global --list
```

### 3-6. Node.js 설치

```powershell
winget install --id OpenJS.NodeJS.LTS -e
```

터미널 새로 열고 확인:

```powershell
node -v     # v22.x.x 형태
npm -v      # 10.x.x 형태
```

> Magic Portfolio의 최소 요구는 Node 18.17+ 인데, Node 20은 2026년 4월에 지원이 끝났어. **LTS(22.x)** 로 가면 돼.

### 3-7. VS Code 설치

```powershell
winget install --id Microsoft.VisualStudioCode -e
```

설치 후 유용한 확장(VS Code 좌측 확장 아이콘에서 검색):
- **MDX** — 포트폴리오 콘텐츠 파일 편집용
- **Docker**
- **ESLint**

### 3-8. WSL2 설치 (Docker Desktop 전제 조건)

**[관리자]** PowerShell에서:

```powershell
wsl --install
```

⚠️ 설치 후 **반드시 재부팅**해야 해. 재부팅하면 Ubuntu 초기 설정 창이 뜨는데, 사용자명·비밀번호를 만들어두면 돼 (이 계정은 이번 작업에서 직접 쓰진 않아).

재부팅 후 확인:

```powershell
wsl --status
# 기본 버전: 2  로 나오면 성공
```

> 안 될 경우: BIOS에서 가상화(Virtualization / Intel VT-x / AMD-V)가 꺼져 있을 수 있어. 노트북 제조사별로 BIOS 진입키가 달라(보통 F2/F10/Del). "가상화" 항목을 Enabled로 바꾸면 돼.

### 3-9. Docker Desktop 설치

```powershell
winget install --id Docker.DockerDesktop -e
```

설치 후:
1. **재부팅**
2. 시작 메뉴에서 **Docker Desktop** 실행
3. 약관 동의 → 로그인은 건너뛰어도 됨(Skip)
4. 우측 하단 트레이 아이콘의 고래가 **멈춰 있으면 준비 완료**(움직이면 아직 시작 중)

확인:

```powershell
docker --version
docker run --rm hello-world
# "Hello from Docker!" 가 나오면 성공
```

⚠️ **Docker Desktop은 실행 중이어야 `docker` 명령이 동작해.** PC를 껐다 켠 뒤에는 Docker Desktop을 먼저 실행하고 작업해.

### 3-10. Google Cloud SDK(gcloud CLI) 설치

```powershell
winget install --id Google.CloudSDK -e
```

터미널 새로 열고 확인:

```powershell
gcloud --version
```

⚠️ winget 설치가 실패하거나 명령을 못 찾으면 설치 파일로 하면 돼:
https://dl.google.com/dl/cloudsdk/channels/rapid/GoogleCloudSDKInstaller.exe
→ 실행 → "Bundled Python" 체크 유지 → 설치 → 터미널 재시작

### 3-11. PowerShell 스크립트 실행 정책

이후에 `.ps1` 스크립트를 하나 쓸 건데, Windows 기본 설정에서는 차단돼. 한 번만 풀어주면 돼.

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

`Y` 입력 후 엔터.

### 3-12. 작업 폴더 만들기

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\projects"
Set-Location "$HOME\projects"
```

이후 모든 작업은 이 폴더에서 해.

### [확인] 3장 완료 조건

아래를 한 번에 실행해서 전부 버전이 나오면 통과야:

```powershell
Write-Host "=== 설치 확인 ===" -ForegroundColor Cyan
git --version
node -v
npm -v
docker --version
gcloud --version
wsl --status
```

- [ ] git, node, npm, docker, gcloud 전부 버전 출력됨
- [ ] `docker run --rm hello-world` 성공
- [ ] `git config --global core.autocrlf` 가 `input` 으로 나옴

---

## 4. 도메인 구매 (고대디)

### 4-1. 도메인명 정하기

취업용이니까 **본인 이름 기반**이 제일 무난해.

- 추천: `이름성.com`, `이름-dev.com`, `이름.dev`
- 예: `kimminsu.com`, `minsu-kim.dev`, `minsudev.com`
- 피할 것: 하이픈 2개 이상, 숫자 혼용, 지나치게 긴 이름 (전화로 불러주기 어려운 도메인은 실패작)

TLD 참고:

| TLD | 연 비용 | 비고 |
|---|---|---|
| `.com` | 2만원 내외 | 가장 무난, 신뢰감 |
| `.dev` | 2만원 내외 | 개발자 이미지. **HTTPS 강제(HSTS preload)** 라 이 구조와 잘 맞음 |
| `.io`, `.me` | 4~6만원 | 갱신비가 비쌈. 비추 |

### 4-2. 구매 절차

1. https://www.godaddy.com 로그인
2. 검색창에 원하는 도메인 입력 → 사용 가능하면 장바구니 담기
3. ⚠️ **부가 옵션은 전부 해제할 것**:
   - 웹 호스팅 → 불필요 (GCP에 올릴 거야)
   - 이메일 → 불필요
   - SSL 인증서 → **불필요** (GCP에서 무료로 자동 발급)
   - 개인정보 보호 → 무료 기본 제공이면 유지, 유료면 판단
4. 결제 기간은 **1년**으로 (여러 해 결제는 나중에 판단)
5. 결제 완료
6. ⚠️ **고대디에서 오는 "도메인 소유권 확인" 메일의 링크를 반드시 클릭할 것.** 미인증 상태로 15일 지나면 도메인이 정지돼.

### 4-3. 구매 정보 메모

```
도메인명: ______________________
구매일:   ______________________
만료일:   ______________________
자동갱신: 켬 / 끔        ← 켜두는 걸 권장
```

### [확인] 4장 완료 조건

- [ ] 고대디 "내 제품"에 도메인이 보임
- [ ] 소유권 확인 메일 인증 완료 (도메인에 경고 표시 없음)

---

## 5. GCP 프로젝트 생성

### 5-1. 무료 체험 등록

1. 조카 Google 계정으로 https://console.cloud.google.com 접속
2. "무료로 시작하기" 클릭
3. 1단계: 국가 **대한민국**, 약관 동의
4. 2단계: 계정 유형 **개인**, 이름·주소·카드 정보 입력
5. 완료되면 콘솔 상단에 **$300 크레딧과 남은 일수**가 표시돼. 만료일을 메모해둘 것

```
크레딧 만료일: ______________________   ← 캘린더에 알림 걸어둘 것
```

### 5-2. 프로젝트 생성

콘솔 상단 프로젝트 선택 드롭다운 → **새 프로젝트**

```
프로젝트 이름: portfolio
프로젝트 ID:   portfolio-XXXXXX   ← 전역에서 유일해야 함. 자동 생성값 그대로 써도 됨
```

⚠️ **프로젝트 ID는 나중에 못 바꿔.** 화면에 나온 ID를 정확히 메모해둘 것.

### 5-3. gcloud 로그인

```powershell
gcloud auth login
```

브라우저가 열리면 조카 Google 계정으로 로그인 → 권한 허용.

```powershell
gcloud config set project portfolio-XXXXXX
gcloud config list
```

### 5-4. 작업 변수 파일 만들기 ⚠️ 유용함

PowerShell 변수는 창을 닫으면 사라져. 매번 다시 입력하지 않도록 파일로 만들어두자.

`$HOME\projects\set-vars.ps1` 파일을 VS Code로 만들고 아래 내용을 넣어:

```powershell
$PROJECT_ID = "portfolio-XXXXXX"        # 본인 프로젝트 ID로 교체
$REGION     = "asia-northeast3"          # 서울
$ZONE       = "asia-northeast3-a"
$DOMAIN     = "example.com"              # 구매한 도메인으로 교체
$REPO       = "portfolio-docker"
$REGISTRY   = "asia-northeast3-docker.pkg.dev"
$SA_NAME    = "github-actions-deployer"
$SA_EMAIL   = "$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"

Write-Host "변수 로드 완료: $PROJECT_ID / $DOMAIN" -ForegroundColor Green
```

**새 터미널을 열 때마다 아래를 먼저 실행**하면 돼 (앞의 점과 공백 주의):

```powershell
cd $HOME\projects
. .\set-vars.ps1
```

### 5-5. 필요한 API 활성화

```powershell
gcloud services enable compute.googleapis.com artifactregistry.googleapis.com dns.googleapis.com iap.googleapis.com iamcredentials.googleapis.com cloudresourcemanager.googleapis.com
```

1~2분 걸려. 완료 확인:

```powershell
gcloud services list --enabled --filter="name:(compute OR artifactregistry OR dns OR iap)"
```

### 5-6. 예산 알림 설정 ⚠️ 건너뛰지 말 것

콘솔 → 좌측 메뉴 **결제** → **예산 및 알림** → **예산 만들기**

```
이름:   portfolio-budget
범위:   프로젝트 = portfolio
금액:   월 $50
알림:   실제 지출 50% / 80% / 100% → 이메일 수신
```

크레딧 만료 후 방치되는 사고를 막는 유일한 안전장치야.

### [확인] 5장 완료 조건

- [ ] `gcloud config list`에 올바른 프로젝트 ID가 나옴
- [ ] API 6개 활성화 완료
- [ ] 예산 알림 생성됨
- [ ] `set-vars.ps1` 작성 완료, `. .\set-vars.ps1` 실행 시 초록색 메시지 출력

---

## 6. 소스 준비 및 GitHub 리포지토리

### 6-1. 원본 받아서 새 리포로 시작

**포크가 아니라 새 리포로 시작하는 걸 권장해.** 포크로 두면 GitHub 프로필에 "forked from"이 붙어서 이력서용으로 약해 보이고, 커밋 히스토리도 원본과 섞여.

```powershell
cd $HOME\projects
git clone --depth 1 https://github.com/once-ui-system/magic-portfolio.git portfolio
cd portfolio
Remove-Item -Recurse -Force .git
```

### 6-2. 로컬 실행부터 확인 ⚠️

인프라보다 먼저 화면이 뜨는지 봐야 해.

```powershell
npm install
npm run dev
```

브라우저에서 http://localhost:3000 접속 → 포트폴리오 화면이 나오면 성공.

확인했으면 터미널에서 `Ctrl + C`로 종료.

> ⚠️ 여기서 안 뜨면 이후 단계는 전부 의미 없어. 반드시 통과하고 넘어갈 것.
> `npm install`에서 에러가 나면 Node 버전(`node -v`)이 22인지부터 확인해.

### 6-3. 새 Git 리포로 초기화

```powershell
git init
git add .
git commit -m "chore: initialize portfolio from Magic Portfolio template"
```

### 6-4. GitHub에 리포 생성

1. https://github.com/new 접속
2. Repository name: `portfolio`
3. **Public** 선택 (이력서에 리포 주소도 같이 적을 수 있어서 권장)
4. ⚠️ **"Add a README file", ".gitignore", "license" 는 전부 체크 해제** (이미 파일이 있어서 충돌 남)
5. **Create repository**

### 6-5. 푸시

```powershell
git branch -M main
git remote add origin https://github.com/조카아이디/portfolio.git
git push -u origin main
```

첫 푸시 때 **Git Credential Manager 창이 뜨면서 브라우저 로그인**을 요구해. GitHub 계정으로 로그인하면 이후로는 자동으로 인증돼.

### 6-6. README에 출처 명시 (라이선스 의무)

`README.md` 최상단에 추가:

```markdown
Based on [Magic Portfolio](https://github.com/once-ui-system/magic-portfolio) by Once UI (CC BY-NC 4.0).
```

### [확인] 6장 완료 조건

- [ ] `npm run dev`로 localhost:3000 정상 표시
- [ ] GitHub 리포에 코드가 올라감
- [ ] README에 출처 표기됨

---

## 7. 콘텐츠 커스터마이징

> 파일 경로는 템플릿 버전에 따라 조금씩 달라. VS Code에서 `src/resources/` 폴더를 먼저 열어보고 실제 구조에 맞춰 작업할 것.

```powershell
code .    # VS Code로 프로젝트 열기
```

### 7-1. 수정 대상

| 대상 | 대략적 위치 | 내용 |
|---|---|---|
| 개인 정보 | `src/resources/content.tsx` | 이름, 직함, 한 줄 소개, 이메일, 소셜 링크 |
| 사이트 설정 | `src/resources/once-ui.config.ts` | **`baseURL`을 구매한 도메인으로 변경**, 테마·컬러 |
| 소개/경력 | `content.tsx`의 about 섹션 | 학력, 경력(써클 인턴 포함), 기술 스택 |
| 프로젝트 | `src/app/work/projects/*.mdx` | 프로젝트별 MDX 파일 (샘플 삭제 후 본인 것으로) |
| 블로그 | `src/app/blog/posts/*.mdx` | 선택. 안 쓸 거면 메뉴에서 숨김 |
| 이미지 | `public/images/` | 프로필 사진, 프로젝트 스크린샷, OG 이미지 |

### 7-2. `baseURL`은 반드시 먼저 바꿀 것 ⚠️

메타태그, OG 이미지, sitemap, robots.txt가 전부 이 값을 기준으로 생성돼. 기본값이 남아 있으면 검색 노출과 카톡 링크 미리보기가 깨져.

```ts
baseURL: "https://구매한도메인.com",
```

### 7-3. 수정하면서 실시간 확인

```powershell
npm run dev
```

켜둔 채로 파일을 저장하면 브라우저가 자동 갱신돼. 콘텐츠 작업 내내 켜두면 편해.

### 7-4. `.gitattributes` 추가 ⚠️ Windows 필수

리눅스 서버에서 실행될 파일들의 개행이 깨지지 않도록 프로젝트 루트에 `.gitattributes` 파일을 만들어:

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

이걸 안 넣으면 §11의 시작 스크립트가 VM에서 실행되다가 깨질 수 있어.

### 7-5. 이력서 관점 작성 팁

- **경력에 써클 인턴 경험을 구체적으로**: 기간(2026-XX ~ 2026-09-04), 담당 업무, 사용 기술을 성과 중심으로. ⚠️ **재직 중에 정리해두는 게 정확해.** 나가고 나면 세부 내용이 흐려져.
- **프로젝트는 3~5개**가 적정. 개수보다 각각의 "문제 → 해결 → 결과"가 명확한 게 중요해.
- **이 포트폴리오 사이트 자체를 프로젝트로 등록할 것.** Next.js + Docker + GitHub Actions + GCP(LB/GCE/Cloud DNS) 구성도를 넣으면 신입 기준으로 강한 차별점이 돼. §0의 아키텍처 그림을 그대로 써도 좋아.
- 연락 가능한 이메일과 GitHub 링크는 첫 화면에서 바로 보이게.

### 7-6. 커밋

```powershell
git add .
git commit -m "feat: customize portfolio content"
git push
```

### [확인] 7장 완료 조건

- [ ] 이름·소개·연락처가 본인 것으로 교체됨
- [ ] `baseURL`이 구매한 도메인으로 변경됨
- [ ] 샘플 프로젝트/블로그 글이 정리됨
- [ ] `.gitattributes` 추가됨

---

## 8. Docker 이미지화

### 8-1. `next.config.mjs`에 standalone 출력 추가

```js
const nextConfig = {
  output: "standalone",   // ← 이 줄 추가
  // ... 기존 설정 유지
};
```

이게 있어야 런타임 이미지가 1GB대 → 200MB대로 줄어. e2-small에서 돌리려면 사실상 필수야.

### 8-2. `Dockerfile` 생성 (프로젝트 루트)

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

> 원본 템플릿은 npm 기준이야. **npm으로 통일하는 걸 권장**해. (`package-lock.json`이 리포에 커밋돼 있어야 `npm ci`가 동작해)

### 8-3. `.dockerignore` 생성

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
```

### 8-4. 로컬에서 컨테이너 검증 ⚠️

Docker Desktop이 실행 중인지 확인하고:

```powershell
docker build -t portfolio:local .
docker run --rm -p 3000:3000 portfolio:local
```

http://localhost:3000 접속해서 화면이 뜨면 성공. `Ctrl + C`로 종료.

> ⚠️ **여기까지 통과해야 다음으로 넘어갈 것.** 로컬 컨테이너에서 안 뜨는 이미지는 GCE에서도 안 떠. 여기서 잡는 게 10배 빨라.

### 8-5. 커밋

```powershell
git add .
git commit -m "feat: add Dockerfile and standalone output for GCE deployment"
git push
```

### [확인] 8장 완료 조건

- [ ] `docker build` 성공
- [ ] `docker run`으로 localhost:3000 정상 표시
- [ ] GitHub에 푸시됨

---

## 9. Artifact Registry 생성

빌드한 이미지를 보관할 저장소야.

```powershell
. .\set-vars.ps1   # 새 터미널이면 먼저 실행

gcloud artifacts repositories create $REPO --repository-format=docker --location=$REGION --description="Portfolio container images"
```

확인:

```powershell
gcloud artifacts repositories list --location=$REGION
```

이미지 경로 형식:

```
asia-northeast3-docker.pkg.dev/<PROJECT_ID>/portfolio-docker/portfolio:<TAG>
```

---

## 10. GitHub Actions용 서비스 계정

GitHub Actions가 GCP에 접근하려면 "로봇 계정"이 필요해.

### 10-1. 서비스 계정 생성 및 권한 부여

```powershell
gcloud iam service-accounts create $SA_NAME --display-name="GitHub Actions Deployer"
```

권한 5개를 순서대로 부여해:

```powershell
# 이미지 푸시
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_EMAIL" --role="roles/artifactregistry.writer"

# VM 조회
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_EMAIL" --role="roles/compute.viewer"

# SSH 접속 (OS Login)
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_EMAIL" --role="roles/compute.osAdminLogin"

# IAP 터널을 통한 SSH
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_EMAIL" --role="roles/iap.tunnelResourceAccessor"

# VM의 서비스 계정 대행 (SSH 시 필요)
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_EMAIL" --role="roles/iam.serviceAccountUser"
```

### 10-2. 키 발급

```powershell
gcloud iam service-accounts keys create "$HOME\Desktop\gha-key.json" --iam-account=$SA_EMAIL
```

클립보드로 복사:

```powershell
Get-Content "$HOME\Desktop\gha-key.json" -Raw | Set-Clipboard
```

### 10-3. GitHub Secret 등록

GitHub 리포 → **Settings** → 좌측 **Secrets and variables** → **Actions** → **New repository secret**

| Secret 이름 | 값 |
|---|---|
| `GCP_SA_KEY` | 방금 복사한 JSON 전체 (붙여넣기) |
| `GCP_PROJECT_ID` | 프로젝트 ID |

### 10-4. 로컬 키 파일 즉시 삭제 ⚠️

```powershell
Remove-Item "$HOME\Desktop\gha-key.json"
```

이 파일은 GCP 계정에 대한 열쇠야. 바탕화면에 남겨두면 안 돼.

> 더 안전한 방식은 키 없는 Workload Identity Federation이야. 이번엔 단순화를 위해 키 방식으로 가고, 여유 있을 때 §부록 B로 전환하면 돼.

### [확인] 10장 완료 조건

- [ ] 서비스 계정 생성 + 권한 5개 부여
- [ ] GitHub Secret 2개 등록
- [ ] 로컬 키 파일 삭제 완료

---

## 11. GCE 인프라 구축

### 11-1. 시작 스크립트 파일 생성 ⚠️ 개행 주의

VM이 부팅될 때 Docker를 자동 설치하는 스크립트야. **이 파일은 반드시 LF 개행이어야 해.** PowerShell로 안전하게 만들자.

```powershell
cd $HOME\projects\portfolio
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

Write-Host "생성 완료" -ForegroundColor Green
```

> `@' ... '@` 는 PowerShell의 문자열 블록이야. 안의 `$` 기호가 변수로 해석되지 않도록 작은따옴표 버전을 썼어. 이걸 큰따옴표(`@" "@`)로 바꾸면 스크립트가 망가지니 그대로 쓸 것.

### 11-2. VM 생성

```powershell
. .\set-vars.ps1

gcloud compute instances create portfolio-vm --zone=$ZONE --machine-type=e2-small --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud --boot-disk-size=20GB --boot-disk-type=pd-balanced --tags=portfolio-web --scopes=https://www.googleapis.com/auth/cloud-platform --metadata=enable-oslogin=TRUE --metadata-from-file=startup-script=infra/startup-script.sh
```

### 11-3. VM에 이미지 읽기 권한 부여

```powershell
$PROJECT_NUM = gcloud projects describe $PROJECT_ID --format="value(projectNumber)"

gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$PROJECT_NUM-compute@developer.gserviceaccount.com" --role="roles/artifactregistry.reader"
```

### 11-4. 방화벽 규칙

```powershell
# LB 헬스체크 + 트래픽 유입 (Google 전용 대역)
gcloud compute firewall-rules create allow-lb-to-portfolio --allow=tcp:3000 --source-ranges=130.211.0.0/22,35.191.0.0/16 --target-tags=portfolio-web --description="Allow GCLB health check and traffic"

# IAP를 통한 SSH
gcloud compute firewall-rules create allow-iap-ssh --allow=tcp:22 --source-ranges=35.235.240.0/20 --target-tags=portfolio-web --description="Allow SSH via IAP tunnel"
```

> ⚠️ 22번 포트를 `0.0.0.0/0`으로 열지 말 것. 몇 분 안에 전 세계에서 무차별 로그인 시도가 들어와. IAP 터널로만 접속하는 지금 구조가 훨씬 안전해.

### 11-5. VM 접속 테스트

시작 스크립트 완료까지 2~3분 걸려. 조금 기다린 뒤:

```powershell
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap --command="docker --version"
```

⚠️ **첫 실행 시 SSH 키 생성 안내가 나오면서 passphrase를 물어봐.** 그냥 **엔터 두 번** 치면 돼(비밀번호 없이 생성).

`Docker version 2x.x.x` 가 나오면 성공.

안 나오면 시작 스크립트가 아직 안 끝났거나 실패한 거야. 로그 확인:

```powershell
gcloud compute ssh portfolio-vm --zone=$ZONE --tunnel-through-iap --command="sudo journalctl -u google-startup-scripts --no-pager | tail -30"
```

### 11-6. 인스턴스 그룹 (LB 백엔드용)

```powershell
gcloud compute instance-groups unmanaged create portfolio-ig --zone=$ZONE

gcloud compute instance-groups unmanaged add-instances portfolio-ig --zone=$ZONE --instances=portfolio-vm

# LB가 3000 포트로 보내도록 named port 지정
gcloud compute instance-groups unmanaged set-named-ports portfolio-ig --zone=$ZONE --named-ports=http:3000
```

### [확인] 11장 완료 조건

- [ ] VM이 RUNNING 상태 (`gcloud compute instances list`)
- [ ] SSH 접속되고 `docker --version` 출력됨
- [ ] 방화벽 규칙 2개 생성됨
- [ ] 인스턴스 그룹에 VM이 등록되고 named port가 http:3000

---

## 12. HTTPS 로드밸런서 + 도메인 연결

이 장이 제일 길고, **순서가 중요해.** 특히 DNS를 먼저 연결해야 SSL 인증서가 발급돼.

### 12-1. 고정 글로벌 IP

```powershell
gcloud compute addresses create portfolio-ip --global

$LB_IP = gcloud compute addresses describe portfolio-ip --global --format="value(address)"
Write-Host "LB IP: $LB_IP" -ForegroundColor Yellow
```

```
LB IP 메모: ______________________
```

### 12-2. 헬스체크 & 백엔드 서비스

```powershell
gcloud compute health-checks create http portfolio-hc --port=3000 --request-path=/ --check-interval=15s --timeout=5s --healthy-threshold=2 --unhealthy-threshold=3

gcloud compute backend-services create portfolio-backend --protocol=HTTP --port-name=http --health-checks=portfolio-hc --global --enable-cdn --cache-mode=CACHE_ALL_STATIC

gcloud compute backend-services add-backend portfolio-backend --instance-group=portfolio-ig --instance-group-zone=$ZONE --global
```

> `--enable-cdn`은 정적 자산을 캐싱해서 응답을 빠르게 하고 VM 부하도 줄여줘. 콘텐츠 수정 후 반영이 늦어 보이면 캐시 무효화(§14-3)를 하면 돼.

### 12-3. Cloud DNS 존 생성

```powershell
gcloud dns managed-zones create portfolio-zone --dns-name="$DOMAIN." --description="Portfolio DNS zone"

gcloud dns managed-zones describe portfolio-zone --format="value(nameServers)"
```

출력되는 네임서버 4개를 메모해:

```
ns-cloud-XX.googledomains.com.
ns-cloud-XX.googledomains.com.
ns-cloud-XX.googledomains.com.
ns-cloud-XX.googledomains.com.
```

### 12-4. A 레코드 등록

```powershell
gcloud dns record-sets create "$DOMAIN." --zone=portfolio-zone --type=A --ttl=300 --rrdatas=$LB_IP

gcloud dns record-sets create "www.$DOMAIN." --zone=portfolio-zone --type=A --ttl=300 --rrdatas=$LB_IP
```

### 12-5. 고대디 네임서버 변경

1. https://www.godaddy.com 로그인 → **내 제품** → 도메인 → **DNS 관리**
2. 아래로 스크롤 → **네임서버** 섹션 → **변경**
3. **"내 고유 네임서버 사용"** 선택
4. §12-3에서 얻은 4개 주소 입력
   ⚠️ **끝의 점(`.`)은 빼고 입력**할 것. `ns-cloud-a1.googledomains.com` 형태로.
5. 저장 → 경고 문구가 뜨면 계속 진행

### 12-6. DNS 전파 확인 ⚠️ 여기서 기다려야 함

전파에 **보통 10분~1시간, 최대 48시간** 걸려.

```powershell
Resolve-DnsName $DOMAIN -Type NS
Resolve-DnsName $DOMAIN -Type A
```

- NS 조회에 `googledomains.com`이 나오면 네임서버 위임 성공
- A 조회에 §12-1의 LB IP가 나오면 성공

캐시 때문에 안 보이면 초기화 후 재시도:

```powershell
Clear-DnsClientCache
Resolve-DnsName $DOMAIN -Type A -Server 8.8.8.8
```

> ⚠️ **여기서 넘어가면 안 되는 지점이야.** DNS가 LB IP를 가리키기 전에는 다음 단계의 SSL 인증서가 **절대 발급되지 않아.** 반드시 확인하고 진행할 것.
> 기다리는 동안 §7(콘텐츠 작성)을 진행하면 시간을 아낄 수 있어.

### 12-7. 관리형 SSL 인증서 생성

```powershell
gcloud compute ssl-certificates create portfolio-cert --domains="$DOMAIN,www.$DOMAIN" --global
```

### 12-8. 프록시 및 전달 규칙

```powershell
# HTTPS 경로
gcloud compute url-maps create portfolio-urlmap --default-service=portfolio-backend

gcloud compute target-https-proxies create portfolio-https-proxy --url-map=portfolio-urlmap --ssl-certificates=portfolio-cert

gcloud compute forwarding-rules create portfolio-https-rule --global --target-https-proxy=portfolio-https-proxy --address=portfolio-ip --ports=443
```

HTTP → HTTPS 리다이렉트용 URL 맵은 YAML 파일이 필요해:

```powershell
$redirectYaml = @'
name: portfolio-redirect-urlmap
defaultUrlRedirect:
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
  httpsRedirect: true
'@

$lf = $redirectYaml -replace "`r`n", "`n"
[System.IO.File]::WriteAllText("$PWD\redirect-urlmap.yaml", $lf, (New-Object System.Text.UTF8Encoding $false))

gcloud compute url-maps import portfolio-redirect-urlmap --source=redirect-urlmap.yaml --global --quiet

gcloud compute target-http-proxies create portfolio-http-proxy --url-map=portfolio-redirect-urlmap

gcloud compute forwarding-rules create portfolio-http-rule --global --target-http-proxy=portfolio-http-proxy --address=portfolio-ip --ports=80
```

### 12-9. 인증서 발급 대기

```powershell
while ($true) {
  $status = gcloud compute ssl-certificates describe portfolio-cert --global --format="value(managed.status)"
  $domainStatus = gcloud compute ssl-certificates describe portfolio-cert --global --format="value(managed.domainStatus)"
  Write-Host "$(Get-Date -Format 'HH:mm:ss')  상태: $status  |  $domainStatus"
  if ($status -eq "ACTIVE") { Write-Host "인증서 발급 완료!" -ForegroundColor Green; break }
  Start-Sleep -Seconds 30
}
```

| 상태 | 의미 | 조치 |
|---|---|---|
| `PROVISIONING` | 발급 진행 중 | 대기 (**보통 15분~1시간, 최대 24시간**) |
| `ACTIVE` | 완료 | 다음 단계로 |
| `FAILED_NOT_VISIBLE` | DNS가 LB IP를 안 가리킴 | §12-5, §12-6 다시 확인 |

중단하려면 `Ctrl + C`.

### [확인] 12장 완료 조건

- [ ] `Resolve-DnsName $DOMAIN -Type A` 에 LB IP가 나옴
- [ ] SSL 인증서 상태가 `ACTIVE`
- [ ] 전달 규칙 2개(80, 443) 생성됨

---

## 13. GitHub Actions 배포 파이프라인

### 13-1. 워크플로우 파일 생성

프로젝트에 `.github/workflows/deploy.yml` 파일을 만들어:

```powershell
New-Item -ItemType Directory -Force -Path ".github\workflows"
```

VS Code로 `.github/workflows/deploy.yml` 생성 후 아래 내용 붙여넣기:

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

### 13-2. 커밋 & 첫 배포

```powershell
git add .
git commit -m "ci: add GitHub Actions deployment to GCE"
git push
```

GitHub 리포 → **Actions** 탭에서 진행 상황을 볼 수 있어. 첫 배포는 5~8분 걸려(이미지 빌드 때문).

### 13-3. 첫 실행에서 자주 나는 오류

| 증상 | 원인 | 해결 |
|---|---|---|
| `Permission denied on resource project` | 서비스 계정 권한 누락 | §10-1 권한 5개 다시 확인 |
| SSH 단계에서 타임아웃 | IAP 방화벽 규칙 없음 | §11-4의 `allow-iap-ssh` 확인 |
| `docker: not found` | VM 시작 스크립트 미완료/실패 | §11-5 로그 확인 명령 실행 |
| 이미지 pull 권한 오류 | VM 서비스 계정 권한 없음 | §11-3 다시 실행 |
| `$'\r': command not found` | 개행이 CRLF | §3-5, §7-4 설정 확인 후 파일 재생성 |
| `npm ci` 실패 | `package-lock.json` 미커밋 | `git add package-lock.json` 후 푸시 |

### [확인] 13장 완료 조건

- [ ] Actions 워크플로우가 초록색 체크로 끝남
- [ ] Verify 단계에서 `200 OK` 출력

---

## 14. 최종 검증

### 14-1. 인프라 상태 확인

```powershell
. .\set-vars.ps1

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

⚠️ PowerShell에서 `curl`은 다른 명령의 별칭이라 **반드시 `curl.exe`로 입력**해야 해.

### 14-2. 브라우저 체크리스트

- [ ] `https://도메인` 정상 표시, 주소창에 자물쇠 아이콘
- [ ] `http://도메인` → https로 자동 전환
- [ ] `www.도메인` 도 접속됨
- [ ] 휴대폰에서 접속 시 레이아웃 정상
- [ ] 카카오톡·슬랙에 링크 붙여넣기 → 썸네일 미리보기 정상 (`baseURL` 설정 확인)
- [ ] `https://도메인/sitemap.xml` 정상
- [ ] `https://도메인/robots.txt` 정상

### 14-3. 수정 반영이 안 될 때 (CDN 캐시)

```powershell
gcloud compute url-maps invalidate-cdn-cache portfolio-urlmap --path="/*" --global
```

### 14-4. 마무리 작업

- [ ] GitHub 프로필(https://github.com/settings/profile)의 **Website 칸에 도메인 등록**
- [ ] 이력서·구직 사이트 프로필에 도메인 기재
- [ ] Google Search Console(https://search.google.com/search-console)에 도메인 등록 → 검색 노출용 (선택)

---

## 15. 운영 및 크레딧 만료 대응

### 15-1. 일상 운영

**콘텐츠 수정 → 반영:**

```powershell
cd $HOME\projects\portfolio
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

### 15-2. 크레딧 만료(약 2026년 11월) 시 선택지 ⚠️

| 선택지 | 월 비용 | 작업량 | 비고 |
|---|---|---|---|
| **A. 그대로 유지** | 약 $39 | 없음 | 취업 확정까지 짧게 쓸 거면 무난 |
| **B. LB 제거 + VM에 Caddy** | 약 $10 | 반나절 | VM에 Caddy를 얹어 Let's Encrypt 자동 발급. DNS를 VM 고정 IP로 변경 |
| **C. Cloud Run 전환** | 약 $0~2 | 반나절 | 트래픽 없으면 0으로 스케일. **개인 포폴엔 사실상 최적** |
| **D. Vercel 무료 플랜** | $0 | 1시간 | Next.js 원저작 플랫폼. 다만 인프라 학습 요소가 사라짐 |

**권장**: 취업 활동이 끝날 때까지는 A로 두고, 장기 보관용으로 넘어갈 때 C(Cloud Run)로 전환. 이력서에 "GCE+LB로 구성 후 비용 최적화를 위해 Cloud Run으로 마이그레이션"이라고 쓸 수 있어서 오히려 스토리가 하나 더 생겨.

### 15-3. 전체 정리(삭제)

더 이상 안 쓸 때 과금을 끊는 순서야. **의존 관계가 있어서 이 순서를 지켜야 해.**

```powershell
. .\set-vars.ps1

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

가장 확실한 방법은 **프로젝트 자체 종료**야. 콘솔 → IAM 및 관리자 → 설정 → **종료**. 30일 후 완전 삭제되고 그 전엔 복구할 수 있어.

---

## 16. 전체 진행 순서 요약

| # | 단계 | 예상 시간 | 선행 조건 |
|---|---|---|---|
| 1 | 계정 4종 생성 + 2단계 인증 (§2) | 40분 | 휴대폰, 카드 |
| 2 | Windows 개발 환경 설치 (§3) | 60분 | **재부팅 2회 필요** |
| 3 | 도메인 구매 (§4) | 20분 | 고대디 계정, 카드 |
| 4 | GCP 프로젝트·API·예산알림 (§5) | 30분 | Google 계정 |
| 5 | 소스 준비 + 로컬 실행 확인 (§6) | 30분 | Node 22 |
| 6 | Dockerfile + 로컬 컨테이너 검증 (§8) | 40분 | Docker Desktop |
| 7 | Artifact Registry + 서비스계정 (§9~10) | 30분 | gcloud |
| 8 | VM + 방화벽 + 인스턴스 그룹 (§11) | 30분 | |
| 9 | 고정IP + LB + Cloud DNS (§12-1~12-4) | 30분 | |
| 10 | 고대디 네임서버 변경 + 전파 (§12-5~12-6) | 10분 + **대기 1h** | 9 완료 |
| 11 | SSL 인증서 (§12-7~12-9) | 5분 + **대기 15m~24h** | 10 완료 |
| 12 | 콘텐츠 커스터마이징 (§7) | **2~6시간** | 이력 내용 준비 |
| 13 | GHA 워크플로우 + 첫 배포 (§13) | 40분 | 7 완료 |
| 14 | 최종 검증 (§14) | 20분 | 11, 13 완료 |

### 효율적인 순서 ⚠️

**병목은 10~11번의 DNS 전파와 인증서 발급이야.** 이건 그냥 기다리는 시간이라 활용이 가능해.

```
1일차: §2 계정 → §3 환경설치 → §4 도메인 → §5 GCP → §6 소스 →
       §8 Docker → §9~11 인프라 → §12-1~12-6 LB+DNS까지 만들고 네임서버 변경
       ↓ (전파 대기하는 동안)
       §7 콘텐츠 작성 시작
2일차: §12-7~12-9 인증서 → §13 배포 → §14 검증 → §7 콘텐츠 마무리
```

---

## 부록 A. 인턴 종료(2026-09-04) 관련 메모

- 이 작업은 조카의 근무 종료(9/4) 전에 끝내두는 게 좋아. 이후엔 원격으로 봐주기가 번거로워져.
- **써클 인턴 경력 내용은 재직 중에 정리**해두는 게 정확해 (담당 업무, 사용 기술, 성과 수치).
- GCP 프로젝트·GitHub 리포·도메인 전부 조카 명의라 인수인계 이슈는 없어.

## 부록 B. Workload Identity Federation 전환 (선택)

서비스 계정 키(JSON)를 GitHub Secret에 두는 방식은 키가 유출되면 그대로 뚫려. 여유가 생기면 키 없는 방식으로 바꾸는 걸 권장해.

```powershell
gcloud iam workload-identity-pools create github-pool --location=global --display-name="GitHub Pool"

gcloud iam workload-identity-pools providers create-oidc github-provider --location=global --workload-identity-pool=github-pool --issuer-uri="https://token.actions.githubusercontent.com" --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" --attribute-condition="assertion.repository=='조카아이디/portfolio'"

$PROJECT_NUM = gcloud projects describe $PROJECT_ID --format="value(projectNumber)"

gcloud iam service-accounts add-iam-policy-binding $SA_EMAIL --role=roles/iam.workloadIdentityUser --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUM/locations/global/workloadIdentityPools/github-pool/attribute.repository/조카아이디/portfolio"
```

워크플로우에서 `credentials_json` 대신:

```yaml
permissions:
  contents: read
  id-token: write

# ...
      - uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: projects/<PROJECT_NUM>/locations/global/workloadIdentityPools/github-pool/providers/github-provider
          service_account: <SA_EMAIL>
```

전환이 확인되면 기존 키 삭제:

```powershell
gcloud iam service-accounts keys list --iam-account=$SA_EMAIL
gcloud iam service-accounts keys delete <KEY_ID> --iam-account=$SA_EMAIL
```

## 부록 C. Windows 특화 문제 모음

| 증상 | 원인 | 해결 |
|---|---|---|
| `'gcloud'은(는) 내부 또는 외부 명령... 이 아닙니다` | PATH 미갱신 | 터미널 닫고 새로 열기. 그래도 안 되면 PC 재부팅 |
| `docker: error during connect` | Docker Desktop 미실행 | 시작 메뉴에서 Docker Desktop 실행 후 고래 아이콘 멈출 때까지 대기 |
| `이 시스템에서 스크립트를 실행할 수 없으므로` | 실행 정책 | §3-11 명령 실행 |
| `$'\r': command not found` (배포 시) | CRLF 개행 | §3-5 `core.autocrlf input` + §7-4 `.gitattributes` 후 파일 재생성·재커밋 |
| `curl -I`가 이상한 결과 | PowerShell 별칭 | `curl.exe -I` 로 입력 |
| WSL 설치 실패 | BIOS 가상화 비활성 | BIOS에서 Virtualization / VT-x / SVM 활성화 |
| `npm install` 권한 오류 | 백신 프로그램 간섭 | 프로젝트 폴더를 백신 실시간 검사 예외로 추가 |
| 한글 깨짐 | 인코딩 | `chcp 65001` 실행 후 재시도 |

## 부록 D. 참고 링크

- Magic Portfolio: https://github.com/once-ui-system/magic-portfolio
- Magic Portfolio 문서: https://magic-portfolio.com/
- Next.js Docker 배포: https://nextjs.org/docs/app/building-your-application/deploying#docker-image
- GCP 외부 애플리케이션 LB: https://cloud.google.com/load-balancing/docs/https
- Google 관리형 SSL 인증서: https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs
- Cloud DNS 빠른 시작: https://cloud.google.com/dns/docs/quickstart
- google-github-actions/auth: https://github.com/google-github-actions/auth

---

# 진행 순서

> 위 본문(§0~§16)은 각 단계의 상세 설명이고, 이 절은 실제로 진행할 순서만 정리한 것이야.
> 최종적으로는 이 「진행 순서」만 남기고 위 본문은 삭제할 예정이라, 각 항목은 본문 없이도 그대로 따라 할 수 있게 작성한다.

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
  ├─ portfolio\        ← 포트폴리오 소스코드 (5번에서 생성)
  └─ set-vars.ps1      ← GCP 작업용 변수 파일 (6번에서 생성)
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

> `Remove-Item -Recurse -Force .git` 은 원본 템플릿의 깃 히스토리를 지우는 것이다. 5번에서 조카 계정의 새 리포지토리로 다시 초기화한다.
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

## 2. 조카 Gmail 계정으로 GitHub 계정 만들기

조카의 Gmail 계정으로 GitHub 계정을 생성한다.

**이미 GitHub 계정이 있는 경우** — 그대로 사용해도 되지만, 아래를 권고한다.

- **취업 준비용 Gmail 계정을 새로 하나 만들 것.** 취업과 관련된 모든 사항(GitHub, GCP, 도메인, 채용 사이트, 기업 지원 메일)을 이 하나의 계정으로 통일하기 위해서야.
- **그 Gmail 계정으로 GitHub 계정까지 새로 만들 것.** 기존 계정을 이메일만 바꿔 쓰는 것보다, 취업용 계정을 분리해두는 편이 관리와 인수인계 모두 깔끔해.

**이렇게 통일하는 이유**

- 계정이 흩어져 있으면 인증 메일·알림이 여러 곳으로 분산돼서 놓치기 쉬워. 특히 도메인 소유권 확인 메일(§4-2)이나 GCP 예산 알림(§5-6)은 놓치면 사고로 이어져.
- 기존 개인 계정에는 학습용 리포지토리나 실습 커밋이 섞여 있는 경우가 많아. 채용 담당자가 보는 프로필은 정돈된 상태가 유리해.
- 이력서에 적을 이메일 주소와 GitHub 계정 주소가 하나로 묶여서 일관성 있게 보여.

관련 상세: 계정 생성은 §2-1(Google), §2-3(GitHub), 아이디 작명 주의사항은 §2-3의 ⚠️, 2단계 인증은 §2-2·§2-4, 프로필 정리는 §2-5 참고.

## 3. 취업용 Gmail로 고대디 가입 후 도메인 구매

2번에서 만든 취업용 Gmail 계정으로 고대디(GoDaddy)에 가입하고, 포트폴리오에서 쓸 도메인을 구매한다.

### 3-1. 고대디 가입

1. https://www.godaddy.com 접속 → 우측 상단 로그인 → **계정 만들기**
2. **취업용 Gmail 주소**와 비밀번호 입력 (GCP·GitHub와 동일한 계정으로 통일)
3. 수신함에서 이메일 인증 완료

### 3-2. 도메인명 정하기 (구매 전에 먼저 결정)

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

### 3-3. 구매 절차

1. 고대디 로그인 상태에서 검색창에 원하는 도메인 입력
2. 사용 가능하면 장바구니에 담기
3. ⚠️ **부가 옵션은 전부 해제할 것.** 여기서 안 빼면 불필요한 비용이 몇 배로 붙어:

   | 옵션 | 처리 | 이유 |
   |---|---|---|
   | 웹 호스팅 | **해제** | GCP에 올릴 거라 불필요 |
   | 이메일(Microsoft 365 등) | **해제** | 불필요 |
   | SSL 인증서 | **해제** | GCP에서 무료로 자동 발급받음 (§12-7) |
   | 개인정보 보호(WHOIS) | 무료 기본 제공이면 유지, 유료면 판단 | |

4. 결제 기간은 **1년**으로 (여러 해 결제는 나중에 판단)
5. 결제 완료
6. ⚠️ **고대디에서 오는 "도메인 소유권 확인" 메일의 링크를 반드시 클릭할 것.**
   미인증 상태로 15일이 지나면 도메인이 **정지**돼. 결제만 하고 넘어가는 실수가 잦은 지점이야.
7. **자동 갱신은 켜두는 걸 권장.** 갱신을 놓치면 도메인이 풀려서 이력서에 적은 주소가 죽어.

### 3-4. 구매 정보 메모

```
도메인명: ______________________
구매일:   ______________________
만료일:   ______________________
자동갱신: 켬 / 끔        ← 켜두는 걸 권장
```

### [확인] 3번 완료 조건

- [ ] 고대디 "내 제품"에 도메인이 보임
- [ ] 소유권 확인 메일 인증 완료 (도메인에 경고 표시 없음)
- [ ] 도메인명을 위 메모란에 기록함 (이후 `set-vars.ps1`의 `$DOMAIN` 값으로 씀 — §5-4)

> 참고: 이 도메인은 §12-5에서 네임서버를 GCP로 위임하게 돼. 고대디에서는 도메인만 사고, DNS 관리는 Cloud DNS에서 하는 구조야.

관련 상세: §4(도메인 구매), §12-5(네임서버 변경) 참고.

## 4. 취업용 Gmail로 GCP Console 가입

2번에서 만든 취업용 Gmail 계정으로 Google Cloud Console에 가입하고 무료 체험을 등록한다.

### 4-1. 가입 전 확인

- [ ] **Google 계정 2단계 인증이 켜져 있을 것** (§2-2)
      ⚠️ **2026년 10월 20일부터 GCP 콘솔 로그인에 2단계 인증이 의무화**돼. 지금 켜두지 않으면 그 시점에 갑자기 로그인이 막혀. 백업 코드도 함께 저장해둘 것.
- [ ] **해외 결제가 가능한 신용/체크카드** 준비 (§2-7)
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
  - 오늘 가입하면 만료 시점은 **2026년 11월 초**. 그때의 선택지는 §15-2 참고.

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

관련 상세: §1-2(무료 크레딧의 함정), §1-3(계정·비용 명의), §2-2(2단계 인증), §2-7(결제 수단), §5-1(무료 체험 등록) 참고.

