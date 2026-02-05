# SQL Server 2012 EOS 대응 인프라 전환 – SQL Server 2019 마이그레이션 및 MSCS Failover 테스트 결과

SQL Server 2012 EOS 대응으로 2019 마이그레이션 수행 후, MSCS 기반 Active/Standby Failover/Failback 전환 및 DML 검증 결과를 증빙 캡처 기반으로 정리

---

# 1. 개요 (Why)
- **배경**: SQL Server 2012 EOS로 인해 보안 패치/기술지원 공백 발생 → 운영 리스크 증가
- **목표**
  - SQL Server 2019로 **안전한 마이그레이션(데이터/권한/배치/운영 자동화 포함)**
  - 전환 과정에서 **이중화(대기/전환) 관점의 복구·전환 테스트** 수행
- **원칙**
  - 데이터 무결성/서비스 연속성 최우선
  - 전환 실패 시 **롤백 가능**한 절차로 구성

---

# 2. 범위 (Scope)
## 포함
- 사용자 DB: 다수 운영/개발/스테이징 DB 일괄 백업 및 2019 환경 복원
- 파일 레이아웃 재구성: Data/Log 분리 (예: `S:\SQL_DATA`, `F:\SQL_LOG`)
- DB 호환성 레벨 조정(2019 기준)
- 로그인/권한/분리 사용자(orphan user) 정리
- SQL Agent Job 이관

## 제외 / 별도 관리
- SSIS/SSRS/SSAS 구성(해당 시 별도 문서)
- 애플리케이션 연결문자열 변경(전환 당일 체크리스트로 관리)

---

# 3. 아키텍처(전환 구성) 요약
- **Source**: SQL Server 2012 (운영)
- **Target**: SQL Server 2019 (신규)
- **백업 저장소(공유)**: `\\192.168.170.11\Backup_Mng` (공유폴더)
- **데이터 파일 경로**
  - MDF: `S:\SQL_DATA\<DB>\...`
  - LDF: `F:\SQL_LOG\<DB>\...`

> 전환 방식은 “백업/복원 기반”으로 신규 2019 서버를 구성하고, 전환/장애 시나리오에서 복구 가능성을 검증하는 형태로 진행.

---

# 4. 작업 절차 (Runbook) – 스크립트 기반
> 아래 순서는 실제 수행 순서에 맞춰 “증빙 파일(스크립트)”과 1:1로 매칭되도록 구성했습니다.

## 4.1 사전 준비 – 백업 SQL 생성 및 공유 경로 연결
### (A) DB 백업 SQL 자동 생성 (Cursor)
- **증빙**: `0. DB 백업SQL문 생성.sql`
- **핵심**: 시스템 DB 제외 후, 모든 사용자 DB에 대해 백업 커맨드를 자동 생성
- **산출물**: `BACKUP DATABASE [DB] TO DISK='\\...\Backup_Mng\DB.bak' WITH INIT, COMPRESSION;`

```sql
-- 증빙: 0. DB 백업SQL문 생성.sql
-- master.sys.databases 를 순회하며 사용자 DB 백업문 자동 생성
```

### (B) 네트워크 공유 드라이브 접근(백업 경로 확보)

* **증빙**: `0. DB 백업SQL문 생성완료 및 네트워크 드라이브 연결.sql`
* **증빙(프로시저 버전)**: `0. DB 백업SQL문 생성완료 및 네트워크 드라이브 연결(SP).sql`


> `xp_cmdshell` 활성화는 보안 리스크가 크므로, “임시 활성화→작업 후 비활성화” 통제 또는 대체방식(Agent Proxy/Credential) 고려.


```sql
-- 증빙: 네트워크 드라이브 연결(SP).sql
-- xp_cmdshell 활성화 및 net use 실행 로직 포함
-- 포트폴리오 공개본에서는 계정/암호/도메인 정보 마스킹 처리함
```

**운영 통제**

* 작업 전/후 `xp_cmdshell` 설정값 증빙(캡처/쿼리)
* 작업 완료 후 `xp_cmdshell` 비활성화 및 접근권한 회수

---

