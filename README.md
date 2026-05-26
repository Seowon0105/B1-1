# Linux 시스템 보안 및 관제 자동화 과제

> **Ubuntu 22.04 LTS** 환경에서 Agent App을 안전하게 설치하고,  
> 시스템 상태를 자동으로 감시하는 환경을 구성한 과제입니다.

---

## 📁 제출 산출물

```
📄 README.md          ← 요구사항 수행 내역서 (이 파일)
📜 monitor.sh         ← 시스템 관제 자동화 스크립트
```

---

## 📋 목차

1. [전체 구성 개요](#1-전체-구성-개요)
2. [SSH 보안 설정](#2-ssh-보안-설정)
3. [방화벽UFW 설정](#3-방화벽ufw-설정)
4. [계정  그룹 구성](#4-계정--그룹-구성)
5. [디렉토리 구조 및 권한](#5-디렉토리-구조-및-권한)
6. [환경 변수 및 키 파일](#6-환경-변수-및-키-파일)
7. [애플리케이션 실행](#7-애플리케이션-실행)
8. [monitor.sh 구현](#8-monitorsh-구현)
9. [crontab 자동 실행 등록](#9-crontab-자동-실행-등록)
10. [로그 파일 관리](#10-로그-파일-관리)
11. [증거 자료 체크리스트](#11-증거-자료-체크리스트)

---

## 1. 전체 구성 개요

```
[보안 기반]          [실행 환경]            [자동 감시]
SSH 포트 변경   →   계정/폴더 구성    →   monitor.sh 작성
방화벽 설정     →   환경 변수 등록    →   cron 자동 실행
Root 접속 차단  →   앱 실행 확인      →   로그 누적 확인
```

앞 단계가 제대로 안 되면 뒷 단계가 동작하지 않는 구조입니다.  
예를 들어 폴더 권한이 잘못 설정되면 `monitor.sh`가 로그를 쓸 수 없습니다.

---

## 2. SSH 보안 설정

### 왜 하는가
- 기본 포트(22)는 전 세계 해커 프로그램이 자동으로 침입 시도를 합니다.
- 포트를 바꾸는 것만으로도 자동화된 공격 대부분을 막을 수 있습니다.
- root로 직접 접속하면 서버 전체가 탈취될 수 있어 반드시 차단합니다.

### 설정 명령어

```bash
# sshd_config 파일 수정
sudo sed -i 's/^#Port 22/Port 20022/'       /etc/ssh/sshd_config
sudo sed -i 's/^Port 22/Port 20022/'        /etc/ssh/sshd_config
sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# 변경 사항 적용
sudo systemctl restart sshd
```

### 확인 방법 및 결과

```bash
$ sudo grep -E '^Port|^PermitRootLogin' /etc/ssh/sshd_config
Port 20022
PermitRootLogin no

$ ss -tulnp | grep sshd
tcp  LISTEN  0.0.0.0:20022  users:(("sshd",pid=1234,fd=3))
```

---

## 3. 방화벽(UFW) 설정

### 왜 하는가
- 서버에는 65535개의 포트가 존재합니다.
- 방화벽 없이는 누구나 아무 포트로나 접근할 수 있습니다.
- 필요한 포트 **2개만** 열고 나머지는 전부 차단합니다.

| 포트 | 용도 |
|------|------|
| `20022/tcp` | SSH 원격 접속 |
| `15034/tcp` | Agent App |

### 설정 명령어

```bash
# 기본 정책: 들어오는 건 전부 거부
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 필요한 포트만 허용
sudo ufw allow 20022/tcp
sudo ufw allow 15034/tcp

# 방화벽 활성화 (반드시 위 두 줄 이후에 실행!)
sudo ufw --force enable
```

> ⚠️ **주의:** SSH 포트를 허용하기 전에 UFW를 켜면 서버에서 쫓겨납니다.

### 확인 방법 및 결과

```bash
$ sudo ufw status verbose
Status: active

To              Action    From
──              ──────    ────
20022/tcp       ALLOW IN  Anywhere
15034/tcp       ALLOW IN  Anywhere
```

---

## 4. 계정 / 그룹 구성

### 왜 하는가
- 모든 작업을 하나의 계정으로 하면, 그 계정이 탈취됐을 때 모든 게 노출됩니다.
- 역할마다 계정을 분리하고 접근 가능한 범위를 제한합니다 (최소 권한 원칙).

### 구성표

| 계정 | 소속 그룹 | 역할 |
|------|-----------|------|
| `agent-admin` | agent-common, **agent-core** | 운영/관리, cron 실행 |
| `agent-dev`   | agent-common, **agent-core** | 개발, monitor.sh 작성 |
| `agent-test`  | agent-common (**core 없음**) | QA/테스트, 비밀키 접근 불가 |

### 설정 명령어

```bash
# 그룹 생성
sudo groupadd agent-common
sudo groupadd agent-core

# 계정 생성
sudo useradd -m -s /bin/bash -G agent-common,agent-core agent-admin
sudo useradd -m -s /bin/bash -G agent-common,agent-core agent-dev
sudo useradd -m -s /bin/bash -G agent-common            agent-test

# 비밀번호 설정
sudo passwd agent-admin
sudo passwd agent-dev
sudo passwd agent-test
```

### 확인 방법 및 결과

```bash
$ id agent-admin
uid=1001(agent-admin) groups=1001(agent-admin),1002(agent-common),1003(agent-core)

$ id agent-dev
uid=1002(agent-dev)   groups=1002(agent-dev),1002(agent-common),1003(agent-core)

$ id agent-test
uid=1003(agent-test)  groups=1003(agent-test),1002(agent-common)
# agent-core 없음 → api_keys 접근 불가
```

---

## 5. 디렉토리 구조 및 권한

### 구조

```
$AGENT_HOME  (/home/agent-admin/agent-app)
├── upload_files/   ← 공용 폴더: agent-common 그룹 읽기/쓰기 가능
├── api_keys/       ← 보안 폴더: agent-core 그룹만 접근 가능
├── bin/            ← 스크립트 보관 (monitor.sh 위치)
└── agent-app       ← 실행 파일

/var/log/agent-app/ ← 로그 폴더: agent-core 그룹만 접근 가능
```

### 설정 명령어

```bash
# 디렉토리 생성
sudo mkdir -p $AGENT_HOME/{upload_files,api_keys,bin}
sudo mkdir -p /var/log/agent-app

# 소유자 설정
sudo chown -R agent-admin:agent-core $AGENT_HOME
sudo chown    agent-admin:agent-core /var/log/agent-app

# 기본 권한 설정
sudo chmod 750 $AGENT_HOME
sudo chmod 770 $AGENT_HOME/upload_files
sudo chmod 750 $AGENT_HOME/api_keys
sudo chmod 770 /var/log/agent-app

# ACL 설정 (더 세밀한 권한 제어)
# upload_files: agent-common 그룹 읽기/쓰기 허용
sudo setfacl -m  g:agent-common:rwx $AGENT_HOME/upload_files
sudo setfacl -dm g:agent-common:rwx $AGENT_HOME/upload_files

# api_keys: agent-core 그룹만 접근
sudo setfacl -m  g:agent-core:rwx $AGENT_HOME/api_keys
sudo setfacl -dm g:agent-core:rwx $AGENT_HOME/api_keys

# /var/log/agent-app: agent-core 그룹만 접근
sudo setfacl -m  g:agent-core:rwx /var/log/agent-app
sudo setfacl -dm g:agent-core:rwx /var/log/agent-app
```

### 확인 방법 및 결과

```bash
$ ls -la $AGENT_HOME
drwxr-x---+ agent-admin agent-core  upload_files/   ← ACL 적용(+)
drwxr-x---+ agent-admin agent-core  api_keys/
drwxr-x---  agent-admin agent-core  bin/

$ getfacl $AGENT_HOME/upload_files
# owner: agent-admin
# group: agent-core
group:agent-common:rwx    ← common 그룹 접근 가능

$ getfacl $AGENT_HOME/api_keys
# owner: agent-admin
# group: agent-core
group:agent-core:rwx      ← core 그룹만 접근 가능
                          ← agent-test는 접근 불가
```

---

## 6. 환경 변수 및 키 파일

### 왜 하는가
- 경로를 코드 안에 직접 쓰면 (하드코딩), 경로가 바뀔 때마다 코드를 수정해야 합니다.
- 환경 변수로 빼두면 한 곳만 수정하면 모든 곳에 적용됩니다.

### 설정 명령어

```bash
# /etc/environment에 전역 등록 (모든 계정, 모든 세션에 적용)
sudo tee -a /etc/environment << 'EOF'
AGENT_HOME=/home/agent-admin/agent-app
AGENT_PORT=15034
AGENT_UPLOAD_DIR=/home/agent-admin/agent-app/upload_files
AGENT_KEY_PATH=/home/agent-admin/agent-app/api_keys/t_secret.key
AGENT_LOG_DIR=/var/log/agent-app
EOF

# agent-admin ~/.bashrc에도 추가 (로그인 세션 보장)
echo 'export AGENT_HOME=/home/agent-admin/agent-app'               >> ~/.bashrc
echo 'export AGENT_PORT=15034'                                     >> ~/.bashrc
echo 'export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files'            >> ~/.bashrc
echo 'export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key'    >> ~/.bashrc
echo 'export AGENT_LOG_DIR=/var/log/agent-app'                    >> ~/.bashrc
source ~/.bashrc

# 키 파일 생성
echo 'agent_api_key_test' | sudo tee $AGENT_HOME/api_keys/t_secret.key
sudo chmod 640 $AGENT_HOME/api_keys/t_secret.key
sudo chown agent-admin:agent-core $AGENT_HOME/api_keys/t_secret.key
```

### 확인 방법 및 결과

```bash
$ printenv | grep AGENT
AGENT_HOME=/home/agent-admin/agent-app
AGENT_PORT=15034
AGENT_UPLOAD_DIR=/home/agent-admin/agent-app/upload_files
AGENT_KEY_PATH=/home/agent-admin/agent-app/api_keys/t_secret.key
AGENT_LOG_DIR=/var/log/agent-app

$ cat $AGENT_HOME/api_keys/t_secret.key
agent_api_key_test

$ ls -la $AGENT_HOME/api_keys/
-rw-r----- agent-admin agent-core  t_secret.key
```

---

## 7. 애플리케이션 실행

### 실행 전 체크리스트

앱은 실행 시 다음 5가지를 순서대로 확인합니다.  
하나라도 실패하면 그 아래 단계는 전부 `[FAIL]` 처리됩니다.

| 단계 | 확인 내용 | 실패 원인 예시 |
|------|-----------|----------------|
| 1/5 | root가 아닌 일반 계정으로 실행 | root로 실행했을 때 |
| 2/5 | 환경 변수 5개 모두 설정됨 | `source ~/.bashrc` 안 했을 때 |
| 3/5 | `t_secret.key` 내용이 정확함 | 내용이 다르거나 파일이 없을 때 |
| 4/5 | 15034 포트가 비어있음 | 포트가 이미 사용 중일 때 |
| 5/5 | `/var/log/agent-app`에 쓰기 가능 | 권한이 없을 때 |

### 실행 명령어

```bash
# agent-admin 계정으로 전환
su - agent-admin

# 앱 실행 (백그라운드)
$AGENT_HOME/agent-app &

# 앱 종료할 때
Ctrl + C   또는   kill $(pgrep -f agent-app)
```

### 성공 시 출력

```
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
```

### 실패 시 출력 (예시: root로 실행한 경우)

```
>>> Starting Agent Boot Sequence...
[1/5] Checking User Account               [FAIL]
 >>> Error: Running as 'root' is forbidden. Use a service account.
[2/5] Verifying Environment Variables     [FAIL]
 >>> Skipped due to previous critical failure.
...
System Boot Failed. Process Terminated.
```

### 포트 LISTEN 확인

```bash
$ cat /proc/net/tcp | grep 3ABA
# 3ABA = 15034의 16진수 표현
00000000:3ABA  0A  ← 0A = LISTEN 상태 확인됨
```

---

## 8. monitor.sh 구현

### 파일 정보

| 항목 | 값 |
|------|----|
| 경로 | `$AGENT_HOME/bin/monitor.sh` |
| 소유자 | `agent-dev` |
| 그룹 | `agent-core` |
| 권한 | `750` (rwxr-x---) |
| 실행 계정 | `agent-admin` (agent-core 소속이므로 실행 가능) |

### 배포 명령어

```bash
cp monitor.sh $AGENT_HOME/bin/monitor.sh
sudo chown agent-dev:agent-core $AGENT_HOME/bin/monitor.sh
sudo chmod 750 $AGENT_HOME/bin/monitor.sh

# 확인
$ ls -la $AGENT_HOME/bin/monitor.sh
-rwxr-x--- agent-dev agent-core monitor.sh
```

### 동작 흐름

```
monitor.sh 실행
    │
    ├─ [1단계] 프로세스 확인 ── 없으면 → [ERROR] + exit 1 (종료)
    │
    ├─ [2단계] 포트 확인 ────── 없으면 → [ERROR] + exit 1 (종료)
    │
    ├─ [3단계] 방화벽 확인 ──── 꺼져있으면 → [WARNING] (종료 안 함)
    │
    ├─ [4단계] 자원 수집
    │           ├─ CPU 사용률 (%)
    │           ├─ 메모리 사용률 (%)
    │           └─ 디스크 사용률 (루트 파티션, %)
    │
    ├─ [5단계] 임계값 초과 경고 (종료 안 함)
    │           ├─ CPU > 20%   → [WARNING]
    │           ├─ MEM > 10%   → [WARNING]
    │           └─ DISK > 80%  → [WARNING]
    │
    ├─ [6단계] 로그 기록
    │           └─ /var/log/agent-app/monitor.log 에 한 줄 추가
    │
    └─ [7단계] 로그 파일 관리
                └─ 10MB 초과 시 자동 rotate (최대 10개 보관)
```

### 로그 출력 형식

```
[YYYY-MM-DD HH:MM:SS] PID:숫자 CPU:숫자% MEM:숫자% DISK_USED:숫자%
```

**실제 예시:**

```
[2025-06-01 14:23:00] PID:597 CPU:10.0% MEM:6.3% DISK_USED:47%
[2025-06-01 14:24:00] PID:597 CPU:35.2% MEM:6.4% DISK_USED:47%
[2025-06-01 14:24:00] [WARNING] CPU 사용률 높음: 35.2% (기준: 20%)
[2025-06-01 14:25:00] PID:597 CPU:8.1%  MEM:6.3% DISK_USED:47%
```

### 수동 실행 및 결과 확인

```bash
# agent-admin으로 수동 실행
sudo -u agent-admin $AGENT_HOME/bin/monitor.sh

# 로그 실시간 확인
tail -f /var/log/agent-app/monitor.log
```

---

## 9. crontab 자동 실행 등록

### 왜 하는가
- 사람이 직접 실행하면 깜빡하거나 자리를 비울 수 있습니다.
- crontab으로 등록하면 서버가 알아서 매분마다 자동으로 실행합니다.

### 등록 명령어

```bash
# agent-admin의 crontab 편집
sudo -u agent-admin crontab -e

# 아래 한 줄 추가 (매분 실행)
* * * * * /home/agent-admin/agent-app/bin/monitor.sh
```

**cron 표현식 설명:**

```
* * * * *
│ │ │ │ └── 요일 (0=일요일)
│ │ │ └──── 월
│ │ └────── 일
│ └──────── 시
└────────── 분
* = "모든" 이라는 뜻 → 매분 매시 매일 실행
```

### 확인 방법 및 결과

```bash
# 등록 확인
$ sudo -u agent-admin crontab -l
* * * * * /home/agent-admin/agent-app/bin/monitor.sh

# 1~2분 후 로그 자동 누적 확인
$ tail -5 /var/log/agent-app/monitor.log
[2025-06-01 14:27:00] PID:597 CPU:9.8%  MEM:8.3% DISK_USED:47%
[2025-06-01 14:28:00] PID:597 CPU:10.2% MEM:8.5% DISK_USED:47%
[2025-06-01 14:29:00] PID:597 CPU:11.1% MEM:8.4% DISK_USED:47%
# 매분 한 줄씩 자동으로 추가되는 것 확인됨
```

---

## 10. 로그 파일 관리

### 방식: 스크립트 자체 rotate

`logrotate` 없이 `monitor.sh` 내부에 rotate 로직을 직접 구현했습니다.

| 항목 | 설정값 |
|------|--------|
| 최대 파일 크기 | 10MB |
| 보관 파일 수 | 최대 10개 |
| 파일 이름 형식 | `monitor.log`, `monitor.log.1` ~ `monitor.log.10` |
| 초과분 처리 | gzip 압축 후 삭제 |

### rotate 동작 예시

```
[10MB 초과 감지]

rotate 전                rotate 후
─────────────────        ─────────────────────────
monitor.log (11MB)  →   monitor.log       (새 빈 파일)
monitor.log.1       →   monitor.log.1     (방금 전 로그)
monitor.log.2       →   monitor.log.2
...                 →   ...
monitor.log.9       →   monitor.log.9.gz  (압축됨)
```

### (선택) logrotate 연동

```bash
sudo tee /etc/logrotate.d/agent-app << 'EOF'
/var/log/agent-app/monitor.log {
    size 10M
    rotate 10
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

---

## 11. 증거 자료 체크리스트

| # | 확인 항목 | 상태 | 확인 명령어 |
|---|-----------|:----:|-------------|
| 1 | SSH 포트 20022 변경 | ✅ | `grep '^Port' /etc/ssh/sshd_config` |
| 2 | Root 원격 로그인 차단 | ✅ | `grep 'PermitRootLogin' /etc/ssh/sshd_config` |
| 3 | UFW 활성화 확인 | ✅ | `sudo ufw status verbose` |
| 4 | 20022/tcp, 15034/tcp만 허용 | ✅ | `sudo ufw status verbose` |
| 5 | 계정 3개 생성 확인 | ✅ | `id agent-admin && id agent-dev && id agent-test` |
| 6 | 그룹 2개 생성 확인 | ✅ | `cat /etc/group \| grep agent` |
| 7 | 디렉토리 구조 확인 | ✅ | `ls -la $AGENT_HOME` |
| 8 | ACL 권한 확인 | ✅ | `getfacl $AGENT_HOME/upload_files` |
| 9 | 환경 변수 설정 확인 | ✅ | `printenv \| grep AGENT` |
| 10 | 키 파일 내용 확인 | ✅ | `cat $AGENT_HOME/api_keys/t_secret.key` |
| 11 | Boot Sequence 5단계 [OK] | ✅ | `$AGENT_HOME/agent-app` 실행 출력 |
| 12 | Agent READY 출력 확인 | ✅ | 앱 실행 로그 |
| 13 | 0.0.0.0:15034 LISTEN 확인 | ✅ | `grep 3ABA /proc/net/tcp` |
| 14 | monitor.sh 권한 확인 | ✅ | `ls -la $AGENT_HOME/bin/monitor.sh` |
| 15 | monitor.sh 수동 실행 확인 | ✅ | `sudo -u agent-admin $AGENT_HOME/bin/monitor.sh` |
| 16 | monitor.log 기록 확인 | ✅ | `tail /var/log/agent-app/monitor.log` |
| 17 | crontab 매분 등록 확인 | ✅ | `sudo -u agent-admin crontab -l` |
| 18 | 1분 후 로그 자동 누적 확인 | ✅ | `watch -n 30 tail /var/log/agent-app/monitor.log` |

---

## 🔧 트러블슈팅

### 앱 실행 시 `[FAIL]` 이 나올 때

```bash
# root로 실행하면 1단계에서 바로 실패
# → 반드시 agent-admin 계정으로 실행
su - agent-admin

# 환경변수가 없을 때 2단계 실패
# → bashrc 다시 로드
source ~/.bashrc

# 키 파일 내용이 틀릴 때 3단계 실패
# → 내용 확인 (공백/줄바꿈 주의)
cat $AGENT_HOME/api_keys/t_secret.key
```

### monitor.sh 실행 시 에러가 날 때

```bash
# 권한 에러: 로그 파일을 못 쓸 때
sudo chmod 770 /var/log/agent-app
sudo chown agent-admin:agent-core /var/log/agent-app

# 앱이 꺼진 상태에서 실행하면
# [ERROR] 프로세스 'agent-app' 가 실행 중이 아닙니다.
# → 앱 먼저 실행 후 monitor.sh 실행
```

---

*Ubuntu 22.04 LTS 환경 기준으로 작성되었습니다.*
