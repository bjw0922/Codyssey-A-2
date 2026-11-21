# Windows 10 개발 워크스테이션 구축

> **수행 상태: 검증 대기**  
> 이 문서는 Windows 10 + WSL2 Ubuntu를 기반으로 한 CLI·Docker·Git 개발 환경 구축 과제의 실행 절차 및 검증 기록 양식입니다. 명령 결과는 실제 수행 후 입력하며, 완료 표시와 스크린샷은 검증한 항목에 한해 추가합니다.

## 0. 필수 증거 체크리스트

- [ ] Windows 빌드 및 WSL2 Ubuntu 확인
- [ ] Ubuntu 배포판·셸·커널·Git·Docker 버전 기록
- [ ] 터미널 파일 생성·복사·이동·삭제 로그 기록
- [ ] 파일 1개, 디렉터리 1개의 권한 변경 전후 기록
- [ ] `docker --version`, `docker info` 성공
- [ ] `hello-world` 실행 및 Ubuntu 컨테이너 내부 진입
- [ ] `docker images`, `docker ps -a`, `docker logs`, `docker stats` 기록
- [ ] Dockerfile 작성 및 이미지 빌드
- [ ] 8088/8081 포트 매핑 후 브라우저 접속 증거
- [ ] 바인드 마운트 수정 전후 응답 비교
- [ ] Named volume 데이터 유지 확인
- [ ] `git config --list` 확인 결과(민감정보 검토 후)
- [ ] Git 설정, GitHub 및 VS Code 연동 증거
- [ ] 트러블슈팅 실제 발생 사례 2건 이상 기록
- [ ] 선택: Compose 단일 서비스·2개 서비스 통신·환경 변수
- [ ] 선택: GitHub SSH 인증

## 1. 프로젝트 개요

이 프로젝트는 터미널 명령어, Linux 권한, Docker 이미지·컨테이너, 사용자 정의 웹 서버, 네트워크 포트 매핑, 스토리지 연결 및 Git/GitHub 협업 환경을 직접 구성하고 검증하는 실습입니다. 실행 명령과 관찰한 결과를 분리하여 기록함으로써 다른 사람이 README만 보고 같은 절차를 재현할 수 있도록 합니다.

## 2. 실행 환경

| 항목 | 실제 실행 환경 |
|---|---|
| 호스트 OS | Windows 10 [에디션, 빌드 확인 후 기입] |
| Linux 환경 | WSL2 / Ubuntu [실제 버전] |
| 터미널 | [예: Windows Terminal에서 Ubuntu 실행] |
| Shell | [실제 `echo "$SHELL"` 결과] |
| Kernel | [실제 `uname -r` 결과] |
| Docker 연결 방식 | [Docker Desktop WSL 통합 / Ubuntu Docker Engine 중 실제 방식] |
| Docker / Compose | [실제 버전] |
| Git | [실제 버전] |
| VS Code | [설치·연동 확인 후 기입] |
| 실행 일시 | [실제 수행 일자] |

### 환경 확인 명령

