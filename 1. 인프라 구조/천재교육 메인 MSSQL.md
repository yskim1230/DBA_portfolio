# 천재교육 메인 DB MSSQL 스키마(다수 DB) 서비스 파악 요약

## 1) 결론: “단일 DB”가 아니라 **다수 DB로 분리된 대규모 교육 플랫폼** 구조

본 MSSQL 서버는 **약 79개 DB(정의서 시트 기준)**로 분리 운영되고 있으며, DB별로 기능이 강하게 분화되어 있습니다(회원/SSO/메시징/로그/서비스별 도메인).
또한 전체 규모는 **DB별 테이블 수 합산 기준 약 8,000여개 테이블 / 90,000여개 컬럼** 입니다.

---

## 2) 이 서버의 핵심 작동 구조

**회원/인증(SSO/Pass)–서비스별 도메인 DB(학습/콘텐츠/커뮤니티/커머스)–메시징(SMS)–감사/운영 로그(Log)** 로 기능이 분리되어 있고,
서비스 트랜잭션은 “각 도메인 DB에서 발생”하되 **공통 기능(SSO, 회원, 메시징, 로그)이 중앙 축처럼 재사용**되는 구조입니다.

---

## 3) DB 군(群)별 역할 분해 (운영 관점의 “도메인 맵”)

아래는 **DB 이름/테이블명/테이블 설명(정의서)**에서 가장 일관되게 드러나는 경계 기준으로 묶은 것입니다.

### A. 회원/인증/권한 (Identity, SSO)

* **Chunjae_SSO**

  * 로그인, 인증 토큰/세션, 동의/정책, 연동 계정 등 “접근 제어” 중심의 테이블이 배치된 형태
* **Chunjae_Pass**

  * 회원 기본정보/연령대(예: 14세 미만) 분리, 약관/동의, 계정 상태 등 “회원 마스터”의 성격이 강함

**운영 포인트**

* 개인정보(PII)와 인증 정보가 모이는 영역이므로: 접근 통제, 감사 로그, 마스킹/암호화, 권한 분리, 계정 잠금 정책이 “최우선”.

### B. 서비스 코어(학습/콘텐츠/커뮤니티/업무)

* **TSherpa**

  * 회원(서비스 레벨), 커뮤니티(게시판/댓글/추천), 알림, 콘텐츠 소비/활동 기록 등 “서비스 기능”이 응집된 DB로 해석
* **Chunjae_Main**

  * 공지/게시판/운영성 테이블 + 일부 상거래성(주문/정산) 테이블이 혼재하는 형태(대형 포털/메인 서비스 성격)

### C. 메시징/외부 통신 (Outbound)

* **Chunjae_SMS** (+ “SMSAgentCJ*”류 DB들)

  * 발송 큐/템플릿/발송 이력/게이트웨이 상태 등 “메시지 파이프라인” 분리 운영 가능성이 큼

### D. 로그/감사/이력 (Audit / Evidence)

* **Chunjae_Log**

  * 회원/행위/발송 이력 등 운영 증빙성 테이블이 모이는 형태
* (추가로 DB명에 Log/Event가 포함된 다수 DB가 존재 → 기능별 로그 분산 저장 가능)

---

## 4) 대표 엔터티(핵심 테이블)로 보는 “서비스 기능 추정”

> 테이블 정의서에서 “설명/명명”이 강하게 암시하는 축만 뽑았습니다. (포트폴리오에서는 이 정도가 가장 설득력 있습니다: “전수 나열”보다 “핵심 엔터티 + 흐름”이 훨씬 강함)

### (1) 계정/회원/권한 계열

* SSO/Pass: 회원/로그인/동의/정책/연령 분리 등
* 서비스(TSherpa/Main): 서비스 내 프로필/권한/활동

**유추되는 흐름**

1. 가입/로그인: SSO/Pass
2. 서비스 이용: TSherpa/Main 등 도메인 DB
3. 행위 이력: Log DB 또는 서비스 로그 테이블

### (2) 커뮤니티/게시판 계열

* Main/TSherpa에 “게시판(공지/문의/댓글/추천)” 성격 테이블이 존재
* 운영상 “민원/공지/학습 Q&A” 트래픽을 DB 레벨에서 수용하는 구조 가능

### (3) 주문/정산(커머스성) 계열 (Chunjae_Main에서 주 관리대상)

* 주문 마스터/상세/결제/정산 등 전형적 명명 패턴
* 교육 서비스에서 흔한 “유료 콘텐츠/교재/서비스 결제” 도메인이 Main에 들어가 있거나, 최소한 “주문 허브”가 Main에 존재할 가능성

### (4) 메시징(SMS) 계열

* 발송 마스터/발송 이력/템플릿류 테이블 구조가 분리되어 있을 가능성
* 운영 포인트: 재발송/중복발송 방지키, 트랜잭션 일관성, 대량 발송 시 큐 관리

---

## 5) “즉시 리스크 체크리스트”(SQL Server)

