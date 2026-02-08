## DB-01 기본 계정 패스워드/정책 변경

### MSSQL (`verify_mssql.sql`)

```sql
/* DB-01 (MSSQL) 기본 계정(예: sa) 사용/정책 점검 */

-- 1) sa 계정 비활성/이름 변경 여부(조직 정책에 따라)
SELECT name, is_disabled, create_date, modify_date
FROM sys.sql_logins
WHERE name IN ('sa');

-- 2) SQL 로그인 정책 적용 여부(Windows policy 연동)
SELECT name, is_policy_checked, is_expiration_checked, is_disabled
FROM sys.sql_logins
ORDER BY name;

-- 3) 최근 로그인 실패/성공 감사는 서버 감사 정책에 따라 별도 증빙(로그/감사)로 확보 권장
```

### MySQL/MariaDB (`verify_mysql.sql`)

```sql
/* DB-01 (MySQL/MariaDB) 기본 계정(root 등) 정책/사용 점검 */

-- 1) root 계정의 원격 허용(host='%' 등) 여부 확인
SELECT user, host
FROM mysql.user
WHERE user IN ('root')
ORDER BY host;

-- 2) 인증 플러그인/패스워드 정책 관련 플러그인(존재 시)
SHOW PLUGINS;

-- 3) (MySQL) validate_password 설정(사용 시)
SHOW VARIABLES LIKE 'validate_password%';
```

### Non-SQL 증빙(필수)

* `root`/`sa` 계정 운영 정책 문서(사용 금지 or 최소화)
* 접근제어 솔루션에서 관리자 계정 접근 통제 화면 캡처(가능 시)

---

## DB-02 불필요 계정 제거/잠금

### MSSQL

```sql
/* DB-02 (MSSQL) 미사용/불필요 로그인 점검 */

-- 1) 비활성(disabled) 로그인 목록
SELECT name, is_disabled, create_date, modify_date
FROM sys.sql_logins
WHERE is_disabled = 1
ORDER BY name;

-- 2) 로그인별 마지막 로그인 흔적(정확한 last login은 별도 감사 필요)
-- 대체 지표: 로그인별 권한/역할 부여 현황을 최소화했는지 확인
SELECT sp.name AS login_name, rp.name AS server_role
FROM sys.server_role_members srm
JOIN sys.server_principals sp ON srm.member_principal_id = sp.principal_id
JOIN sys.server_principals rp ON srm.role_principal_id = rp.principal_id
ORDER BY sp.name, rp.name;
```

### MySQL/MariaDB

```sql
/* DB-02 (MySQL/MariaDB) 불필요 계정/호스트 조합 점검 */

-- 1) 사용자 계정 목록(특히 test/anonymous 계정 존재 여부)
SELECT user, host
FROM mysql.user
ORDER BY user, host;

-- 2) 익명 계정( user='' ) 존재 여부
SELECT user, host
FROM mysql.user
WHERE user = '';
```

### Non-SQL 증빙

* “계정 생성/폐기(잠금) 기준” 운영 정책 문서
* 조치 스크립트(`apply.sql`) + 결과 캡처

---

## DB-03 패스워드 사용기간/복잡도 정책

### MSSQL

```sql
/* DB-03 (MSSQL) 패스워드 정책/만료 강제 */

SELECT name,
       is_policy_checked AS policy_checked,
       is_expiration_checked AS expiration_checked
FROM sys.sql_logins
ORDER BY name;
```

### MySQL/MariaDB

```sql
/* DB-03 (MySQL) validate_password / (MariaDB) password 정책 플러그인 여부 */

SHOW VARIABLES LIKE 'validate_password%';   -- MySQL
SHOW VARIABLES LIKE 'default_password_lifetime'; -- MySQL

-- MariaDB는 버전에 따라 password 정책 플러그인/파라미터가 다름
SHOW PLUGINS;
```

### Non-SQL 증빙

