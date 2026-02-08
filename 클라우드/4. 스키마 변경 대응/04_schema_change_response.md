# 스키마 변경 대응 (Keris 연동 컬럼 길이 변경 / EXERD 반영)

- 작성일: 2026-02-09
- 작성자: 김영석
- 범위: Keris(통합 인증 포털) 연동 과정에서 전달되는 ID 길이 증가에 따른 **컬럼 길이 확장** 및 ERD/메타 반영
- 근거 자료:  
  - `DA_DE_50.DB변경관리신청서_Keris_대응_컬럼길이변경_송기범.xlsx`  
  - `exerd 변경내역.xlsx`  
  - 일일 인프라 정리(2026-02-02 중심)

> **보안 안내**: 본 문서는 포트폴리오 공개/공유를 고려하여 내부 도메인, 계정, 비밀번호, 티켓/시스템 식별자 등 민감정보를 **마스킹**했습니다.  
> 실제 운영 문서에는 사내 보안 정책(시크릿 매니저/권한분리/접근통제)에 따라 원본 정보를 별도 보관합니다.


## 1. 배경

- AIDT 1년차(통합교육자료) 서비스는 Keris 인증 포털을 통해 로그인/토큰 발급 흐름이 연결됨.
- Keris 측 UUID/식별자 길이 확대로 인해, 기존 컬럼 길이로는 저장/비교 과정에서 오류 가능성이 발생.
- 대응 목표:
  1) **DB 컬럼 길이 확장**(운영 반영)  
  2) **EXERD/ERD 메타 반영**(변경 이력 관리)

## 2. 변경 요청 요약(신청서 기반)

- 변경구분: **컬럼길이변경**
- 신청일자: **2026-02-02**
- 신청자: **송기범**

### 2.1 변경 범위(컬럼/대상 테이블 수)

| 컬럼         | 변경                         |   대상 테이블 수 | NULL 허용   |
|:-------------|:-----------------------------|-----------------:|:------------|
| USR_ID       | VARCHAR(36) -> VARCHAR(64)   |               55 | NULL 허용   |
| KERIS_USR_ID | VARCHAR(36) -> VARCHAR(64)   |               13 |             |
| LOGIN_USR_ID | VARCHAR(100) -> VARCHAR(200) |                3 |             |

### 2.2 영향 테이블(발췌)

- `USR_ID` 변경 대상: **55개 테이블**  
  - 예: ai_cmt_usr_data, ai_fdbk, ai_lrn_pgrs_prof_v2, ai_lrnr_lv_v2, ai_obsv_recd_usr_data, ai_rcm_tssh_hst, ai_rcm_tssh_qtm_hst, ai_sbc_flt_usr_fle, ai_usrly_qtm_prof, ai_usrly_tc_lrn_ordn ...
- `KERIS_USR_ID` 변경 대상: **13개 테이블**  
  - 예: cm_auth_token, cm_cla_cp_log, cm_keris_dsbd_log, cm_keris_front_log, cm_token, cm_token_bak, cm_trm_lect_mpn, cm_trm_lect_mpn_del_hst, cm_usr, cm_usr_bak ...
- `LOGIN_USR_ID` 변경 대상: **3개 테이블**  
  - 예: cm_persona_usr, cm_token, cm_token_bak

## 3. 수행 내용 (DB 반영 + 메타 반영)

### 3.1 DB 반영(DDL 적용)

- `exerd 변경내역.xlsx` 기준 DDL:
  - 총 변경문: **74건**
  - `USR_ID` 포함: **71건**
  - `KERIS_USR_ID` 포함: **13건**
  - `LOGIN_USR_ID` 포함: **0건**
  - 적용 상태: 대부분 `O`(적용)

#### DDL 예시(발췌)

| 변경문                                                                                                   | 적용   | 비고   |
|:---------------------------------------------------------------------------------------------------------|:-------|:-------|
| ALTER TABLE <SCHEMA>.lw_sbc_lrn_lsn_ev MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';     | O      | nan    |
| ALTER TABLE <SCHEMA>.lw_sbc_lrn_acn_st MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';     | O      | nan    |
| ALTER TABLE <SCHEMA>.lw_ocr_bak1 MODIFY COLUMN USR_ID varchar(64) NULL COMMENT '사용자ID';               | O      | nan    |
| ALTER TABLE <SCHEMA>.tl_txb_wkb_tnte MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';       | O      | nan    |
| ALTER TABLE <SCHEMA>.tl_lu_lrnr_lv MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';         | O      | nan    |
| ALTER TABLE <SCHEMA>.my_wd_srh_hst MODIFY COLUMN USR_ID varchar(64) NULL COMMENT '사용자ID';             | O      | nan    |
| ALTER TABLE <SCHEMA>.my_usr_daily_voc MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';      | O      | nan    |
| ALTER TABLE <SCHEMA>.my_my_wdlst MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';           | O      | FK     |
| ALTER TABLE <SCHEMA>.my_lsn_memo MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';           | O      | FK     |
| ALTER TABLE <SCHEMA>.my_fln_hst MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';            | O      | FK     |
| ALTER TABLE <SCHEMA>.ea_ev_spp_ntn_rs MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID';      | O      | FK     |
| ALTER TABLE <SCHEMA>.ea_ev_spp_ntn_qtm_anw MODIFY COLUMN USR_ID varchar(64) NOT NULL COMMENT '사용자ID'; | O      | FK     |

