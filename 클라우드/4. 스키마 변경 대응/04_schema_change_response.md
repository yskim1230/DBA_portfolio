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

- [x] 변경 신청서 기반 변경 범위/근거 정리
- [x] EXERD 변경내역(DDL) 파일로 이력화
- [x] 일일 업무일지에 이슈 배경/판단/대응 기록