* “기관 패스워드 정책” 문서 캡처(요구사항)
* (MSSQL) Windows 계정 정책 화면 캡처(엑셀 상세에 ‘윈도우 정책 적용’ 근거가 있어 이게 제일 강함)

---

## DB-04 DBA(관리자) 권한 최소화

### MSSQL

```sql
/* DB-04 (MSSQL) sysadmin 최소화 */

SELECT sp.name AS login_name
FROM sys.server_role_members srm
JOIN sys.server_principals sp ON srm.member_principal_id = sp.principal_id
JOIN sys.server_principals rp ON srm.role_principal_id = rp.principal_id
WHERE rp.name = 'sysadmin'
ORDER BY sp.name;
```

### MySQL/MariaDB

```sql
/* DB-04 (MySQL/MariaDB) SUPER/ALL PRIVILEGES 등 고권한 점검 */

-- MySQL 8.0은 role/privilege 뷰가 더 명확. 5.7은 SHOW GRANTS로 개별 확인이 현실적.
SELECT user, host
FROM mysql.user
ORDER BY user, host;

-- 특정 계정(예: @@@)의 권한 확인
-- SHOW GRANTS FOR '@@@'@'@@@';
```

### Non-SQL 증빙

* 관리자 권한 승인/부여 프로세스(요청-승인-반영 기록)

---

## DB-05 패스워드 재사용 제약

### MSSQL

```sql
/* DB-05 (MSSQL) 재사용 제한은 Windows 정책 증빙이 핵심 */
-- SQL Server 단독으로 "password history"를 직접 제공하진 않음.
-- 따라서 Windows Local Security Policy / GPO 캡처가 메인 증빙.

-- SQL 로그인 정책 강제 여부(보조 증빙)
SELECT name, is_policy_checked, is_expiration_checked
FROM sys.sql_logins;
```

### MySQL/MariaDB

```sql
/* DB-05 (MySQL) validate_password / password_history(버전 의존) */

SHOW VARIABLES LIKE 'validate_password%';
SHOW VARIABLES LIKE 'password_history';     -- MySQL 8.0 계열에서 주로 등장
```

### Non-SQL 증빙

* Windows/GPO의 “암호 기록 강제(password history)” 화면 캡처

---

## DB-06 사용자 계정 개별 부여(공용 계정 최소화)

### MSSQL

```sql
/* DB-06 (MSSQL) 로그인/사용자 매핑 및 공유 계정 여부 점검 */

-- 서버 로그인 목록
SELECT name, type_desc, is_disabled
FROM sys.server_principals
WHERE type_desc IN ('SQL_LOGIN','WINDOWS_LOGIN','WINDOWS_GROUP')
ORDER BY type_desc, name;
```

### MySQL/MariaDB

```sql
/* DB-06 (MySQL/MariaDB) 사용자 목록 및 공용계정 패턴 점검 */

SELECT user, host
FROM mysql.user
ORDER BY user, host;
```

### Non-SQL 증빙

* 공용계정 금지/예외 기준 문서 + 예외 승인 기록(있다면)

---

## DB-07 원격 접속 제한(방화벽/접근제어)

### MSSQL / MySQL / MariaDB (DB 쿼리는 보조)

```sql
/* DB-07은 DB 내부에서 '방화벽 정책'을 완전히 증빙 불가.
   DB 쿼리는 "리스닝/포트" 확인 보조 수준. */
```

### Non-SQL 증빙(이 항목은 이게 메인)

* 방화벽 정책(허용 IP/포트 1433, 3306 등) 캡처
* DB접근제어 솔루션 정책/로그 캡처(엑셀 상세 근거가 “방화벽 및 DB접근제어”임)

---

## DB-08 시스템 테이블 접근 제한(인가 사용자만)

### MSSQL

