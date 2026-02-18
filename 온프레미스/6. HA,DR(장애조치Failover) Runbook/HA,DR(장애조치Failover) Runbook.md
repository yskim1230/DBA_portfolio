# HA / DR(재해 복구) / Failover / Runbook / 자동화 레퍼런스

## 1) 개요 

천재교육 서비스 DB를 대상으로 **고가용성(HA) 확보**, **DR(재해복구) 지침 준수**, **Failover(장애조치) 리허설**, **복구 Runbook 표준화**, **백업/복구 자동화 정책 정립**을 수행했다.
특히 DR 지침에 정의된 **목표복구시점(RPO) / 목표복구시간(RTO)** 기준을 근거로 복구 시나리오와 훈련/리허설 절차를 문서화하고, 실제 운영 환경에서 **장애조치 및 복구 시뮬레이션**을 수행했다. 

---

## 2) 추진 배경 

* 서비스 핵심 DB 장애 시 **다운타임 최소화** 및 **데이터 손실 최소화** 요구
* 내부 **IT 재해복구 지침(Version 3.5)** 기반으로 복구 목표(RPO/RTO) 준수 필요 
* 장애조치/복구는 “실행할 줄 안다”보다 **리허설 + 결과 문서화 + 절차 표준화**가 핵심 (감사/보안/운영 신뢰도)

---

## 3) 범위 
### 3.1 대상 영역

* **HA(이중화/Failover)**:

  * (사례) MSCS 기반 **Active/Standby** 구조 DB Failover 테스트(작업결과서 기반)
* **DR(복구 시나리오/훈련/측정)**:

  * DB 파일 손상 등 장애 시 **Backup + Log 기반 복구** 및 **시점복구(StopAt)** 절차
* **Runbook 표준화**:

  * MySQL/MariaDB **Master/Slave 기동 및 연동**, **MHA 모니터링 기동**
* **백업/복구 정책 및 자동화**:

  * mysqldump/xtrabackup 기반 **일 단위 Full 백업**, 기본 보관 **7일**, 정기 복구 테스트 수행(정책 문서 기반)

---

## 4) 목표 

### 4.1 DR 지침 준수 목표 (RPO/RTO)

재해복구 지침에서 등급별 목표 복구 기준을 근거로 복구 목표를 설정
* **핵심업무(1등급)**: 데이터 Loss < 1Day(24시간), RTO 2시간 이내 
* **일반업무(2등급)**: 데이터 Loss < 2Day(48시간), RTO 4시간 이내 


---

## 5) 산출물 (Deliverables) – 증빙 목록

> 파일명은 실제 첨부 증빙과 1:1 매칭

1. **IT 재해복구 지침(근거 문서)**

   * `GL_04_IT재해복구지침_v3.5.pdf`
   * RPO/RTO 및 모의훈련 측정 항목 근거 

2. **DB Failover(장애조치) 작업결과서**

   * `천재교육_DB_이중화 테스트_작업결과서_20250519.doc`

3. **DB 복구 절차서(공식 Runbook)**

   * `2025년_천재교육_데이터베이스_복구절차서.docx`

4. **DB 복구 시뮬레이션(리허설 시나리오/SQL 포함)**

   * `DB복구 시뮬레이션.docx`

5. **MySQL 기동 절차서(Runbook)**

   * `MySQL 기동 절차서.docx`

6. **MySQL/MariaDB 백업·복구 정책 및 시나리오(정책/절차/명령어)**

   * `(mysql)DB백업복구정책v0.1-2.xlsx`

---

## 6) HA(고가용성) – Failover(장애조치) 리허설

### 6.1 테스트 목적

* Active 노드 장애/전환 상황에서 **Standby로 서비스 역할 전환**이 가능한지 검증
* 전환 후 **DB 리소스(데이터/로그/DTC/Quorum 등) 정상 인계** 및 **DML 동작 검증**
* 복구 완료 후 **원복(Failback)** 절차까지 수행하여 운영 안정성 확보

### 6.2 테스트 구성(예시: MSCS Active/Standby)

* Active 노드: `cjdb01p (A)`
* Standby 노드: `cjdb02p (S)`
* 전환 검증 항목(작업결과서 기반):

  * 디스크/리소스 인계: `MSSQL`, `DB_Log`, `DTC`, `Quorum`
  * 전환 직후 DML 검증: `SELECT / INSERT / UPDATE / DELETE`

### 6.3 수행 절차(요약 Runbook)

> ※ 실제 수행 시간은 사용자가 작성: `[작성]`

1. (사전점검) Active/Standby 상태 확인

   * Active: `[작성]` / Standby: `[작성]`
2. (Failover) MSCS 그룹 전환 수행

   * `Active → Standby` 전환
3. (검증) Standby에서 Active 역할 수행 확인

   * 디스크 리소스 인계 확인
   * 애플리케이션/쿼리 접속 확인
   * DML 4종 테스트 수행
4. (Failback) 원복 전환

   * `Standby → Active`
5. (최종점검) 서비스/모니터링 정상 여부 확인

   * 장애 알람/이벤트 확인 `[작성]`

### 6.4 결과 정리(템플릿)

* 전환 성공 여부: `[작성]`
* 전환 소요 시간: `[작성]`
* 전환 후 데이터 정합성/서비스 영향: `[작성]`
* 이슈 및 조치: `[작성]`

---

## 7) DR(재해복구) – 복구 절차서 & 시뮬레이션

### 7.1 장애 시나리오(예시)

