# Codyssey 4-1

### GOAL

1. 네트워크 및 SSH 기본 보안 설정- 가장 먼저 외부 침입을 막고 통신 통로를 확보
2. 계정, 그룹 및 디렉토리 권한 체계 구축 (RBAC) - 사용자 역할 분담, 최소 권한 부여
3. 애플리케이션 실행 환경 및 배포 - 환경 변수를 세팅하고 구동
4. 시스템 관제 자동화 스크립트(`monitor.sh`) 작성
5. 주기적 스케줄링(`crontab`)

--

- SSH 기본 보안 (Port & Root 차단):
    임의의 포트(`20022`)로 변경하는 것만으로도 대부분의 무작위 공격을 차단할 수 있습니다. 
    시스템의 모든 권한을 가진 `root` 계정의 원격 접속을 막아, 해커가 아이디를 root로 고정하고 비밀번호를 찍어 맞추는 시도를 원천 봉쇄합니다.
    
- 최소 권한 원칙과 ACL (Access Control List):
    리눅스의 기본 권한(`rwxrwxrwx`)은 소유자/그룹/기타 사용자 3가지 단계로만 권한을 제어할 수 있어 협업 환경에서 한계가 있습니다. 
    ACL(`setfacl`, `getfacl`)을 사용하면 특정 그룹이나 사용자에게만 추가적인 읽기/쓰기 권한을 정교하게 부여할 수 있어, 공유 디렉토리와 보안 디렉토리를 안전하게 분리할 수 있습니다.
    
- 환경 변수의 역할
    : 애플리케이션에 내부 코드를 수정하지 않고 외부 설정을 주입하는 방법
    : 실행 환경(개발 서버, 실운영 서버 등)이 바뀌더라도 환경 변수만 변경하면 앱이 유연하게 동작할 수 있도록 고정하는 역할
    
- 시계열 로깅과 자원 관리:
    주기적으로 시스템 자원(CPU, 메모리, 디스크)을 수집하여 시계열(시간 순서) 로그로 남겨두어야 사후 분석이 가능합니다. 단, 로그가 무한정 쌓이면 디스크가 가득 차 서버가 멈추므로, 일정 용량/개수가 넘으면 오래된 로그를 압축하거나 지우는 **순환(Rotation) 정책**이 필수적

  --

  # 1. docker

- Ubuntu 22.04 컨테이너 생성

```docker
# /bin/bash 는 리눅스 대표 터미널 Bash 셸이 설치되어 있는 경로.
docker run -it --privileged --name linux-mission -p 20022:20022 -p 15034:15034 ubuntu:22.04 /bin/bash
```

- — privileged : 컨테이너에 모든 장치 접근 권한과 루트 수준의 커널 권한을 부여하여 실행하는 모드 설정

 

- 멈춘 컨테이너 다시 깨우기

```docker
1. docker start linux-mission
	 docker attach linux-mission
	 
2. docker exec -it linux-mission /bin/bash

# 빠져나가기
exit
```

- 개인 계정 생성

```docker
useradd -m -s /bin/bash myuser
echo "myuser:password123!" | chpasswd

# 생성한 계정에 비밀번호 없는 완전한 sudo 권한 부여
echo "myuser ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# 필수 도구 설치
apt-get update && apt-get install -y \
    sudo \
    openssh-server \
    ufw \
    iptables \
    procps \
    net-tools \
    iproute2 \
    systemctl \
    vim \
    acl
  
# 일반 계정으로 전환
su - myuser
```

- **-m:** 사용자 홈 디렉터리(/home/myuser)를 자동으로 함께 생성
**-s:** 사용자가 로그인할 때 사용할 기본 셸 지정(/bash or /sh)
- **su(Swich User) - [계정명]**

# 2. /etc/ssh/sshd_config 수정

### → SSH 포트 변경(20022) 및 Root 원격 접속 차단 설정

- **ssh : 암호화 네트워크 프로토콜**
    
    네트워크를 통해 원격 컴퓨터/서버에 안전하게 로그인 하고 제어할 수 있도록 함.  
    

```docker
# SSH 접속 포트를 기본값인 22에서 20022로 변경.
$ sudo sed -i 's/#Port 22/Port 20022/' /etc/ssh/sshd_config

# root계정 접속 차단
$ sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

$ sudo sh -c 'echo "PermitRootLogin no" >> /etc/ssh/sshd_config'
```