Windows PowerShell:

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption,Version,BuildNumber
wsl -l -v
```

Ubuntu 터미널:

```bash
pwd
whoami
cat /etc/os-release
uname -r
echo "$SHELL"
docker --version
docker compose version
docker info
git --version
```

**결과:** [명령/출력 및 캡처를 확인한 뒤 여기에 요약]  
**증거:** [환경 화면 캡처](docs/screenshots/README.md)

> 주의: Docker Desktop의 WSL 통합과 Ubuntu Docker Engine은 설치·실행 위치가 다릅니다. 현재 활성화된 Docker 컨텍스트와 서버를 확인한 후 하나의 구성으로 실습합니다. 프로젝트는 가능하면 Ubuntu의 `~/` 아래에 두어 Linux 권한과 파일 시스템 실습에 사용합니다.

## 3. 프로젝트 파일 구조

```text
windows10-workstation/
├── README.md
├── Dockerfile
├── compose.yaml
├── .env.example
├── .gitignore
├── site/
│   ├── index.html
│   └── style.css
├── bind-site/
│   └── index.html
├── docs/
│   ├── logs/
│   │   ├── terminal.md
│   │   ├── docker.md
│   │   ├── storage.md
│   │   ├── compose.md
│   │   └── git.md
│   └── screenshots/
└── labs/  (터미널 실습 시 생성, Git 추적 제외)
```

## 4. 터미널 기본 조작과 파일 권한

### 4-1. 파일·디렉터리 명령어

**실행 위치: Ubuntu 터미널**

```bash
cd ~/windows10-workstation
mkdir -p labs/cli-practice && cd labs/cli-practice
pwd
ls -la
mkdir docs
touch notes.txt
echo "WSL2 terminal practice" > notes.txt
cat notes.txt
cp notes.txt docs/notes-copy.txt
mv docs/notes-copy.txt docs/renamed.txt
ls -la docs
rm docs/renamed.txt
rmdir docs
```

- `pwd` 현재 위치 확인
- `ls -la` 숨김 파일 포함 목록 확인
- `mkdir` 디렉터리 생성 / `touch` 빈 파일 생성
- `cp` 복사 / `mv` 이동·이름 변경 / `rm` 삭제
- `cat` 파일 내용 확인

**실제 출력:** [docs/logs/terminal.md](docs/logs/terminal.md)

### 4-2. 파일 권한 변경 전/후

```bash
cd ~/windows10-workstation/labs/cli-practice
touch permission.txt
mkdir permission-dir
ls -l permission.txt
ls -ld permission-dir
chmod 600 permission.txt
chmod 700 permission-dir
ls -l permission.txt
ls -ld permission-dir
chmod 644 permission.txt
chmod 755 permission-dir
ls -l permission.txt
ls -ld permission-dir
```

`r=4`, `w=2`, `x=1`이며, 세 자릿수는 소유자·그룹·기타 사용자 순서입니다. `644`는 `rw-r--r--`, `755`는 `rwxr-xr-x`를 의미합니다. 디렉터리의 `x`는 그 경로를 통과·진입할 수 있는 권한입니다.

**변경 전/후 실제 출력:** [docs/logs/terminal.md](docs/logs/terminal.md)

### 절대 경로와 상대 경로

- 절대 경로: `/home/<Ubuntu사용자>/windows10-workstation/site/index.html`
- 상대 경로: 프로젝트 루트에서 `site/index.html`
- 상대 경로는 현재 작업 위치에 따라 의미가 달라집니다.

## 5. Docker 설치 확인과 컨테이너 운영

### 5-1. Docker 동작 확인

```bash
docker --version
docker compose version
docker info
docker run --rm hello-world
docker images
docker ps
docker ps -a
```

`docker --version`은 클라이언트 설치 확인, `docker info`는 데몬과의 실제 연결 확인입니다. `hello-world`의 성공 메시지도 기록합니다.

**실제 로그:** [docs/logs/docker.md](docs/logs/docker.md)

### 5-2. Ubuntu 컨테이너 내부 명령

```bash
docker pull ubuntu:24.04
docker run -dit --name workstation-ubuntu ubuntu:24.04 bash
docker exec -it workstation-ubuntu bash
# 여기부터는 컨테이너 내부
pwd
ls -la
echo 'Hello from Ubuntu container'
exit
# 다시 호스트 Ubuntu 터미널
docker ps -a
docker stats --no-stream
docker stop workstation-ubuntu
docker start workstation-ubuntu
```

### 5-3. 로그 및 attach/exec

```bash
docker run --name workstation-log-demo ubuntu:24.04 echo 'Docker log check'
docker logs workstation-log-demo
docker ps -a
docker attach workstation-ubuntu
```

`attach`는 메인 프로세스의 입출력에 연결합니다. 위 컨테이너의 셸에서 나오되 종료하지 않으려면 `Ctrl+P` 다음 `Ctrl+Q`를 사용합니다. `exec`는 실행 중인 컨테이너 안에서 별도의 프로세스를 실행합니다. `exit`로 메인 셸을 종료하면 컨테이너가 중지될 수 있습니다.

**명령어 및 실제 출력:** [docs/logs/docker.md](docs/logs/docker.md)

## 6. Dockerfile 기반 커스텀 웹 서버

### 선택한 베이스와 커스터마이징

- 베이스 이미지: `nginx:1.27-alpine`
- `COPY site/`: 과제용 정적 HTML과 CSS를 웹 서버 콘텐츠로 지정
- `ENV APP_ENV=local`: 실행 환경을 표시하기 위한 기본 환경 변수 선언
- `HEALTHCHECK`: 컨테이너 내부 HTTP 응답 상태를 점검

실행 파일: [Dockerfile](Dockerfile), [HTML](site/index.html), [CSS](site/style.css)

```bash
cd ~/windows10-workstation
docker build -t windows10-workstation-web:1.0 .
docker images
docker run -d --name workstation-web-8088 -p 127.0.0.1:8088:80 windows10-workstation-web:1.0
docker ps
docker logs workstation-web-8088
curl -i http://127.0.0.1:8088
```

**성공 기준:** HTTP 200, HTML 내용 표시, 브라우저 `http://localhost:8088` 접속. `-p 127.0.0.1:8088:80`은 호스트의 로컬 포트 8088을 컨테이너 포트 80에 연결합니다.