* 장애 원인: **DB 파일 손상으로 정상 작동 불가**
* 복구 목표: 장애 발생 직전까지 가능한 범위 내 **시점복구**(StopAt)

### 7.2 백업 체계(근거: 복구절차서/시뮬레이션 문서)

* **Full 백업**: 매주 일요일 03:00
* **트랜잭션 로그 백업**: 매일 03:00
* **활성 로그(ldf)**: 마지막 로그 백업 이후 ~ 장애 시점까지 구간 복구에 활용

### 7.3 복구 SQL Runbook (실무형, 바로 실행 가능한 형태)

> 아래 SQL은 `2025년_천재교육_데이터베이스_복구절차서.docx` / `DB복구 시뮬레이션.docx`에 포함된 절차를 기반으로 재정리

#### (1) 장애 DB 오프라인 및 손상 파일 보존

```sql
USE master;
ALTER DATABASE [DB명] SET OFFLINE WITH ROLLBACK IMMEDIATE;
-- 손상 .mdf/.ldf 파일은 별도 경로로 복사 보관(원인 분석/증빙 목적)
```

#### (2) Full Backup 복원 (NORECOVERY)

```sql
USE master;
RESTORE DATABASE [DB명]
FROM DISK = 'S:\Backup_Mng\Sunday_FullBackup.bak'
WITH
    MOVE '[DB명]'     TO 'S:\SQL_DATA\DB명.mdf',
    MOVE '[DB명]_log' TO 'F:\SQL_LOG\DB명.ldf',
    NORECOVERY;
```

#### (3) Log Backup 순차 복원 (NORECOVERY)

```sql
RESTORE LOG [DB명]
FROM DISK = 'S:\Backup_Mng\Monday_LogBackup.trn'
WITH NORECOVERY;

RESTORE LOG [DB명]
FROM DISK = 'S:\Backup_Mng\Tuesday_LogBackup.trn'
WITH NORECOVERY;
```

#### (4) 시점 복구 (STOPAT + RECOVERY)

```sql
RESTORE LOG [DB명]
FROM DISK = 'S:\Backup_Mng\Tuesday_LogBackup.trn'
WITH STOPAT = 'YYYY-MM-DDThh:mm:ss', RECOVERY;
-- STOPAT은 목표 복구 시점(예: 장애 직전)으로 입력
```

#### (5) 복구 후 검증

```sql
SELECT name, state_desc
FROM sys.databases
WHERE name = 'DB명';

SELECT TOP 10 *
FROM [DB명].[dbo].[Table명];

ALTER DATABASE [DB명] SET MULTI_USER;
```

### 7.4 복구 수행 기록(감사 대응용)

* 복구 시작 시각: `[작성]`
* 복구 종료 시각: `[작성]`
* 사용한 백업본(Full/Log) 목록: `[작성]`
* STOPAT 적용 시각: `[작성]`
* 검증 결과(테이블/서비스/로그): `[작성]`
* 이슈/리스크 및 개선안: `[작성]`

---

## 8) Runbook – MySQL/MariaDB 기동 & MHA 모니터링

`MySQL 기동 절차서.docx` 기반으로, **Master/Slave 기동 + 복제 설정 + MHA 모니터링 기동**을 운영 표준 절차로 정리했다.

### 8.1 Master DB Start

```bash
# (케이스1) systemd
systemctl start mysql
```

```bash
# (케이스2) mysql.server
cd /usr/local/mysql/support-files/
./mysql.server start
```

```sql
-- MySQL 접속 후 binary log 확인
SHOW MASTER STATUS;
```

### 8.2 Slave DB Start & Replication 설정

```sql
CHANGE MASTER TO
  MASTER_HOST='MasterDB IP',
  MASTER_USER='로그인ID',
  MASTER_PASSWORD='로그인PW',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='binary log파일명',
  MASTER_LOG_POS=154;

START SLAVE;

SHOW SLAVE STATUS\G;
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
```

### 8.3 MHA 모니터링 Start

```bash
cd /mysqlha/scripts/
./mha_start.sh

ps -ef | grep -i mha
```

---

## 9) 백업/복구 자동화 정책 (MySQL/MariaDB)

`(mysql)DB백업복구정책v0.1-2.xlsx` 기준으로 정책을 표준화했다.

### 9.1 백업 정책(요약)

* **일 단위 FULL 백업 수행**: `mysqldump` 또는 `xtrabackup` 기반
* **기본 보관 기간 7일**(별도 요청 없을 시)
* 설치형 DB는 **월/분기 단위 복구 테스트**로 복구 가능 여부 및 시점복구 가능성 점검
* RDS(관리형 DB)는 **시점복구 소요시간 파악 목적의 복구 테스트** 수행
* 테이블 DROP 요청은 원칙적으로 즉시 DROP 대신 **`테이블명_날짜`로 RENAME 후 후속 처리**(운영 안전장치)

### 9.2 EC2(설치형) 복구 시나리오(요약)

* xtrabackup 설치/구성 → 전체 백업 → 장애(데이터 파일 삭제 등) 유발 → copy-back 복구 → 기동/검증
* Dump 기반 전체 복구 절차 및 binlog 기반 시점복구(mysqlbinlog 변환 후 적용) 포함

### 9.3 RDS 복구 시나리오(요약)

* AZ 단일 장애 시 AZ Failover로 서비스 순단 처리
* 멀티 AZ 동시 장애 등 비상 시 Snapshot Restore 또는 Point-in-time Restore 수행

---
