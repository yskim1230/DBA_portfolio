# MariaDB/MySQL MHA 구성 및 운영 Runbook (VIP Failover 포함)

## 1) 목표 및 범위

* **목표**: MariaDB/MySQL Replication 환경에서 **MHA(Master High Availability)** 를 이용해 **자동 Failover** 및 **VIP(가상 IP) 이동**까지 포함한 운영 가능 구성 확립
* **범위**

  * 서버 간 SSH 무비번 통신 구성
  * MHA Manager/Node 설치 및 기본 디렉토리 구조 구성
  * `app1.cnf` 기반 클러스터 정의
  * Failover/Online Master Switch 시 **VIP 자동 이동 스크립트 연동**
  * 점검/기동/장애 테스트/복구(재기동) 런북 포함

    

---

## 2) 아키텍처 개요

### 2-1. 구성 요소

* **MHA Manager 서버**: `mhamgr`
* **DB 노드**: `db1`(Master 후보), `db2`(Master 후보)
* **VIP**: `vip` (Failover 시 신규 Master로 이동)

### 2-2. /etc/hosts 예시

```bash
# [모든 서버]
vi /etc/hosts

172.21.122.47  mhamgr
192.168.170.48 db1
192.168.170.49 db2
192.168.170.47 vip
```



---

## 3) 사전 준비 (OS/계정/권한)

### 3-1. OS 계정 생성 (mha)

* Manager 서버와 DB 서버에 `mha` 계정 생성

```bash
# [mhamgr]
useradd -m -s /bin/bash mha
passwd mha

# [db1, db2] (mysql 그룹 사용)
useradd -m -s /bin/bash -g mysql mha
passwd mha
```

* 계정 만료 정책 확인/조정

```bash
chage -l mha
chage -E -1 -I 0 -m 0 -M 99999 mha
```



### 3-2. SSH Key 기반 무비번 통신

* 패스워드 없이 통과

```bash
su - mha
ssh-keygen -t rsa

ssh-copy-id mha@192.168.170.48
ssh-copy-id mha@192.168.170.49

ssh mha@192.168.170.48
ssh mha@192.168.170.49
```



### 3-3. sudo 권한 (VIP 제어용)

* `ifconfig` 위치 확인 후 `visudo`에 NOPASSWD 설정

```bash
which ifconfig   # 예: /usr/sbin/ifconfig

visudo
# 아래 중 택1
mha ALL=(ALL) NOPASSWD:/usr/sbin/ifconfig
# 또는
mha ALL=(ALL) NOPASSWD:ALL
```



---

## 4) DB 계정 및 복제 계정

### 4-1. MHA 관리용 DB 유저 (예: dbmha)

```sql
CREATE USER 'dbmha'@'%' IDENTIFIED BY '<REDACTED>';
GRANT ALL PRIVILEGES ON *.* TO 'dbmha'@'%';
FLUSH PRIVILEGES;
```

### 4-2. Replication 유저 (예: repl_user)

```sql
CREATE USER 'repl_user'@'%' IDENTIFIED BY '<REDACTED>';
GRANT ALL PRIVILEGES ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;
```

```sql
-- 특정 호스트로 제한 (예시)
CREATE USER 'dbmha'@'192.168.170.48' IDENTIFIED BY '<REDACTED>';
CREATE USER 'dbmha'@'192.168.170.49' IDENTIFIED BY '<REDACTED>';
CREATE USER 'dbmha'@'172.21.122.47' IDENTIFIED BY '<REDACTED>';

GRANT ALL PRIVILEGES ON *.* TO 'dbmha'@'192.168.170.48';
GRANT ALL PRIVILEGES ON *.* TO 'dbmha'@'192.168.170.49';
GRANT ALL PRIVILEGES ON *.* TO 'dbmha'@'172.21.122.47';

DROP USER 'dbmha'@'%';
```

 

---

## 5) MHA 설치 (Manager/Node)

### 5-1. 선행 리포지토리 및 Perl 의존성

```bash
# [mhamgr]
dnf config-manager --set-enabled powertools
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

yum install perl-DBD-MySQL \
  perl-Config-Tiny \
  perl-Log-Dispatch \
  perl-Parallel-ForkManager \
  perl-Time-HiRes \
  perl-CPAN \
  perl-Module-Install

yum update
```