1. **인증/회원(SSO/Pass) 영역**

   * PII 컬럼(주민/연락처/이메일/생년/아동 정보 등) 암호화/마스킹 여부
   * 계정 잠금/비밀번호 정책, 권한 분리(운영/개발/배치), 감사(Audit) 설정 여부

2. **로그/이력(Log) 영역**

   * 로그 테이블 보관 정책(RETENTION), 파티셔닝/아카이빙, 인덱스 전략(조회/감사 요구 대비)
   * “증적” 관점에서 무엇을 남기는지(접근/변경/다운로드/발송 등)

3. **메시징(SMS) 영역**

   * 발송 중복 방지 키, 실패 재시도 정책, 대량 발송 시 잠금/대기열 병목 가능성
   * 외부 게이트웨이 장애 시 “유실/중복” 리스크

4. **서비스 코어(TSherpa/Main)**

   * 게시판/주문/활동 로그 등 핫 테이블의 인덱스 사용 현황, 상위 대기(Wait) 유형, 잠금 경합
   * “DB 분리”가 되어 있어도, 공통 엔터티(회원/권한) Join이 잦으면 병목이 생길 수 있음

---

## 6) 분석 포인트

### (1) DB별 테이블/용량 Top (대략적인 핫스팟 찾기)

```sql
;WITH s AS (
  SELECT
    DB_NAME() AS db_name,
    t.name AS table_name,
    SUM(p.rows) AS row_count,
    SUM(a.total_pages) * 8 / 1024.0 AS total_mb
  FROM sys.tables t
  JOIN sys.indexes i ON t.object_id = i.object_id
  JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
  JOIN sys.allocation_units a ON p.partition_id = a.container_id
  WHERE i.index_id IN (0,1)
  GROUP BY t.name
)
SELECT TOP (50) *
FROM s
ORDER BY total_mb DESC;
```

### (2) 인덱스 사용/미사용 (운영 개선 포인트 직행)

```sql
SELECT TOP (200)
  OBJECT_SCHEMA_NAME(s.object_id) AS schema_name,
  OBJECT_NAME(s.object_id) AS table_name,
  i.name AS index_name,
  s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i
  ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.database_id = DB_ID()
ORDER BY (s.user_scans + s.user_lookups) DESC;
```

### (3) 상위 쿼리(Top CPU / Top Duration) 

```sql
SELECT TOP (50)
  qs.total_worker_time / 1000 AS total_cpu_ms,
  qs.total_elapsed_time / 1000 AS total_elapsed_ms,
  qs.execution_count,
  (qs.total_elapsed_time / qs.execution_count) / 1000 AS avg_elapsed_ms,
  SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
    ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
      ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS stmt_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_elapsed_time DESC;
```

### (4) 잠금/대기(Wait) 현황 – 장애 대응 포인트

```sql
SELECT TOP (50) wait_type, waiting_tasks_count, wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE '%SLEEP%'
ORDER BY wait_time_ms DESC;
```

### (5) FK 관계(조인 구조) 덤프

```sql
SELECT
  OBJECT_SCHEMA_NAME(fk.parent_object_id) AS parent_schema,
  OBJECT_NAME(fk.parent_object_id) AS parent_table,
  OBJECT_SCHEMA_NAME(fk.referenced_object_id) AS ref_schema,
  OBJECT_NAME(fk.referenced_object_id) AS ref_table,
  fk.name AS fk_name
FROM sys.foreign_keys fk
ORDER BY parent_table, ref_table;
```

### (6) 민감 컬럼(PII) 후보 스캔

```sql
SELECT
  TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE COLUMN_NAME LIKE '%phone%'
   OR COLUMN_NAME LIKE '%tel%'
   OR COLUMN_NAME LIKE '%email%'
   OR COLUMN_NAME LIKE '%addr%'
   OR COLUMN_NAME LIKE '%birth%'
   OR COLUMN_NAME LIKE '%name%'
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

---

## 7) 메인 DB 운영 중 관리 포인트

* 다수 DB로 분리된 대규모 MSSQL 환경(정의서 기준 79개 DB, 8,500+ 테이블)의 스키마를 도메인(SSO/회원/서비스/메시징/로그) 관점으로 재분해하여, 핵심 엔터티와 데이터 흐름을 1페이지 아키텍처 맵으로 정리하고 운영 기준(권한/감사/보관/인덱스)을 수립.
* SSO/회원(PII) 영역과 로그/메시징 영역을 중심으로 접근 통제·감사 증빙·보관 정책을 점검하고, 상위 쿼리/인덱스 사용 현황을 기반으로 병목 후보 테이블을 선별하여 개선 우선순위를 정의.
* 서비스 코어 DB(TSherpa/Main 등)와 공통 DB(SSO/Pass/Log/SMS)의 경계에서 발생 가능한 조인/트랜잭션 병목을 가정하고, DMVs 기반으로 Top Query/Wait/Index Usage 증빙을 확보하여 장애 대응 런북 초안을 작성.

---
