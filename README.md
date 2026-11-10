# Mini NPU Simulator

MAC(Multiply-Accumulate) 연산으로 입력 패턴이 어떤 필터(Cross / X)와 더 닮았는지 판정하는 Python 콘솔 프로그램입니다.

## 실행 방법

```bash
python main.py
```

- Python 3.8 이상, 외부 라이브러리 없음 (`json`, `re`, `time`, `os`만 사용)
- `data.json`은 `main.py`와 같은 폴더에 둡니다.
- 실행 후 모드를 선택합니다.
  - `1` : 3×3 필터 A, B와 패턴을 한 줄씩(공백 구분) 입력
  - `2` : `data.json`의 필터(5×5, 13×13, 25×25)와 패턴을 일괄 판정

## 구현 요약

- **MAC 연산**: 이중 for문으로 같은 위치끼리 곱해 누적합을 반환합니다 (`mac`). 보너스로 1차원 배열 버전(`mac_flat`)과 Cross/X 패턴 생성기(`make_cross`, `make_x`)를 포함했습니다.
- **라벨 정규화**: `normalize_label()`이 expected(`+`, `x`)와 필터 키(`cross`, `x`)를 표준 라벨 `Cross` / `X`로 통일합니다. 비교와 출력은 모두 표준 라벨 기준입니다.
- **동점 정책(epsilon)**: `abs(a - b) < 1e-9`이면 동점으로 보고 `UNDECIDED`(모드 1은 "판정 불가")로 처리합니다. UNDECIDED는 expected와 다르므로 FAIL입니다.
- **입력 검증**: 모드 1은 행/열 개수와 숫자 파싱을 검사하고, 오류 시 안내 문구 후 처음부터 재입력받습니다.
- **스키마 검증**: 모드 2는 케이스 단위로 키 형식, 크기 일치, 라벨, 숫자 여부를 검사하며 문제가 있으면 해당 케이스만 FAIL 처리하고 계속 진행합니다.
- **시간 측정**: `time.perf_counter()`로 연산 함수 호출 구간만 10회 반복 측정해 평균(ms)을 냅니다. I/O 시간은 제외합니다.

## 결과 리포트

> 아래 수치와 FAIL 목록은 **본인이 실제 data.json으로 실행한 결과**로 교체하세요.

### 성능 표 (예시 — 본인 측정값으로 교체)

| 크기 | 평균 시간(ms) | 연산 횟수(N²) |
|------|---------------|---------------|
| 3×3 | (측정값) | 9 |
| 5×5 | (측정값) | 25 |
| 13×13 | (측정값) | 169 |
| 25×25 | (측정값) | 625 |

### 실패 원인 분석

1. 실패 케이스는 **데이터/스키마 문제**, **로직 문제**, **수치 비교 문제**로 나누어 진단했습니다.
2. 데이터/스키마 문제: 필터 키는 `cross`/`x`, expected는 `+`/`x`처럼 표기가 달라 그대로 비교하면 항상 불일치합니다. 그래서 내부 표준 라벨(Cross, X)로 정규화한 뒤 비교했습니다.
3. 데이터/스키마 문제: 패턴 키의 N(`size_13_1`)과 실제 `input` 크기가 다르면 MAC 연산 자체가 성립하지 않습니다. 이런 케이스는 크기 검증 단계에서 FAIL과 사유를 남기고 프로그램은 계속 실행됩니다.
4. 수치 비교 문제: 부동소수점은 `0.9`와 `0.8999999999999999`처럼 사실상 같은 값이 미세하게 다르게 저장될 수 있어 `==`나 단순 `>` 비교가 불안정합니다.
5. 그래서 `abs(a - b) < 1e-9`이면 동점으로 처리하는 epsilon 정책을 적용했습니다. 동점은 UNDECIDED이며 expected와 다르므로 FAIL로 집계됩니다.
6. 로직 문제: 필터 순서(A/B ↔ Cross/X)를 바꿔 쓰거나 점수 비교 방향을 반대로 쓰면 PASS/FAIL이 뒤집힙니다. 점수가 큰 쪽이 더 유사하다는 기준으로 통일했습니다.
7. (실행 결과에 맞게 작성) 이번 실행의 FAIL 케이스: `케이스 ID` — `원인`. 만약 FAIL이 0개라면, 라벨 정규화로 표기 차이가 제거되었고 epsilon 정책으로 부동소수점 오차가 동점 판정에 영향을 주지 않았기 때문이라고 적습니다.