> FK가 있는 테이블은 DDL 방식에 따라 Lock/제약조건 영향이 발생할 수 있어,  
> 사전 점검 → 단계적 적용 → 검증 절차로 수행하도록 런북화했습니다.

### 3.2 EXERD/ERD 메타 반영

- 변경 신청서의 “엔터티/속성/컬럼” 기준으로 ERD 도구(EXERD)에 반영
- 변경문(ALTER TABLE)을 이력으로 보관하여:
  - 개발/운영/스테이징 환경 간 일관성 체크
  - 향후 감사(ISMS 등) 시 변경 근거 증빙

## 4. 운영 반영 런북(권장 절차)

### 4.1 사전 점검

1) 컬럼 현재 길이 확인

```sql
select table_schema, table_name, column_name, column_type, is_nullable
from information_schema.columns
where column_name in ('USR_ID','KERIS_USR_ID','LOGIN_USR_ID')
order by table_schema, table_name, column_name;
```

2) FK/인덱스 영향 확인(특히 FK 코멘트가 있는 테이블)

```sql
select table_schema, table_name, constraint_name, referenced_table_name
from information_schema.referential_constraints
where constraint_schema = '<SCHEMA>';
```

### 4.2 DDL 적용 전략

- 1) **스테이징 선반영 → 운영 반영**(릴리즈/점검 창 확보)
- 2) 가능한 경우 `ALGORITHM=INPLACE`, `LOCK=NONE` 옵션을 고려(환경 지원 여부 확인)
- 3) 대량 테이블/핵심 트래픽 테이블은 **분할 적용(배치)** + 실시간 세션 모니터링(3초 수집) 병행

### 4.3 사후 검증

- 컬럼 타입 변경 확인 + 애플리케이션 로그인/토큰 흐름 회귀 테스트
- 장애 리스크:
  - Metadata Lock 장기 유지
  - DDL 중 복제 지연(복제 구성 시)
  - 인덱스 길이/Row size 이슈(대형 row/utf8mb4)

## 5. 산출물(증빙)

- [x] 변경 신청서 기반 변경 범위/근거 정리(본 문서)
- [x] EXERD 변경내역(DDL) 파일로 이력화
- [x] 일일 업무일지에 이슈 배경/판단/대응 기록(2026-02-02)

## Appendix. 일일 인프라 정리(발췌)

### 2026-01-29
  - 2년차 운영 lms -> 통합본이라 하자
  - 교육자료로 변경되면서 통합할 예정
  - 통합본 으로 전환해야 하는 시기가3월 1일
  - 2월 3주차까지 정리 작업
  - 스펙도 줄이고, 최소스펙으로 줄이는 작업이 필요하다.
  - 번외.문항통합플랫폼 서비스
  - 2년차 -> 1년차로 마이그레이션 작업이 예정
  - MYSQL 버전 업데이트 작업

### 2026-01-30
  - 오늘 목표
  - 1. 저작도구 2년차를 1년차로 통합하는 과정 확인
  - 2. lms 복원/복구/네이버 클라우드로 복원작업
  - 작업 스크립트
  - NCP Cloud DB(MySQL) 개발 DB 백업/복원 리허설 (ATS)
  - 목적
  - - 2년차 개발 DB(ATS) 논리 백업(mysqldump) 후 1년차 테스트 DB로 복원하고 정합성 검증 수행
  - 테스트용 NCP 자원(VPC,Subnet). 2월28일까지만 유효
  - VPC : lms-dev-vpc-01
  - Subnet : lms-dev-sub-prv-test-01|KR-1|Private