- **sed** **(Stream EDitor): 파일의 특정 텍스트를 찾고 교체하는 리눅스 스트림 에디터 도구.**
    
    **-i** (in-place): 원본 파일을 직접 수정하고 저장하는 옵션.
    
- **sh -c :** shell 실행과 리다이렉션 작업을 root 권한으로 한 번에 실행.
- **>> :** 파일의 기존 내용을 유지하면서 맨 아랫줄에 내용을 덧붙이는 리다이렉션 기호.

---

***SSH 포트 변경과 Root 원격 접속 차단이 왜 기본 보안에 해당하는지?**

: 기본 포트 번호 22를 20022로 바꾸는 것만으로도 무차별 공격의 90% 이상을 차단할 수 있다.

: 해커가 root를 고정으로 두고 비밀번호만 맞히면 서버 전체를 탈취할 수 있어 매우 위험. 일반 사용자 계정으로 로그인한 뒤 필요 시 sudo를 쓰도록 유도하는 것이 보안 표준임.

# 3. UFW 설정

### → ubuntu 환경의 방화벽 UFW에 규칙 추가

### → 방화벽 활성화 및 20022/tcp, 15034/tcp 허용 내역.

```docker
#외부에서 서버로 들어오는 모든 네트워크 요청을 기본적으로 차단.
$ sudo ufw default deny incoming 
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)

#서버에서 외부 인터넷으로 나가는 모든 네트워크 요청을 기본적으로 허용.
$ sudo ufw default allow outgoing
Default outgoing policy changed to 'allow'
(be sure to update your rules accordingly)

#TCP 프로토콜 기반의 20022, 15034 포트로 들어오는 연결을 예외적으로 허용.
$ sudo ufw allow 20022/tcp comment 'New SSH Port'
Rules updated
Rules updated (v6)
$ sudo ufw allow 15034/tcp
Rules updated
Rules updated (v6)

#방화벽 실행.
$ sudo ufw enable
Firewall is active and enabled on system startup
```

```docker
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
20022/tcp                  ALLOW       Anywhere                   # New SSH Port
15034/tcp                  ALLOW       Anywhere                   
20022/tcp (v6)             ALLOW       Anywhere (v6)              # New SSH Port
15034/tcp (v6)             ALLOW       Anywhere (v6)              

Status: active

To                         Action      From
--                         ------      ----
20022/tcp                  ALLOW       Anywhere                  
15034/tcp                  ALLOW       Anywhere                  
20022/tcp (v6)             ALLOW       Anywhere (v6)             
15034/tcp (v6)             ALLOW       Anywhere (v6)  
```

- (v4) 기존의 32비트 IP 주소 체계인 **IPv4** 환경에서 접속할 때 적용되는 규칙
- (v6) 차세대 128비트 IP 주소 체계인 **IPv6** 환경에서 접속할 때 적용되는 규칙

  

---

*TCP프로토콜 

: 데이터를 주고 받을 때의 규칙. 데이터를 보내기 전에 미리 연결을 확인하고 통신을 시작함. 중간에 데이터가 손실되면 재요청해 완벽하게 수신. 웹사이트 접속(HTTP), 파일 전송(FTP), SSH 원격접속(20022) 등 1글자도 틀리면 안 되는 중요한 통신

*UDP프로토콜

: 실시간 영상 스트리밍 등 약간 끊겨도 속도가 중요한 통신에 사용. 빠르게 전송하는 방식.

*UFW

: 다양한 리눅스 환경에서 작동되어 사용하기 쉽게 설계된 넷필터 방화벽을 관리하는 프로그램.

*firewall

: 미리 정의된 보안 규칙에 기반한, 들어오고 나가는 네트워크 트래픽을 모니터링&제어하는 네트워크 보안 시스템.

UFW firewall 차이

# 3. 계정, 그룹 및 디렉토리 권한 체계 구축

- 2그룹 3계정 생성
    - agent-common: admin, dev, test
    - agent-core: admin, dev
    
    - agent-admin (운영/관리, cron 실행자)
    - agent-dev (개발/운영, monitor.sh 작성자)
    - agent-test (QA/테스트)