## 4.2 Target(2019) DB 생성 – 파일 레이아웃/디렉토리 구조 준비

* **증빙**: `1. DB 생성.sql`
* **핵심**

  * DB별 MDF/LDF 위치를 분리하여 생성
  * 향후 운영 시 IO/백업/복구 관리 용이성 확보

```sql
-- 증빙: 1. DB 생성.sql
-- CREATE DATABASE ... ON (FILENAME='S:\SQL_DATA\...') LOG ON (FILENAME='F:\SQL_LOG\...')
```

---

## 4.3 DB 파일 사이징(초기 공간 확보)

* **증빙**: `2. DB 공간확보.sql`
* **핵심**

  * DB별 초기 SIZE를 사전에 설정하여 **자동증가(autogrowth) 난발로 인한 성능 저하** 리스크 감소
  * 예: `ALTER DATABASE <DB> MODIFY FILE(NAME = <LogicalName>, SIZE = <n>MB)`

```sql
-- 증빙: 2. DB 공간확보.sql
ALTER DATABASE Chunjae_Main MODIFY FILE(NAME = Chunjae_Main, SIZE = 80000MB);
ALTER DATABASE Chunjae_Log  MODIFY FILE(NAME = Chunjae_Log,  SIZE = 48600MB);
-- ... 다수 DB에 대해 일괄 적용
```

---

## 4.4 DB 복원(RESTORE) – MDF/LDF 경로 재매핑

* **증빙**: `3. DB 복구 쿼리.sql`
* **핵심**

  * 백업파일(`S:\Backup_Mng\*.bak`)을 기반으로 복원
  * `WITH MOVE`로 파일 경로를 신규 스토리지 구조에 맞게 재매핑
  * `WITH REPLACE`로 대상 DB overwrite 허용(전환/재수행 시 유용)

```sql
-- 증빙: 3. DB 복구 쿼리.sql (예시)
RESTORE DATABASE Chunjae_Main
FROM DISK='S:\Backup_Mng\Chunjae_Main.bak'
WITH
MOVE 'Chunjae_Main'     TO 'S:\SQL_DATA\Chunjae_Main\Chunjae_Main.mdf',
MOVE 'Chunjae_Main_log' TO 'F:\SQL_LOG\Chunjae_Main\Chunjae_Main_log.ldf',
REPLACE;
```

---

## 4.5 호환성 레벨(Compatibility Level) 조정

* **증빙**: `4. 호환성 수정.sql`
* **핵심**: SQL Server 2019 기준 **Compatibility Level = 150** 적용

```sql
-- 증빙: 4. 호환성 수정.sql (원본은 다수 DB에 대해 150 적용)
ALTER DATABASE Chunjae_Main SET COMPATIBILITY_LEVEL = 150;
-- ...

-- 개선: 시스템 DB 제외 + 사용자 DB만 일괄 적용
-- SELECT name FROM sys.databases WHERE database_id > 4;
```

---

## 4.6 로그인/계정 이관 (sp_help_revlogin 기반)

* **증빙**: `5. 계정 생성.sql`
* **핵심**

  * 기존 서버 로그인 정보를 **SID 포함**하여 재생성 → 사용자 매핑/권한 연속성 유지
  * 일부 계정은 `DENY CONNECT` 및 `DISABLE`로 잠금 처리(보안/운영 정책 반영)

```sql
-- 증빙: 5. 계정 생성.sql (예시)
CREATE LOGIN [app_user]
WITH PASSWORD = 0x02... HASHED,
SID = 0x...,
DEFAULT_DATABASE = [SomeDB],
CHECK_POLICY = ON,
CHECK_EXPIRATION = OFF;
```

---

## 4.7 분리된 사용자(Orphan User) 정리 – 로그인 재매핑/정리

* **증빙**: `6. 분리된 DB 사용자 연결.sql`
* **핵심**

  * DB 내부 사용자(principal)와 서버 로그인(sid) 불일치 시

    * 존재하면 `ALTER USER ... WITH LOGIN=...`
    * 없으면 `DROP USER ...` 후보로 식별