### 2026-02-02
  - ---
  - 2026-02-02 업무 파악 정리 (Cloud DBA 인수인계)
  - 1) 오늘 이슈 배경
  - * **AIDT 1년차(통합교육자료)** 서비스 로그인은 **Keris(통합 인증 포털)** 을 통해 유입됨.
  - * 금요일(요청서 기준)부터 Keris 측에서 **UUID/식별자(ID) 문자 수를 늘려 전달**하기 시작.
  - * 현업 문의 요지:
  - * “2025년도향 DB도 바뀌었는지?”
  - * “운영에도 반영되어야 한다.”
  - 2) 핵심 판단: DB가 관여될 가능성
  - * DB가 관여될 수 있는 대표 케이스:

### 2026-02-02
  - 2026-02-02 (월) 업무일지 — Cloud DBA 인수인계 / AIDT Keris 로그인 이슈 대응
  - 1) 오늘의 핵심 이슈
  - * **AIDT 1년차(통합교육자료)** 서비스 로그인은 **Keris(통합인증 포털)** 을 통해 유입됨.
  - * Keris 측 변경으로 **식별자(UUID/ID) 길이가 증가**하여 로그인 장애 가능성이 제기됨.
  - * 문의 요지:
  - * “금요일자부터 Keris 포털에서 UUID 문자 수가 늘어 들어오기 시작했다.”
  - * “2025년도향 DB도 바뀌었는지, 운영에도 반영이 필요하다.”
  - 2) 초기 분석 및 판단
  - * DB가 관여될 가능성은 “있음”.
  - * Keris ID가 사용자 매핑 키로 저장/조회되거나,

### 2026-02-03
  - 3일 업무내용
  - 1.자원최적화 엑셀 갱신
  - 2. exerd 갱신
  - 작업 : exerd 1년차도 갱신필요(통합ERD.exerd)
  - - 갱신대상 컬럼
  - 3. 저작도구(ats) 서비스 이관 (2년차->1년차) mysql 8.0 -> 8.4
  - --> 1년차에도 동일하게 생성예정
  - 1년차 DB 신규생성 후 작업
  - /*생성 작업내용*/
  - 1. DB 생성은 민호님한테 요청

### 2026-02-03
  - 아래는 네가 적은 “3일 업무내용”을 **그대로 살리되**, 바로 **증빙 가능한 이관 런북(운영 문서)** 형태로 재구성한 마크다운이야.
  - ---
  - [RUNBOOK] 클라우드 DBA 일일업무 (Day 3) — 자원/ERD 갱신 + ATS DB 이관 (MySQL 8.0 → 8.4)
  - 0. 문서 메타
  - * 작성일: `YYYY-MM-DD`
  - * 작성자: `김영석`
  - * 대상 서비스: **저작도구(ATS)**
  - * 작업 유형: **(1) 자원 최적화 엑셀 갱신 (2) EXERD 갱신 (3) DB 이관**
  - * 작업 범위: **2년차 → 1년차 DB 이관**, **MySQL 8.0 → 8.4**
  - * 작업 목표

### 2026-02-04
  - 2026-02-04 클라우드 DBA 업무요약 (NCP 자원 회수/통합 이관 사전 검증)
  - 1. 오늘 목표
  - * **2년차 AIDT 교과목 클라우드 자원 회수 및 1년차로 통합**을 위한 사전 작업 수행
  - * 2년차 산재 DB(교과서 단위)를 신규 통합 DB 네이밍 규칙으로 재구성하기 전, **테이블 구조(스키마)만 먼저 덤프/복원**하여
  - * 계정/권한/접속
  - * 덤프/복원 절차
  - * 스키마 충돌 가능성
  - 2. 작업 1) 로그 수집 방식/로그 파일 내역 확인
  - * 로그 저장 경로 확인:
  - * `/backup/DB_logs/2026stg_log/`

### 2026-02-05
  - 2026-02-05 클라우드 DBA 업무내역 (1안: 상시 통합 PoC)
  - 0. 배경/목표
  - * 기준안: **1년차 DB 통합안 1안(상시 통합)**
  - * 목표: 2년차에 분산된 교과목 DB를 **1년차 신규 통합 DB 네이밍 규칙(예: `lms_lrm_eng01`)으로 상시 운영 가능한 형태로 재구성**하기 위한 사전 검증
  - * 오늘 범위: **실데이터가 아닌 “스키마(구조) 덤프 → 통합 테스트 DB에 복구” 중심의 PoC**
  - ---
  - 1. 수행 작업 요약
  - 1) 교과목군별(영어/수학) 스키마 덤프 자동화 스크립트 정리
  - * 백업 경로 생성 및 표준화
  - * 덤프 종료 문구(`Dump completed on`) 기반으로 **덤프 “완결성” 검증 로직 포함**