다른 호스트 포트 검증:

```bash
docker run -d --name workstation-web-8081 -p 127.0.0.1:8081:80 windows10-workstation-web:1.0
curl -i http://127.0.0.1:8081
docker inspect --format '{{.State.Health.Status}}' workstation-web-8088
```

**실제 출력:** [docs/logs/docker.md](docs/logs/docker.md)  
**브라우저 캡처(주소창과 화면 포함):** [docs/screenshots](docs/screenshots/README.md)

## 7. 바인드 마운트 변경 반영

바인드 마운트는 Ubuntu의 호스트 폴더를 컨테이너의 특정 경로에 연결합니다. 이미지 안의 콘텐츠를 새로 빌드하지 않고 호스트 파일의 변경을 확인할 수 있습니다.

```bash
cd ~/windows10-workstation
docker run -d --name workstation-bind -p 127.0.0.1:8089:80 \
  --mount type=bind,source="$(pwd)/bind-site",target=/usr/share/nginx/html \
  windows10-workstation-web:1.0
curl -i http://127.0.0.1:8089
```

`bind-site/index.html`의 제목 문구를 `바인드 마운트 변경 후`로 바꾸고 저장한 다음 다시 실행:

```bash
curl -i http://127.0.0.1:8089
```

주의: 디렉터리를 마운트하면 이미지에 포함된 해당 디렉터리의 원래 파일들은 마운트된 콘텐츠에 가려집니다. 따라서 `bind-site/`에는 독립 실행 가능한 HTML을 별도로 준비했습니다.

**변경 전/후 실제 응답:** [docs/logs/storage.md](docs/logs/storage.md)

## 8. Docker Named Volume 영속성

```bash
docker volume create workstation-data
docker run -d --name workstation-volume1 -v workstation-data:/data ubuntu:24.04 sleep infinity
docker exec workstation-volume1 sh -c 'echo "Saved after removal" > /data/hello.txt'
docker exec workstation-volume1 cat /data/hello.txt
docker rm -f workstation-volume1
docker run -d --name workstation-volume2 -v workstation-data:/data ubuntu:24.04 sleep infinity
docker exec workstation-volume2 cat /data/hello.txt
docker volume ls
```