```docker
$ sudo groupadd agent-common
$ sudo groupadd agent-core

$ sudo useradd -m -s /bin/bash agent-admin
$ sudo useradd -m -s /bin/bash agent-dev
$ sudo useradd -m -s /bin/bash agent-test 

# 계정을 그룹에 추가
$ sudo usermod -aG agent-common,agent-core agent-admin
$ sudo usermod -aG agent-common,agent-core agent-dev
$ sudo usermod -aG agent-common agent-test
```

- **usermod - aG [그룹명] [사용자명]:** 이미 생성된 사용자 계정의 속성(홈 DIR, 소속 그룹, 기본 셸, UID 등)을 수정할 때 사용.
    
    -a: 기존 그룹 목록을 삭제하지 않고 추가.
    
    -G: 사용자가 속할 보조 그룹 목록 설정.
    

- 디렉토리 구조

```docker
# 앱 실행 환경이 될 기본 경로(AGENT_HOME)와 로그 디렉토리 생성.
$ export AGENT_HOME="/home/agent-admin/agent-app"

$ sudo mkdir -p $AGENT_HOME/upload_files
$ sudo mkdir -p $AGENT_HOME/api_keys
$ sudo mkdir -p $AGENT_HOME/bin
$ sudo mkdir -p /var/log/agent-app
```

- 공유 디렉토리(upload_files)와 보안 디렉토리(api_keys, log)의 권한을 분리

접근 권한(핵심 정책) 

최소 권한 원칙: **반드시 필요한 최소한의 접근 권한만 부여**해야 한다는 사이버 보안 개념

- upload_files: group=agent-common, R/W 가능
- api_keys 및 /var/log/agent-app: group=agent-core ONLY, R/W 가능

```docker
sudo chown -R agent-admin:agent-common $AGENT_HOME/upload_files

# agent-common 그룹 전체 R/W 허용
sudo chmod 770 $AGENT_HOME/upload_files

# 새로 생기는 파일도 그룹 권한 상속
sudo chmod g+s $AGENT_HOME/upload_files 
```

```docker
# agent-core 그룹만 R/W 허용

sudo chown -R agent-admin:agent-core $AGENT_HOME/api_keys
sudo chmod 770 $AGENT_HOME/api_keys
-
sudo chown -R agent-admin:agent-core /var/log/agent-app
sudo chmod 770 /var/log/agent-app
-
sudo chown -R agent-admin:agent-core $AGENT_HOME/bin
sudo chmod 770 $AGENT_HOME/bin
```

- **chown [옵션] [소유자]:[소유그룹] [대상경로]:** 소유권 (User, Group) 변경

```docker
# ACL 설정

sudo setfacl -m g:agent-common:rwX,d:g:agent-common:rwX $AGENT_HOME/upload_files

sudo setfacl -m g:agent-core:rwX $AGENT_HOME/api_keys /var/log/agent-app
sudo setfacl -d -m g:agent-core:rwX $AGENT_HOME/api_keys /var/log/agent-app
```

- **setfacl :** 특정 사용자나 그룹에게 세밀한 파일/디렉터리 권한을 부여할 때 사용하는 ACL 설정 명령어.
    
    (Set File Access Control List) 파일 접근 제어 목록 설정
    
    **-u [그룹명]:[권한] :** 특정 사용자에게 권한을 지정합니다.
    
    **-g [그룹명]:[권한] :** 특정 그룹에 권한을 지정합니다.
    
    **-m :** 기존 ACL 규칙을 수정하거나 추가합니다. ****
    
    **-d (default):** 디렉터리 내 새 파일 생성 시 ACL 권한 규칙 자동 상속(복사).
    
- **rwX :** w, r 권한을 주되, 실행 권한(X)을 디렉터리거나, 이미 실행 권한이 있던 파일에만 부여.
    
         (일반 파일이 불필요하게 실행 파일이 되는 것을 방지) 
    
- 소유 권한 확인

```docker
# 계정/그룹 설정 확인
$ id agent-admin && id agent-dev && id agent-test
uid=1001(agent-admin) gid=1003(agent-admin) groups=1003(agent-admin),1001(agent-common),1002(agent-core)
uid=1002(agent-dev) gid=1004(agent-dev) groups=1004(agent-dev),1001(agent-common),1002(agent-core)
uid=1003(agent-test) gid=1005(agent-test) groups=1005(agent-test),1001(agent-common)

# 디렉토리 ACL 권한 확인 
$ sudo getfacl /home/agent-admin/agent-app/api_keys
# file: home/agent-admin/agent-app/api_keys
# owner: agent-admin
# group: agent-core
user::rwx
group::rwx
group:agent-common:rwx
mask::rwx # 최대 권한의 한계선
other::---
default:user::rwx # 모든 디렉터리와 폴더에 자동으로 상속될 권한
default:group::rwx
default:group:agent-common:rwx
default:mask::rwx
default:other::---
```