```sql
/* DB-08 (MSSQL) 메타데이터 접근 통제 */

-- 권장: VIEW ANY DEFINITION 권한 부여 여부 점검
SELECT pr.name AS principal_name, pe.permission_name, pe.state_desc
FROM sys.server_permissions pe
JOIN sys.server_principals pr ON pe.grantee_principal_id = pr.principal_id
WHERE pe.permission_name IN ('VIEW ANY DEFINITION')
ORDER BY principal_name;

-- 데이터베이스별 public 권한 점검(예시: 특정 DB에서 VIEW DEFINITION 등)
-- USE [@@@DBNAME];
-- SELECT * FROM sys.database_permissions WHERE grantee_principal_id = DATABASE_PRINCIPAL_ID('public');
```

### MySQL/MariaDB

```sql
/* DB-08 (MySQL/MariaDB) mysql.* 접근권한 최소화 */

-- mysql 스키마 권한 확인은 SHOW GRANTS 기반으로 계정별 점검 권장
-- SHOW GRANTS FOR '@@@'@'@@@';
```

### Non-SQL 증빙

* “DBA 외 메타데이터 조회 제한” 운영 정책 + 접근제어 솔루션 룰(가능 시)

---

## DB-09 Oracle Listener 패스워드 (대부분 N/A)

* MSSQL/MySQL/MariaDB 환경이면 **N/A 근거**가 증빙임.

### Non-SQL 증빙

* “Oracle 미사용” 진단 결과 캡처(엑셀 상세에 N/A 사유가 이미 있음)

---

## DB-10 불필요 ODBC/OLE-DB 제거

### MSSQL(보조)

```sql
/* DB-10은 OS 레벨(드라이버/DSN) 점검이 핵심.
   SQL로는 완전 증빙 불가. */
```

### Non-SQL 증빙(메인)

* Windows: ODBC Data Sources(64/32bit) 캡처
* 사용 중 드라이버만 남겼다는 목록(정리표)
* “업무상 필요” 예외(엑셀 상세에 ODBC 사용 근거가 있으니, 제거가 아니라 최소화/정당화 증빙도 가능)

---

## DB-11 로그인 실패 잠금 정책

### MSSQL

```sql
/* DB-11 (MSSQL) Windows 정책 기반 계정 잠금은 OS 증빙이 메인 */

-- SQL 로그인 정책 강제(보조)
SELECT name, is_policy_checked
FROM sys.sql_logins;

-- Windows 정책 캡처가 최우선 증빙:
-- Account lockout threshold / duration / reset counter
```

### MySQL/MariaDB

```sql
/* DB-11 (MySQL/MariaDB) 계정 잠금 기능은 버전/플러그인 의존 */

-- MySQL: 계정 잠금 컬럼(존재 시)
SELECT user, host, account_locked
FROM mysql.user
WHERE user <> ''
ORDER BY user, host;

-- MariaDB는 버전별 차이. 계정 잠금 정책이 OS/접근제어로 대체되는 경우 Non-SQL 증빙을 붙임.
```

### Non-SQL 증빙

* Windows 계정 잠금 정책 화면 캡처(엑셀 상세에 “임계값 3회/잠금 60분” 같은 수치가 있어 캡처가 결정타)

---

## DB-12 DB 계정 umask 022 이상 (Linux DBMS에 주로 적용)

### MySQL/MariaDB (OS 메인)

```sql
/* DB-12는 SQL로 증빙 불가. OS에서 mysql 계정 umask 확인이 핵심 */
```

### Non-SQL 증빙(메인)

* `su - mysql` 후 `umask` 결과 캡처
* systemd 서비스 파일/프로필에서 umask 설정 캡처

---

## DB-13 주요 설정파일/패스워드 파일 권한

### MSSQL(Windows)

```sql
/* DB-13 (MSSQL) DB 파일/설정파일 ACL은 OS 증빙이 메인 */
```

### MySQL/MariaDB

```sql
/* DB-13은 my.cnf, datadir, keyfile 등의 퍼미션/소유자 확인이 핵심 (OS 메인) */
```

### Non-SQL 증빙(메인)