**성공 기준:** 첫 번째 컨테이너 삭제 후에도 두 번째 컨테이너에서 `Saved after removal`을 읽을 수 있습니다. Named volume은 컨테이너 수명과 별개로 Docker가 관리하는 저장소입니다.

**삭제 전/후 실제 출력:** [docs/logs/storage.md](docs/logs/storage.md)

## 9. Git·GitHub·VS Code 연동

```bash
git --version
git config --global init.defaultBranch main
git config --global user.name "<Git에 표시할 이름>"
git config --global user.email "<GitHub에 사용할 이메일>"
git config --list
git init
git add .
git commit -m "Init: create Windows workstation project"
git branch -M main
```

`git config --list` 결과를 제출할 때 이메일 등 개인정보와 비밀 정보는 마스킹합니다. `git init`은 프로젝트를 새 Git 저장소로 만들 때 **한 번만** 실행합니다. 이미 `git clone`으로 가져온 저장소라면 `git init`을 반복할 필요가 없습니다.

GitHub에서 새 **비어 있는** 저장소를 만든 뒤:

```bash
git remote add origin https://github.com/<GitHub계정>/<새저장소명>.git
git remote -v
git push -u origin main
```

VS Code에서 WSL 확장으로 해당 폴더를 열고 GitHub에 로그인한 뒤 변경 파일과 원격 저장소 상태를 확인합니다. 인증 토큰, 이메일, 개인키 등이 로그에 공개되지 않도록 조심합니다.

**설정 확인 및 연결 증거:** [docs/logs/git.md](docs/logs/git.md), [docs/screenshots](docs/screenshots/README.md)

## 10. 선택 과제: Docker Compose

[compose.yaml](compose.yaml)은 NGINX 웹 서버 `web`과 내부 HTTP 요청을 보내는 BusyBox 서비스 `inspector`를 함께 구성합니다. 서비스 이름 `web`은 Compose 내부 네트워크에서 호스트 이름으로 해석됩니다.

```bash
cd ~/windows10-workstation
cp .env.example .env
docker compose config
docker compose up -d --build
docker compose ps
docker compose logs inspector
curl -i http://127.0.0.1:8091
docker compose down
```

`WEB_PORT=8091`은 컨테이너 내부 포트를 바꾸는 것이 아니라 외부에서 접근할 호스트 포트를 설정합니다. `APP_ENV`는 컨테이너 환경 변수로 전달됩니다.

**실제 명령/출력:** [docs/logs/compose.md](docs/logs/compose.md)

## 11. 선택 과제: GitHub SSH

```bash
ssh-keygen -t ed25519 -C "<이메일>"
cat ~/.ssh/id_ed25519.pub
ssh -T git@github.com
```

공개키(`.pub`)를 GitHub **Settings → SSH and GPG keys**에 등록하고 연결을 확인합니다. 비밀키 `~/.ssh/id_ed25519`는 절대로 공유하거나 Git에 추가하지 않습니다. 원격 주소 변경이 필요할 때만 `git remote set-url origin git@github.com:<계정>/<저장소>.git`을 실행합니다.

## 12. 검증 방법·결과 위치

| 검증 대상 | 실행 명령 | 결과 링크 |
|---|---|---|
| 환경·터미널·권한 | `pwd`, `ls -la`, `chmod`, `ls -l` | [terminal.md](docs/logs/terminal.md) |
| Docker 엔진·운영 | `docker info`, `docker ps -a`, `docker logs` | [docker.md](docs/logs/docker.md) |
| 빌드·포트 접속 | `docker build`, `curl` | [docker.md](docs/logs/docker.md) |
| 바인드·볼륨 | `docker run --mount`, `docker volume` | [storage.md](docs/logs/storage.md) |
| Compose | `docker compose up/ps/logs/down` | [compose.md](docs/logs/compose.md) |
| Git 및 연동 | `git config`, `git remote -v` | [git.md](docs/logs/git.md) |
| 브라우저·VS Code | 실제 화면 캡처 | [screenshots](docs/screenshots/README.md) |