---

***역할 기반 계정/그룹과 ACL을 통해 “공유 디렉토리”와 “보안 디렉토리”를 분리하는 이유?**  

: 인가된 범위 내에서만 자원을 참조·수정할 수 있도록 접근 경계를 명확히 격리하는 것

*왜 upload_files에만 chmod g+s를 걸었을까?

: user1이 upload_files에 새 파일을 생성하면, 기본적으로 소유 그룹이 user1의 기본 그룹으로 들어감. 그렇게 되면 agent-common 그룹에 속한 user2는 user1이 만든 파일에 접근 불가능.

→ upload_files에 g+s를 걸어두면, 누가 파일을 올리든 무조건 파일의 소유 그룹이 agent-common으로 고정됨. 그룹원 전체가 서로가 올린 파일을 함께 읽고 쓸 수 있음.

*api_keys와 log폴더는?

: 지정된 관리자만 접근해서 읽고 써야 하는 영역

# 4. 애플리케이션 실행 환경 구축

- 앱 파일을 실행용 디렉터리($AGENT_HOME/bin)로 옮기고 실행 권한 추가.

```docker
# 호스트 PC 터미널에서 호스트 안에 있는 파일을 컨테이너 안으로 복사.
% docker cp ~/Downloads/agent-app/agent-app-linux-x86 linux-mission:/home/agent-admin/agent-app/bin/
Successfully copied 6.5MB to linux-mission:/home/agent-admin/agent-app/bin/

$ sudo chmod +x /home/agent-admin/agent-app/bin/agent-app-linux-x86
```

- API 키 파일 생성

```docker
echo "agent_api_key_test" | sudo tee /home/agent-admin/agent-app/api_keys/secret.key > /dev/null
```

- **tee :** 명령어의 출력 결과를 지정한 파일(…/t_secret.key)과 터미널 화면에 동시에 출력
- **/dev/null :** 터미널 화면에 출력되는 문자열 삭제.

- api_keys의 소유권과 권한 지정

```docker
$ sudo chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys/secret.key
$ sudo chmod 660 /home/agent-admin/agent-app/api_keys/secret.key
```

- 필수 환경변수 영구 등록

```docker
# 특정 사용자 계정의 환경 변수 설정 파일
$ sudo sh -c "cat << 'EOF' >> /home/agent-admin/.bashrc

# 앱이 설치된 최상위 홈 디렉터리 경로 정의
export AGENT_HOME=/home/agent-admin/agent-app
# 네트워크 포트 번호 지정
export AGENT_PORT=15034
# ACL 설정 파일 업로드 디렉터리 경로
export AGENT_UPLOAD_DIR=\$AGENT_HOME/upload_files
# API 키 디렉터리 경로
export AGENT_KEY_PATH=\$AGENT_HOME/api_keys
# 로그 파일 저장 디렉터리 경로
export AGENT_LOG_DIR=/var/log/agent-app
EOF"
```

```docker
$ sudo sh -c "cat << 'EOF' >> /home/agent-admin/.bashrc
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=\$AGENT_HOME/upload_files
export AGENT_KEY_PATH=\$AGENT_HOME/api_keys
export AGENT_LOG_DIR=/var/log/agent-app
EOF"
```

- **cat :** 파일 내용을 읽어서 출력함.
    
    **cat << ‘EOF’** : 터미널 화면에서 여러 줄의 글을 입력받아 하나의 텍스트로 만듦.
    
           **‘**EOF**’** : 입력하는 내용 안에 변수나 명령어가 있어도 해석(치환)하지 않고 입력한 문자 그대로 저장. 
    

- 앱 실행

```docker
# 일반 계정으로 전환.
$ sudo su - agent-admin

# 앱 출력 결과를 화면 대신 로그 파일로 저장
$ nohup $AGENT_HOME/bin/agent-app-linux-x86 > $AGENT_LOG_DIR/app_boot.log 2>&1 &
[1] 39
```