```sql
-- 증빙: 6. 분리된 DB 사용자 연결.sql
SELECT 
  CASE 
    WHEN sl.sid IS NULL THEN 'DROP USER ' + dp.name 
    ELSE 'ALTER USER ' + dp.name + ' WITH LOGIN = ' + dp.name 
  END
FROM sys.database_principals dp
LEFT JOIN sys.syslogins sl ON dp.sid = sl.sid
WHERE dp.authentication_type_desc = 'INSTANCE' 
  AND dp.name <> 'dbo';
```

---

## 4.8 SQL Agent Job 이관

* **증빙**: `7. Job Create.sql`
* **핵심**

  * Job/Step/Schedule/Owner 정보를 신규 서버 msdb에 재생성
  * 운영 자동화(배치/정기 작업) 연속성 확보

```sql
-- 증빙: 7. Job Create.sql
USE [msdb];
-- sp_add_job / sp_add_jobstep / sp_add_jobschedule / sp_add_jobserver ...
```

---

# 5. 이중화(전환) 테스트 – 검증 항목(체크리스트)


## 5.1 복구 관점 테스트

* [ ] **RESTORE VERIFYONLY** 수행 (백업 손상 여부)
* [ ] 복원 후 **DBCC CHECKDB** (무결성)
* [ ] 핵심 테이블 rowcount 샘플 비교(이관 전/후)
* [ ] 주요 프로시저/배치 실행 스모크 테스트(에러로그 확인)
* [ ] SQL Agent Job 실행 테스트(성공/실패 로그 캡처)
* [ ] 권한/로그인 연결 테스트(애플리케이션 계정 기준)

## 5.2 전환 관점 테스트

* [ ] 서비스 연결 문자열 전환 리허설(운영/스테이징 우선)
* [ ] 장애 가정: 신규 2019 서버 Down 시 복귀 절차(롤백)
* [ ] 전환 소요시간 측정(T-0, T+5, T+10… 타임라인)

---

# 6. 리스크 및 통제

## 6.1 xp_cmdshell 사용 리스크

* **리스크**: OS 명령 실행 권한 → 침해 사고 시 악용 가능
* **통제**

  * 작업 시간에만 임시 활성화, 작업 종료 후 비활성화
  * 실행 주체 제한(권한 최소화) 및 감사 로그 확보
  * 대체방안: SQL Agent Proxy / Credential 기반 접근 검토

## 6.2 Compatibility Level 변경 리스크

* **리스크**: 쿼리 옵티마이저 변경으로 실행계획 변화 가능
* **통제**

  * 단계적 적용(핵심 DB부터), 성능 모니터링 및 플랜 회귀 대비
  * 시스템 DB는 스크립트 대상에서 제외(개선안 병기)

## 6.3 로그인/권한 이관 리스크

* **리스크**: SID 불일치 시 사용자 매핑 깨짐, 서비스 장애
* **통제**

  * sp_help_revlogin 기반 SID 유지
  * orphan user 점검/정리 스크립트로 사전 검증

---

# 7. 산출물(증빙 파일) 목록

* `0. DB 백업SQL문 생성.sql` : 사용자 DB 백업 커맨드 자동 생성
* `0. DB 백업SQL문 생성완료 및 네트워크 드라이브 연결.sql` : 공유 경로 기반 백업 실행
* `0. DB 백업SQL문 생성완료 및 네트워크 드라이브 연결(SP).sql` : 프로시저 형태로 백업/공유연결 수행
* `1. DB 생성.sql` : Target(2019) DB 생성 및 파일 경로 분리
* `2. DB 공간확보.sql` : DB 파일 초기 사이징
* `3. DB 복구 쿼리.sql` : WITH MOVE 기반 복원 스크립트
* `4. 호환성 수정.sql` : Compatibility Level 150 적용
* `5. 계정 생성.sql` : 로그인/SID 포함 이관(sp_help_revlogin 결과)
* `6. 분리된 DB 사용자 연결.sql` : orphan user 진단 및 조치 커맨드 생성
* `7. Job Create.sql` : SQL Agent Job 이관 스크립트

---