* Windows: DB 데이터/로그 디렉토리 ACL에서 Everyone 없음 캡처(엑셀 상세 근거)
* Linux: `ls -al /etc/my.cnf /etc/mysql/* /var/lib/mysql ...` 캡처

---

## DB-14 Oracle Listener 로그/trace 변경권한 제한 (대부분 N/A)

* Oracle 미사용이면 N/A 근거로 처리.

---

## DB-15 Role/Public 설정 금지(공개 권한 최소화)

### MSSQL

```sql
/* DB-15 (MSSQL) public 역할 권한 과다 부여 점검 */

-- 서버 레벨 public 권한
SELECT pr.name AS grantee, pe.permission_name, pe.state_desc
FROM sys.server_permissions pe
JOIN sys.server_principals pr ON pe.grantee_principal_id = pr.principal_id
WHERE pr.name = 'public'
ORDER BY pe.permission_name;

-- DB 레벨 public 권한(각 DB에서 실행)
-- USE [@@@DBNAME];
-- SELECT dp.class_desc, dp.permission_name, dp.state_desc
-- FROM sys.database_permissions dp
-- WHERE dp.grantee_principal_id = DATABASE_PRINCIPAL_ID('public');
```

### MySQL/MariaDB

```sql
/* DB-15 (MySQL/MariaDB) '모두에게 열려있는' 권한은 존재 구조가 다름.
   핵심은 계정별 최소권한 + '%' 호스트 최소화 */
SELECT user, host
FROM mysql.user
ORDER BY user, host;
```

### Non-SQL 증빙

* Role 설계/권한 승인 프로세스 문서(있으면 강함)

---

## DB-16 OS_ROLES / REMOTE_OS_AUTHENTICATION / REMOTE_OS_ROLES = FALSE (Oracle 전용, 보통 N/A)

* MSSQL/MySQL/MariaDB는 N/A 처리

---

## DB-17 패스워드 확인함수/정책 적용

엑셀 상세에서 MSSQL은 “윈도우 정책 적용” 근거가 반복됨. 즉 **OS 정책 증빙이 핵심**.

### MSSQL

```sql
/* DB-17 (MSSQL) 패스워드 정책 강제(보조) */
SELECT name, is_policy_checked, is_expiration_checked
FROM sys.sql_logins;
```

### MySQL/MariaDB

```sql
/* DB-17 (MySQL) validate_password 플러그인/파라미터 */
SHOW VARIABLES LIKE 'validate_password%';
SHOW PLUGINS;
```

### Non-SQL 증빙

* Windows/GPO 정책 캡처(복잡도/길이/만료)

---

## DB-18 인가되지 않은 Object Owner 제한 (Oracle/일부 DBMS 전용, 대개 N/A)

* MSSQL은 N/A로 적혀있음(엑셀 상세).
* MySQL/MariaDB에서도 “Owner” 개념이 Oracle과 달라서 대부분 N/A/대체통제로 설명.

### 대체통제(증빙 아이디어)

* “스키마/오브젝트 생성 권한은 DBA/배포계정만” 정책
* 배포계정의 권한(DDL 권한) 증빙

---

## DB-19 GRANT OPTION 제한 (Oracle 중심, MSSQL은 N/A)

MySQL/MariaDB는 `WITH GRANT OPTION` 존재하므로 **실제로 점검 가능**.

### MySQL/MariaDB

```sql
/* DB-19 (MySQL/MariaDB) GRANT OPTION 부여 여부 점검 */

-- 계정별 권한 확인(대표 계정만이라도 증빙)
-- SHOW GRANTS FOR '@@@'@'@@@';

-- information_schema.user_privileges에서 is_grantable 확인(지원 시)
SELECT grantee, privilege_type, is_grantable
FROM information_schema.user_privileges
ORDER BY grantee, privilege_type;
```

### Non-SQL 증빙

* “권한 부여 시 GRANT OPTION 금지” 기준 문서

---

## DB-20 자원 제한 기능 TRUE (Oracle resource_limit 류, 대부분 N/A)