### 5-2. mha4mysql-node 설치 (모든 서버: mhamgr, db1, db2)

```bash
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58.tar.gz
tar -zxvf mha4mysql-node-0.58.tar.gz
cd mha4mysql-node-0.58
perl Makefile.PL
make; make install
```



### 5-3. mha4mysql-manager 설치 (mhamgr 서버)

```bash
wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58.tar.gz
tar -zxvf mha4mysql-manager-0.58.tar.gz
cd mha4mysql-manager-0.58
perl Makefile.PL
make; make install
```



---

## 6) 표준 디렉토리/파일 구조
```bash
# [mhamgr]
mkdir -p /mysqlha/scripts
mkdir -p /mysqlha/conf
mkdir -p /mysqlha/app1

# 필요 시 mysql 그룹 생성
groupadd mysql

chown -R mha:mysql /mysqlha
```

* 샘플(conf/scripts) 복사

```bash
# 샘플 위치는 설치 환경에 따라 상이
cp conf/* /mysqlha/conf
cp scripts/* /mysqlha/scripts
```



---

## 7) MHA 설정 파일 (app1.cnf) 예시

```ini
[server default]
user=dbmha
password=<REDACTED>
ssh_user=mha
repl_user=repl_user
repl_password=<REDACTED>

manager_workdir=/mysqlha/app1
manager_log=/mysqlha/app1/app1.log
remote_workdir=/mysqlha/app1

# SHOW VARIABLES LIKE 'log_bin_basename';
master_binlog_dir=/data/mariadb/mariadb-bin

secondary_check_script=/usr/local/bin/masterha_secondary_check \
  -s 192.168.170.48 -s 192.168.170.49 \
  --user=mha --master_host=192.168.170.48 --master_ip=192.168.170.48 --master_port=3306

master_ip_failover_script=/mysqlha/scripts/master_ip_failover
master_ip_online_change_script=/mysqlha/scripts/master_ip_online_change

[server1]
hostname=192.168.170.48
candidate_master=1

[server2]
hostname=192.168.170.49
candidate_master=1
```



---

## 8) VIP Failover 자동화 (핵심 차별점)

### 8-1. `master_ip_failover` 수정

* 샘플 스크립트의 “앱 유저 생성/카탈로그 업데이트” 같은 예제 코드를 **주석 처리**
* Failover 완료 시점에 **VIP 이동 스크립트 호출** 추가

```perl
system("/bin/bash /mysqlha/scripts/mha_change_vip.sh $new_master_ip");
```



### 8-2. `master_ip_online_change` 수정

* Online master switch 시에도 동일하게 VIP 이동 호출 추가

```perl
system("/bin/bash /mysqlha/scripts/mha_change_vip.sh $new_master_ip");
```



### 8-3. `mha_change_vip.sh` (VIP 이동 스크립트)


```bash
#!/bin/bash
NIC="ens192"

V_NEW_MASTER=$(cat /etc/hosts | grep $1 | awk '{print $2}')
V_EXIST_VIP_CHK=$(ping -c 1 -W 1 vip | grep "packet loss" | awk '{print $6}')
V_VIP_IP=$(cat /etc/hosts | grep vip | awk '{print $1}')

if [ "$V_EXIST_VIP_CHK" = "0%" ]; then
  echo "VIP IS Alive, VIP Relocate $V_NEW_MASTER "
  /bin/ssh vip /bin/sudo /sbin/ifconfig ${NIC}:0 down &
  ssh $V_NEW_MASTER /bin/sudo /sbin/ifconfig ${NIC}:0 $V_VIP_IP netmask 255.255.255.0
  ssh $V_NEW_MASTER /sbin/arping -c 5 -D -I ${NIC} -s $V_VIP_IP $V_VIP_IP

  VIP_NOHUP_PS=$(ps -ef | grep "ifconfig ${NIC}:0" | grep ssh | grep -v grep | awk '{print $2}')
  [ -n "$VIP_NOHUP_PS" ] && kill -9 $VIP_NOHUP_PS
else
  echo "VIP IS dead, VIP Relocate $V_NEW_MASTER "
  /bin/ssh $V_NEW_MASTER /bin/sudo /sbin/ifconfig ${NIC}:0 $V_VIP_IP netmask 255.255.255.0
  /bin/ssh $V_NEW_MASTER /sbin/arping -c 5 -D -I ${NIC} -s $V_VIP_IP $V_VIP_IP
fi
```