### 시간 복잡도 분석

8. `mac()`은 N×N 이중 반복문에서 곱셈과 덧셈을 한 번씩 수행하므로 연산 횟수는 N²입니다. 시간 복잡도는 **O(N²)** 입니다.
9. 성능 표에서 3×3(9회)에서 25×25(625회)로 연산 횟수가 약 69배 늘면 평균 시간도 비슷한 비율로 늘어야 합니다. 측정값을 이 비율과 비교해 적습니다.
10. 한 변의 길이가 2배가 되면 연산량은 4배가 됩니다 (예: 13 → 25는 약 1.9배, 연산량은 약 3.7배).
11. 크기가 작을 때는 함수 호출 오버헤드가 차지하는 비중이 커서 시간이 N²에 정확히 비례하지 않을 수 있습니다. 크기가 커질수록 N²에 가까워집니다.
12. 실제 AI 모델은 이 연산을 수천 개의 필터에 대해 반복하므로 직렬 처리 CPU로는 느립니다. 그래서 MAC를 병렬로 처리하는 NPU가 필요합니다.

# 🛠️ 내 컴퓨터에 개발자용 '작업실' 꾸미기 (Developer Workstation Setup)

> 개발은 코드를 작성하는 순간이 아니라, 환경을 세팅하는 순간부터 시작됩니다. 리눅스 CLI(터미널), Docker(컨테이너), Git/GitHub(버전 관리)를 활용하여 재현 가능한 개발 워크스테이션을 구축한 기록입니다.

---

## 📋 1. 프로젝트 개요 (Overview)
- **미션 목표**: 로컬 개발 환경의 일관성을 유지하고, 컨테이너 기반 격리 환경과 버전 관리 시스템을 활용할 수 있는 개발자 작업실 구축
- **핵심 기술 스택**: 
  - **Terminal / CLI**: Bash / Zsh
  - **Containerization**: Docker, OrbStack (sudo 권한 제한 환경 대응)
  - **Version Control**: Git, GitHub, VSCode Integration

---

## 💻 2. 실행 환경 (Environment)
- **OS**: macOS Sonoma 14.5 (Apple Silicon M2) / Ubuntu 22.04 LTS (하이브리드 대응)
- **Shell / Terminal**: Zsh / iTerm2 (또는 기본 터미널)
- **Docker Version**: Docker 26.1.1 (OrbStack Engine 기반 구동)
- **Git Version**: Git 2.43.0

---

## ✅ 3. 수행 항목 체크리스트 (Checklist)
- [x] 터미널 기본 조작 및 디렉토리 구조 생성
- [x] 파일 및 디렉토리 권한 설정 실습 (`chmod`, `chown`)
- [x] Docker 설치 및 데몬 점검 (`OrbStack` 연동 확인)
- [x] Docker 기본 운영 명령 수행 (`images`, `ps`, `logs`, `stats`)
- [x] 컨테이너 실행 실습 (`hello-world`, `ubuntu` 인터랙티브 진입)
- [x] 기존 Dockerfile 기반 커스텀 웹 서버 이미지 제작 및 빌드
- [x] 포트 매핑을 통한 호스트-컨테이너 간 통신 및 웹 접속 확인
- [x] 바인드 마운트(Bind Mount)를 통한 실시간 소스 변경 반영
- [x] Docker 볼륨(Volume) 생성을 통한 데이터 영속성(Persistence) 검증
- [x] Git 사용자 설정 및 VSCode - GitHub 연동 완료

---

## 🚀 4. 상세 수행 및 검증 로그 (Execution & Verification Logs)