* MSSQL N/A, MySQL/MariaDB도 동일 개념이 다르므로 N/A/대체통제로 처리하는 경우 많음.

### 대체통제(추천)

* 커넥션 제한(max_connections), 계정별 리소스 제한(가능 시) 점검

```sql
SHOW VARIABLES LIKE 'max_connections';
-- MySQL 계정별 리소스 제한(사용 시)
-- SELECT user, host, max_questions, max_updates, max_connections, max_user_connections FROM mysql.user;
```

---

## DB-21 최신 보안패치/벤더 권고 적용

### MSSQL

```sql
/* DB-21 (MSSQL) 버전/패치 레벨 확인 */

SELECT @@VERSION AS sqlserver_version;

-- 빌드번호 기반으로 CU 적용 여부는 릴리즈 노트/내부 패치 기록(Non-SQL)로 증빙하는 게 정확함.
```

### MySQL/MariaDB

```sql
/* DB-21 (MySQL/MariaDB) 버전 확인 */
SELECT VERSION() AS version;

-- 패치/업그레이드 이력은 운영 변경관리 문서/릴리즈 노트(Non-SQL)로 증빙 권장
```

### Non-SQL 증빙(이 항목은 이게 메인)

* 패치 적용 변경관리 티켓/승인 기록
* 적용 전/후 버전 캡처
* 테스트 후 적용했다는 근거(릴리즈 검증 체크리스트)

---

## DB-22 접근/변경/삭제 감사기록 정책 적합 설정

엑셀 상세에서 “DB접근제어를 통해 접속 로그 저장”이 근거로 등장함 → **솔루션 로그가 메인**.

### MSSQL(보조: 자체 Audit 사용 시)

```sql
/* DB-22 (MSSQL) SQL Server Audit 사용 시 점검 (사용하는 경우에만) */

SELECT name, status_desc
FROM sys.server_audits;

SELECT name, is_state_enabled
FROM sys.server_audit_specifications;
```

### MySQL/MariaDB(보조)

```sql
/* DB-22 (MySQL/MariaDB) Audit 플러그인 사용 시 점검 (사용하는 경우에만) */
SHOW PLUGINS;
-- audit_log 플러그인 변수는 환경별 상이
```

### Non-SQL 증빙(메인)

* 접근제어 솔루션에서 “접속/쿼리/변경” 로그 보관 화면 캡처
* 로그 보관주기/권한통제 정책 문서

---

## DB-23 보안에 취약하지 않은 버전 사용(버전 적정성)

### 공통

```sql
SELECT VERSION();
```

### Non-SQL 증빙(메인)

* “지원되는 버전인지(EOL 여부)” 내부 기준표(벤더 지원 정책 캡처/사내 기준)
* 업그레이드 로드맵(취약으로 남아있으면 예외 승인 + 보완통제)

---

## DB-24 Audit Table 접근을 DBA로 제한

엑셀 상세에서 MSSQL은 N/A로 나오는 케이스도 있음(환경에 따라 감사테이블 자체가 없을 수 있음).
반대로 MySQL/MariaDB에서 Audit 로그를 테이블로 보관하면 **권한 통제 증빙 가능**.

### MSSQL(사용 시)

```sql
/* DB-24 (MSSQL) 감사 로그/테이블(또는 Audit 파일) 접근 통제는 OS/권한이 메인.
   DB 내에서 확인 가능한 경우만 점검. */
-- 예: 특정 감사 DB/스키마의 SELECT 권한이 DBA에만 있는지 확인
```

### MySQL/MariaDB

```sql
/* DB-24 (MySQL/MariaDB) 감사 테이블(있다면) 접근권한 확인 */
-- 예: audit 스키마 테이블 권한 확인
-- SHOW GRANTS FOR '@@@'@'@@@';
```

### Non-SQL 증빙(메인)

* 감사 로그 파일/테이블 위치
* 접근 권한(ACL) 캡처
* DBA 계정만 접근 가능하다는 근거(권한 화면)

---