## 13. 트러블슈팅 (실제 발생 사례 최소 2건)

### 사례 1 — [실제 발생한 문제 제목]

- **문제:** [실제 오류 메시지]
- **원인 가설:** [원인을 추정한 이유]
- **확인:** [실제로 실행한 확인 명령과 출력]
- **해결/대안:** [실제 변경 사항 및 확인 결과]
- **증거:** [해당 로그 또는 화면 캡처 링크]

### 사례 2 — [실제 발생한 문제 제목]

- **문제:** [실제 오류 메시지]
- **원인 가설:** [원인을 추정한 이유]
- **확인:** [실제로 실행한 확인 명령과 출력]
- **해결/대안:** [실제 변경 사항 및 확인 결과]
- **증거:** [해당 로그 또는 화면 캡처 링크]

> 문제가 발생하지 않았다면 가상의 사고를 실제 문제처럼 기록하지 않습니다. 대신 재현 가능한 장애 실험을 수행했다는 점과 실제 관찰 결과를 명시합니다.

## 14. 보안·재현성 관련 주의

- 비밀번호, 토큰, SSH 개인키, 인증 코드를 GitHub에 올리지 않습니다.
- 스크린샷에는 명령어 및 실제 출력, 브라우저 접속의 경우 주소창을 함께 담습니다.
- Windows의 `C:\` 경로보다 WSL Ubuntu의 `~/windows10-workstation`에서 소스를 관리하는 것이 Linux 파일 권한 실습에 적합합니다.
- WSL2, Docker Desktop/Engine, 포트 사용 현황에 따라 출력값이 다를 수 있으므로 확인 결과를 사실대로 기록합니다.
- Docker 컨테이너 삭제와 볼륨 삭제는 다릅니다. 영속성 검증에 사용한 볼륨은 시험 중 삭제하지 않습니다.

## 15. 실제 결과 수집 순서와 상태 관리

**실행하지 않은 작업은 `[ ]`로 유지합니다.** 아래 명령은 파일로 보관할 출력의 일부를 직접 수집하는 예시이며, 성공 여부는 컴퓨터에서 확인한 뒤 기록합니다.

```bash
cd ~/windows10-workstation
mkdir -p docs/evidence-local
# 이 디렉터리는 로컬 원본 기록용이며, 공개 전 민감정보 검토 필요
script -q docs/evidence-local/session.txt
# 실습 명령들을 터미널에 직접 입력
exit
```

`script`는 터미널 입력·출력 전체를 저장하므로 **비밀번호나 토큰을 입력하지 마세요.** 로그 파일을 검토하고 필요한 출력만 `docs/logs/*.md`로 옮깁니다. 공개할 로그와 스크린샷에서 개인 경로·이메일 등 불필요한 개인정보를 마스킹하세요.

```bash
# 검증용 핵심 명령 (Ubuntu 터미널)
docker info --format '{{.ServerVersion}}'
docker ps -a
docker images
docker stats --no-stream
docker volume ls
git status
```

이상의 출력 및 브라우저 접속 캡처를 README 각 절의 링크에 추가하고 나서만 체크박스를 `[x]`로 변경합니다.

## 16. 최종 제출 전 점검

- [ ] README의 OS/버전/일시가 실환경과 일치함
- [ ] 코드 파일 및 로그 문서의 링크가 모두 열림
- [ ] 명령어와 **실제 출력**이 함께 기록됨
- [ ] 브라우저 주소창이 보이는 스크린샷을 연결함
- [ ] 저장 및 마운트 전후를 비교할 수 있음
- [ ] 실제 트러블슈팅 2건 기록 완료
- [ ] GitHub 저장소에 필요한 결과물이 푸시됨
- [ ] 개인 정보 및 인증 비밀 값 점검 완료