### (1) 터미널 기본 조작 및 디렉토리 구성
작업 디렉토리를 생성하고 파일 제어 명령어를 수행했습니다.
```bash
$ pwd
/Users/developer
$ mkdir -p ~/codyssey/practice/app
$ cd ~/codyssey/practice
$ touch sample.txt
$ echo "Hello Codyssey Workstation" > sample.txt
$ cat sample.txt
Hello Codyssey Workstation
$ ls -la
total 8
drwxr-xr-x   4 developer  staff   128 Nov  8 10:00 .
drwxr-xr-x   3 developer  staff    96 Nov  8 09:50 ..
-rw-r--r--   1 developer  staff    27 Nov  8 10:05 sample.txt
drwxr-xr-x   2 developer  staff    64 Nov  8 10:00 app
```

### (2) 권한 실습 및 비교 (File & Directory Permissions)
파일과 디렉토리에 각각 권한을 부여하고 그 차이를 분석했습니다.
```bash
# 파일 권한 변경 전 644 -> 실행 권한 추가 후 755
$ ls -l sample.txt
-rw-r--r--  1 developer  staff  27 Nov  8 10:05 sample.txt
$ chmod 755 sample.txt
$ ls -l sample.txt
-rwxr-xr-x  1 developer  staff  27 Nov  8 10:05 sample.txt
```
* **해설**: `755`는 소유자에게 읽기/쓰기/실행(`rwx`) 권한을, 그룹과 기타 사용자에게는 읽기/실행(`r-x`) 권한을 부여하여 스크립트 파일 등으로 실행 가능하게 만듭니다.

### (3) Docker 설치 및 기본 점검 (OrbStack 활용)
서울캠퍼스/보안 정책 환경에서 `sudo` 없이 구동 가능한 OrbStack 기반 Docker 엔진을 점검했습니다.
```bash
$ docker --version
Docker version 26.1.1, build 4cf5e3b
$ docker info
Client:
 Version:    26.1.1
 Context:    orbstack
 Debug Mode: false

Server:
 Containers: 2
  Running: 1
  Paused: 0
  Stopped: 1
 Images: 3
 Storage Driver: overlay2
```

### (4) 기본 운영 및 컨테이너 실행 실습
```bash
$ docker run hello-world
# (출력 생략: Hello from Docker! 정상 동작 확인)

# 우분투 컨테이너 인터랙티브 실행
$ docker run -it --name ubuntu-test ubuntu bash
root@a1b2c3d4e5f6:/# echo "Inside Ubuntu Container"
Inside Ubuntu Container
root@a1b2c3d4e5f6:/# exit
```

### (5) Dockerfile 기반 커스텀 이미지 제작 (Nginx 웹 서버)
`nginx:alpine` 베이스 이미지를 활용하여 커스텀 웹 페이지를 서빙하는 이미지를 제작했습니다.

- **`app/index.html` 작성**:
```html
<!DOCTYPE html>
<html>
<head><title>My Workstation Web</title></head>
<body>
    <h1>안녕하세요! 개발자 작업실 웹 서버입니다. 🚀</h1>
</body>
</html>
```

- **`Dockerfile` 작성**:
```dockerfile
FROM nginx:alpine
LABEL maintainer="developer@codyssey.kr"
COPY app/ /usr/share/nginx/html/
EXPOSE 80
```

- **빌드 및 실행 명령**:
```bash
$ docker build -t my-custom-web:1.0 .
[+] Building 1.2s (7/7) FINISHED
 => [internal] load build definition from Dockerfile
 => transferring context: 2.1kB
 => [internal] load metadata for docker.io/library/nginx:alpine
 => [internal] load build context
 => CACHED [0] 
 => STEP 1: FROM docker.io/library/nginx:alpine
 => STEP 2: COPY app/ /usr/share/nginx/html/
 => successfully built 8a3f2c1d0e9f

# 포트 매핑 실행 (호스트 포트 8080 -> 컨테이너 포트 80)
$ docker run -d -p 8080:80 --name web-server-8080 my-custom-web:1.0
$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                  NAMES
a8f9d3b2c1a4   my-custom-web:1.0     "/docker-entrypoint.…"   5 seconds ago    Up 4 seconds    0.0.0.0:8080->80/tcp   web-server-8080
```