- **nohup(no hangup) :** 터미널을 닫아도 프로세스를 계속 실행함.
- **2>&1 :** 에러 메세지(2)도 일반 메세지(1)와 같은 로그 파일로 보냄.
- **& :** 백그라운드에서 앱을 실행
- **[1] 28 :** 현재 터미널 세션에서 background로 실행 중인 작업의 순번 / PID

- 성공 기준

```docker
# Boot Sequence 5단계 [OK] 및 "Agent READY" 확인.
$ cat /var/log/agent-app/app_boot.log
nohup: ignoring input
>>> Starting Agent Boot Sequence...
[1/5] Checking User Account               [OK]
   ... Running as service user 'agent-admin' (uid=1001)
[2/5] Verifying Environment Variables     [OK]
   ... All required Envs correct
[3/5] Checking Required Files             [OK]
   ... Verified 'secret.key' with correct key string.
[4/5] Checking Port Availability          [OK]
   ... Port 15034 is available.
[5/5] Verifying Log Permission            [OK]
   ... Log directory is writable: /var/log/agent-app
------------------------------------------------------------
All Boot Checks Passed!
Agent READY
2026-07-30 05:50:15,271 [INFO] [SafetyGuard] Process priority lowered (nice=10).
2026-07-30 05:50:15,271 [INFO] Agent listening at port 15034
2026-07-30 05:50:15,271 [INFO] === Agent Worker Started ===
2026-07-30 05:50:15,271 [INFO]    > Cycle: 0 -> 256MB/Lv10 -> 0
2026-07-30 05:50:15,271 [INFO] --- Step Info: Mode=UP, CPU Lv=1, Mem=0MB ---
2026-07-30 05:50:15,308 [INFO] [Memory] Increasing... (+25 MB) Total: 25 MB
2026-07-30 05:50:15,308 [INFO] [CPU] Occupy core for 1s (Level 1)
2026-07-30 05:50:17,314 [INFO] --- Step Info: Mode=UP, CPU Lv=2, Mem=25MB ---
2026-07-30 05:50:17,357 [INFO] [Memory] Increasing... (+25 MB) Total: 50 MB
2026-07-30 05:50:17,357 [INFO] [CPU] Occupy core for 2s (Level 2)
2026-07-30 05:50:20,364 [INFO] --- Step Info: Mode=UP, CPU Lv=3, Mem=50MB ---
2026-07-30 05:50:20,368 [INFO] [Memory] Increasing... (+25 MB) Total: 75 MB
2026-07-30 05:50:20,368 [INFO] [CPU] Occupy core for 3s (Level 3)
2026-07-30 05:50:24,376 [INFO] --- Step Info: Mode=UP, CPU Lv=4, Mem=75MB ---
2026-07-30 05:50:24,390 [INFO] [Memory] Increasing... (+25 MB) Total: 100 MB
2026-07-30 05:50:24,390 [INFO] [CPU] Occupy core for 4s (Level 4)
2026-07-30 05:50:29,397 [INFO] --- Step Info: Mode=UP, CPU Lv=5, Mem=100MB ---
2026-07-30 05:50:29,435 [INFO] [Memory] Increasing... (+25 MB) Total: 125 MB
2026-07-30 05:50:29,436 [INFO] [CPU] Occupy core for 5s (Level 5)

# 포트 15034 LISTEN 상태 확인.
$ sudo ss -tulnp | grep 15034
tcp   LISTEN 0      1            0.0.0.0:15034      0.0.0.0:*    users:(("agent-app-linux",pid=102,fd=4))
```

- **ss(Socket Statistics) -t** (TCP소켓만) **-u** (UDP소켓만) **-l** (대기Listening상태 소켓만) **-n** (포트 번호를 숫자로 표시 ex 22 → ssh, 80 → http) **-p** (사용 중인 프로세스 표시)

---

*echo “…” > file을 안 쓰는 이유

: sudo 권한이 필요한 디렉터리에는 echo 명령만 sudo 권한으로 실행되고 ‘>’ 연산자는 일반 사용자 권한으로 실행되어 ‘Permission denied’가 발생.

# 5. 시스템 관제 자동화 스크립트(monitor.sh) 구현

- monitor.sh 구현