* 권한

```bash
chmod 755 /mysqlha/scripts/mha_change_vip.sh
```



---

## 9) 구동/점검 명령 (운영 체크리스트)

### 9-1. SSH 체크

```bash
su - mha
masterha_check_ssh --conf=/mysqlha/conf/app1.cnf
```

### 9-2. Replication 체크

```bash
masterha_check_repl --conf=/mysqlha/conf/app1.cnf
```

### 9-3. Manager 상태 확인

```bash
masterha_check_status --conf=/mysqlha/conf/app1.cnf
```

 

---

## 10) MHA Manager 기동 스크립트 (표준화)

```bash
# /mysqlha/scripts/mha_start.sh
nohup masterha_manager \
  --conf=/mysqlha/conf/app1.cnf \
  --last_failover_minute=1 &
```

* 실행 및 프로세스 확인

```bash
cd /mysqlha/scripts
./mha_start.sh

ps -ef | grep -i mha
```

* 로그 모니터링

```bash
tail -100f /mysqlha/app1/app1.log
```

 

---

## 11) 장애 테스트 시나리오 

### 11-1. 장애 유도 (예: db1 다운)

```bash
# [db1]
systemctl stop mysqld   # 또는 mariadb
```

### 11-2. Manager 로그로 Failover 성공 확인

* `Master failover to db2 ... completed successfully` 메시지 확인

```bash
tail -100f /mysqlha/app1/app1.log
```



---

## 12) 운영 런북: DB 재기동(리부팅) 절차 (서비스 중단 최소화)

### 12-1. 쇼핑몰 운영DB 재기동 절차(예시)

```text
0) Master 덤프 백업 수행
1) MHA 중지: masterha_stop --conf=/mysqlha/conf/app1.cnf
2) Slave → Master 순으로 DB 중지
3) 서버 재기동 후 Master → Slave 순으로 DB 기동
4) 슬로우로그 등 파라미터 확인
5) VIP 설정 후 MHA 기동(mha_start.sh)
6) show slave status로 복제 상태 확인
```


### 12-2. 문제은행 운영DB 재기동 절차(예시)

```text
0) Master 덤프/서버 백업 수행
1) MHA 중지
2) Slave → Master 순 DB 중지 (systemctl stop mariadb 등)
3) Master → Slave 순 DB 기동 및 start slave
4) VIP 설정 후 MHA 기동(mha_start.sh)
5) show slave status\G 로 복제 정상 확인
```



---

## 13) 트러블슈팅

### 13-1. `mysqlbinlog` not found

* 증상: `Can't exec "mysqlbinlog": 그런 파일이나 디렉터리가 없습니다`
* 조치: `mysqlbinlog` 심볼릭 링크 생성, PATH 보정

```bash
# 예: MariaDB 환경에서 mariadb-binlog를 mysqlbinlog로 링크
sudo ln -s /usr/bin/mariadb-binlog /usr/bin/mysqlbinlog

# PATH 보정
export PATH=/usr/bin:$PATH
echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

 

### 13-2. Replication delay 메시지 포맷 이슈(NodeUtil.pm)

* 증상: 특정 에러 메시지 포맷 때문에 체크 실패
* 조치: `NodeUtil.pm`의 메시지 포맷 수정(환경 의존)



### 13-3. `super_read_only` 관련 오류(환경 의존)

* 증상: MHA 스크립트에서 `@@global.super_read_only` 조회로 장애
* 조치: `SlaveUtil.pm` 내 쿼리를 환경에 맞게 수정(주의: 운영 반영 시 리스크 평가 필요)



---

## 부록) 관련 원본 메모/설정 파일

* SSH 구성/계정 생성
* USER/sudo/VIP 테스트, DB 유저 생성
* 패키지 설치/스크립트 수정/VIP 스크립트
* 점검/기동/장애테스트 명령
* app1.cnf 샘플
* 트러블슈팅 메모 
* 운영 재기동 런북(쇼핑몰/문제은행)

---