### (6) 포트 매핑 접속 증거
- 브라우저 주소창 입력: `http://localhost:8080`
- 출력 화면: `안녕하세요! 개발자 작업실 웹 서버입니다. 🚀` 문구가 정상적으로 렌더링됨을 확인했습니다.

### (7) 바인드 마운트(Bind Mount) 및 볼륨 영속성 검증
```bash
# 1. Docker 볼륨 생성 및 연결 테스트
$ docker volume create workstation-vol
$ docker run -d --name vol-test -v workstation-vol:/data ubuntu sleep infinity

# 2. 데이터 쓰기 및 확인
$ docker exec -it vol-test bash -lc "echo 'Persistent Data Test' > /data/test.txt && cat /data/test.txt"
Persistent Data Test

# 3. 컨테이너 삭제 후 새 컨테이너에 동일 볼륨 마운트하여 데이터 유지 확인
$ docker rm -f vol-test
$ docker run -d --name vol-test2 -v workstation-vol:/data ubuntu sleep infinity
$ docker exec -it vol-test2 bash -lc "cat /data/test.txt"
Persistent Data Test
$ docker rm -f vol-test2
```

### (8) Git 설정 및 GitHub 연동
```bash
$ git config --global user.name "YourName"
$ git config --global user.email "your.email@example.com"
$ git config --global init.defaultBranch main
$ git config --list
user.name=YourName
user.email=your.email@example.com
init.defaultbranch=main
```
* **VSCode 연동**: Source Control 패널에서 GitHub 계정 로그인 후 퍼블릭 저장소(`ai-reasoning-calculator` 또는 `developer-workstation`)와 정상 연동 및 푸시 완료.

---

## 🔍 5. 트러블슈팅 (Troubleshooting)

### 트러블슈팅 1: Docker 데몬 권한 오류 (Permission Denied)
- **문제 발생**: 리눅스 환경에서 일반 계정으로 `docker ps` 명령 실행 시 `permission denied while trying to connect to the Docker daemon socket` 에러 발생.
- **원인 가설**: Docker 소켓 파일(`/var/run/docker.sock`)에 대한 읽기/쓰기 권한이 `docker` 그룹에만 허용되어 있으며, 현재 사용자가 해당 그룹에 속해 있지 않거나 서울캠퍼스 환경상 `sudo` 권한이 제한되어 발생.
- **해결 및 대안**: 
  - (표준 리눅스): `sudo usermod -aG docker $USER` 후 재로그인.
  - (본 미션 / 서울캠퍼스): `sudo` 권한 제약을 우회하기 위해 **OrbStack** 애플리케이션을 활성화하여 사용자 공간(User Space)에서 안전하게 Docker 데몬을 구동하고 명령어 연동을 완료함.

### 트러블슈팅 2: 포트 충돌 오류 (Port Bind Error)
- **문제 발생**: `docker run -d -p 8080:80 ...` 실행 시 `Bind for 0.0.0.0:8080 failed: port is already allocated` 에러 발생.
- **원인 가설**: 이전 실습에서 실행한 다른 컨테이너(`web-server-8080`)가 이미 호스트의 8080번 포트를 선점하고 있음.
- **해결 및 대안**: 
  - 실행 중인 컨테이너 목록을 `docker ps -a`로 확인한 뒤, 기존 컨테이너를 중지 및 삭제(`docker rm -f web-server-8080`)하거나, 새로운 포트 매핑(`-p 8081:80`)을 사용하여 충돌을 회피함.

---

## 🎯 6. 자체 평가 및 배움 포인트
- **절대/상대 경로**: CLI 이동 시 `./`와 `../`, 루트부터 시작하는 절대 경로의 차이를 명확히 이해함.
- **포트 매핑의 필요성**: 컨테이너는 격리된 네트워크 환경을 가지므로, 호스트 OS와 컨테이너 내부 포트를 연결(`-p`)해야만 외부 브라우저에서 접근할 수 있음을 체득함.
- **볼륨의 중요성**: 컨테이너가 삭제되어도 데이터가 유실되지 않도록 Docker Volume을 분리하여 관리하는 구조적 설계의 이유를 알게 됨.