```
cat << 'EOF' > monitor.sh
#!/bin/bash

LOG_DIR="/var/log/agent-app"
LOG_FILE="/var/log/agent-app/monitor.log"
APP_NAME="agent-app-linux-arm64"
APP_PORT=15034

CPU_THRESHOLD=80
MEM_THRESHOLD=80
DISK_THRESHOLD=80

MAX_LOG_SIZE=$((10 * 1024 * 1024))   # 10MB
MAX_BACKUPS=10

# ── 로그 함수 ─────────────────────────────────────

log() {
    echo "$1" >> "$LOG_FILE"
}
 
fail() {
    echo "[ERROR] $1"
    log "[ERROR] $1"
    exit 1
}
 
# ── 사전 준비 ─────────────────────────────────────

mkdir -p "$LOG_DIR"
touch "$LOG_FILE"
 
# ── 1. 프로세스 실행 확인 ──────────────────────────

check_process() {
    if ! pgrep -f "$APP_NAME" > /dev/null; then
        fail "Process $APP_NAME is not running"
    fi
    pgrep -f "$APP_NAME" | head -n 1
}
 
# ── 2. 포트 리스닝 확인 ────────────────────────────

check_port() {
    if ! ss -tln | grep -q ":${APP_PORT} "; then
        fail "Port $APP_PORT is not listening"
    fi
}
 
# ── 3. 방화벽 활성 확인 (경고만, 종료하지 않음) ──────

check_firewall() {
    if command -v ufw > /dev/null 2>&1; then
        if command -v systemctl > /dev/null 2>&1; then
            if ! systemctl is-active --quiet ufw; then
                log "[WARNING] UFW firewall is not active"
            fi
}
 
# ── 4. 리소스 사용량 측정 ──────────────────────────

get_cpu_usage() {
    top -bn1 | awk '/Cpu\(s\)/ {printf "%d", $2 + $4}' # $2|user 사용률, $4|system 사용률
}
 
get_mem_usage() {
    free | awk '/Mem:/ {printf "%d", $3/$2 * 100}' # $3|used, $2|total (사용량/전체 비율)
}
 
get_disk_usage() {
    df / | awk 'NR==2 {print $5}' | tr -d '%' # df의 Use% 컬럼
}
 
# ── 5. 임계값 초과 경고 ────────────────────────────

check_threshold() {
    local label="$1" value="$2" threshold="$3"
    if [ "$value" -gt "$threshold" ]; then
        log "${label} 경고: ${value}%"
    fi
}
 
# ── 6. 로그 로테이션 (10MB 초과 시 최대 10개 백업) ───

rotate_log_if_needed() {
    [ -f "$LOG_FILE" ] || return
    local size
    size=$(stat -c%s "$LOG_FILE")
    [ "$size" -gt "$MAX_LOG_SIZE" ] || return
 
    rm -f "${LOG_FILE}.${MAX_BACKUPS}"
    for ((i = MAX_BACKUPS - 1; i >= 1; i--)); do
        [ -f "${LOG_FILE}.${i}" ] && mv "${LOG_FILE}.${i}" "${LOG_FILE}.$((i + 1))"
    done
    mv "$LOG_FILE" "${LOG_FILE}.1"
    touch "$LOG_FILE"
}
 
# ── 메인 실행 흐름 ──────────────────────────────────

main() {
    local pid cpu mem disk timestamp
 
    pid=$(check_process)
    check_port
    check_firewall
 
    cpu=$(get_cpu_usage)
    mem=$(get_mem_usage)
    disk=$(get_disk_usage)
 
    check_threshold "CPU" "$cpu" "$CPU_THRESHOLD"
    check_threshold "MEM" "$mem" "$MEM_THRESHOLD"
    check_threshold "DISK" "$disk" "$DISK_THRESHOLD"
 
    timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    log "[$timestamp] PID:${pid} CPU:${cpu}% MEM:${mem}% DISK_USED:${disk}%"
 
    rotate_log_if_needed
}
 
main
exit 0
```

--

- **#!/bin/bash (쉬뱅/해시뱅) :** #(Hash) + !(Bang) + /bin/bash (실행할 인터프리터 경로)

*[1] Health Check: 프로세스 확인*

`pgrep -f`로 프로세스 이름이 포함된 명령줄을 찾습니다. 없으면 즉시 에러 출력 후 종료(exit 1)

- **pgrep -f :** 전체 명령줄(full command) 로 검색

*[2] 포트 리스닝 확인*

`ss -tln`으로 TCP 리스닝 소켓 목록을 가져와 해당 포트가 열려있는지 확인합니다.

- **ss :** 현재 열려있는 네트워크 연결/포트 목록을 확인
    
    `-t` : TCP 소켓만 확인
    
    `-l` : 연결을 기다리며 대기 중인(Listening) 소켓만 확인
    
    `-n` : 포트나 주소를 사람이 읽는 이름(예: `http`) 대신 숫자 그대로 확인 (`80` 대신 `www`로 나오는 걸 방지).
    

*[3] 상태 점검: 방화벽*

ufw 또는 firewalld 중 설치된 것을 감지해 활성 상태인지 확인합니다.

- **command -v :** 시스템에 설치되어 있는지 확인하고 경로 출력

*[4] 자원 수집*

CPU 사용률(top), 메모리 사용률(free), 디스크 사용률(df)을 각각 계산합니다.

- **top :** 실시간 프로세스/시스템 상태 표시
    
       **-b :** Batch 모드(화면 갱신 없이 출력)
    
       **-n1 :** 1회만 실행
    
- **awk '패턴 {동작}' 파일명 :** 텍스트 파일의 특정 열을 추출하고 데이터를 가공하는 도구

ex) awk ‘/Mem:/’ : “Mem:”으로 시작하는 줄만 처리.

 awk ‘NR==2’  : Number of Record = 2, 2번째 줄

- **free :** 메모리 사용량 표시

- **df / :** 루트 파티션의 디스크 사용량
- **tr [옵션] [문자1] [문자2] (translate) :** 변환, 삭제, 중복 제거를 위해 사용.

ex) tr -d ‘%’ : 숫자 비교를 위해 %문자 삭제

*[5] 임계값 초과 시 경고 로그*

80%를 넘으면 로그 파일에 경고를 추가합니다.

- **-gt (>) :** 초과 비교 연산자

*[6] 상태 로그 기록*

매 실행마다 상태 스냅샷을 기록합니다.

- **`-f :`** 경로가 존재하는 일반 파일인지 확인
- **|| :** 왼쪽이 실패하면 오른쪽을 실행 (OR)

*[7] 로그 로테이션*

로그가 10MB를 넘으면 `.1`~`.10`까지 순환 백업하고 새 로그 파일을 만듭니다.

- **stat** **-c :** 형식을 지정해서 출력

- 스크립트 동작 테스트

```docker
$ ./monitor.sh

$ cat /var/log/agent-app/monitor.log

[2026-07-30 10:17:16] PID:101 CPU:0% MEM:df% DISK_USED:1%
[2026-07-30 10:21:35] PID:101 CPU:0% MEM:3% DISK_USED:1%
[2026-07-30 10:25:15] PID:101 CPU:1% MEM:3% DISK_USED:1%
[2026-07-30 10:26:00] PID:101 CPU:0% MEM:3% DISK_USED:1%
```

---

*인터프리터 

-python 스크립트(.py)를 통역하는 인터프리터 경로 /usr/bin/python3

-bash 셸 스크립트(.sh)를 통역하는 인터프리터 경로 /bin/bash

*awk

리눅스 기본 쉘(`bash`)이 소수점(실수) 계산을 못 하기 때문에 실수 크기 비교용 으로 자주 활용됨.

*-gt

-eq(같음), -lt(작음), -ne(다름)

# 6. crontab 매분 자동 등록 및 검증

- 파일 권한 설정

```docker
$ sudo chown agent-dev:agent-core $AGENT_HOME/bin/monitor.sh
$ sudo chmod 750 $AGENT_HOME/bin/monitor.sh
```

- crontab 등록

```docker
# apt-get update && apt-get install -y cron

# service cron start
```

```docker
$ crontab -e

# 분 시 일 월 요일 (매분 실행)
* * * * * /home/agent-dev/monitor.sh
```

- **crontab -e(edit)**

- 자동 실행 확인
```
tail -f /var/log/agent-app/monitor.log
[2026-08-26 12:45:49] PID:325 CPU:0% MEM:4% DISK_USED:1%
[2026-08-26 12:46:01] PID:325 CPU:0% MEM:3% DISK_USED:1%
[2026-08-26 12:47:02] PID:325 CPU:0% MEM:5% DISK_USED:1%
[2026-08-26 12:48:01] PID:325 CPU:0% MEM:4% DISK_USED:1%
[2026-08-26 12:49:01] PID:325 CPU:0% MEM:5% DISK_USED:1%
```
