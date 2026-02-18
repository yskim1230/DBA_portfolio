# 자원 최적화 작업 (AIDT DB 통합안 PoC / 표준 네이밍 적용)

- 작성일: 2026-02-09
- 작성자: 김영석
- 범위: **2년차 분산 교과목 DB → 1년차 통합 DB 구조**로 전환하기 위한 사전 검증(PoC)
- 근거 자료: `AIDT_통합안_v0.1.xlsx` + 일일 인프라 정리(1/29~2/5)

> **보안 안내**: 본 문서는 포트폴리오 공개/공유를 고려하여 내부 도메인, 계정, 비밀번호, 티켓/시스템 식별자 등 민감정보를 **마스킹**했습니다.  
> 실제 운영 문서에는 사내 보안 정책(시크릿 매니저/권한분리/접근통제)에 따라 원본 정보를 별도 보관합니다.


## 1. 배경

- 교과서 단위로 **DB가 산재(분산)** 되어 운영/관리 비용이 증가.
- 구독제(정액)에서 종량제로 전환되는 정책 변화로, **자원 회수/스펙다운/통합**이 필요.
- 목표 전환 시점: **2026-03-01** (통합본 운영 전환)

## 2. 목표

1. 분산 DB를 통합 DB로 재구성할 때 발생할 수 있는 리스크를 **사전에 제거**
2. 통합 DB 네이밍 규칙을 적용(예: `lms_lrm_eng01`)하여 운영 편의성 확보
3. 실데이터 이전 전에 **스키마(구조) 덤프/복원** 중심으로 “가능/불가능”을 빠르게 판별

## 3. 통합안 구조 요약 (현재 → 변경)

### 3.1 현재(분산)

- 교과목/학년/저자 단위로 DB 인스턴스가 분산되어 있고,
  - 학습창(LRW), 학습관리(LRM), CMS가 분리/조합된 형태
- 운영 관점에서는 “교과목 단위 이슈”가 “인프라 이슈(스케일/백업/장애)”로 확대되는 구조

### 3.2 변경(통합/표준화)

- 교과목군별로 DB 네이밍을 표준화하고, 운영 정책(백업/권한/모니터링)을 **일관 적용**
- 예시: 영어 `eng01 ~ eng12` 등 suffix 기준으로 정렬/관리

## 4. PoC 접근 방식 (스키마만으로 빠른 검증)

### 4.1 PoC 전략

- **스키마 덤프 → 통합 테스트 DB에 복원**:
  - 계정/권한/접속 가능 여부
  - 덤프/복원 절차
  - 테이블/컬럼 충돌 가능성(명명 규칙/대소문자/charset/collation)
- 실데이터는 제외: 데이터 이관은 비용/시간/리스크가 크므로, 1차는 “구조 검증”에 집중

### 4.2 검증 포인트(실무 체크리스트)

- `lower_case_table_names` 사전 세팅(초기화 필요하므로 **DB 생성 직후 선행**)
- `character_set_server=utf8mb4`, `collation_server=utf8mb4_general_ci` 등 표준 설정
- `long_query_time`(예: 0.1s) 기준으로 튜닝 후보군을 조기 탐지
- `ONLY_FULL_GROUP_BY` 등 SQL mode 정책 통일
- 백업 보관일 정책(운영 7일 / 비운영 3일) 및 로그/백업 디스크 Full 리스크 관리

## 5. 통합안 매핑

- 통합안  수: **52**
- 교과서(변경안) 기준 고유 항목 수(대략): **13**

| 학교급(현재)   | 교과서(현재)   | 저자(현재)   | 접속정보(현재)       | DB명(현재)        | 교과서(변경)   | 접속정보(변경)                   | DB명(변경)                                       |
|:---------------|:---------------|:-------------|:---------------------|:------------------|:---------------|:---------------------------------|:-------------------------------------------------|
| 초등           | 영어 3         | 이동환       | <CDB_PRIVATE_DOMAIN> | lms_lrm / lms_cms | 영어 3         | 영어통합…<CDB_DOMAIN> | lms_lrm_eng01 / lms_cms_eng01 / lms_lrw_eng01    |
| 초등           | 영어 4         | 이동환       | <CDB_PRIVATE_DOMAIN> |                   | 영어 4         |                                  | lms_lrm_eng02 / lms_cms_eng02 / lms_lrw_eng02    |
| 초등           | 영어 3         | 김태은       | <CDB_PRIVATE_DOMAIN> |                   | 영어 3         |                                  | lms_lrm_eng03 / lms_cms_eng03 / lms_lrw_eng03    |
| 초등           | 영어 4         | 김태은       | <CDB_PRIVATE_DOMAIN> |                   | 영어 4         |                                  | lms_lrm_eng04 / lms_cms_eng04 / lms_lrw_eng04    |
| 초등           | 영어 3         | 함순애       | <CDB_PRIVATE_DOMAIN> |                   | 영어 3         |                                  | lms_lrm_eng05 / lms_cms_eng05 / lms_lrw_eng05    |
| 초등           | 영어 4         | 함순애       | <CDB_PRIVATE_DOMAIN> |                   | 영어 4         |                                  | lms_lrm_eng06 / lms_cms_eng06 / lms_lrw_eng06    |
| 중학           | 영어           | 이상기       | <CDB_PRIVATE_DOMAIN> |                   | 영어           |                                  | lms_lrm_eng07 / lms_cms_eng07 / lms_lrw_eng07    |
| 중학           | 영어           | 소영순       | <CDB_PRIVATE_DOMAIN> |                   | 영어           |                                  | lms_lrm_eng08 / lms_cms_eng08 / lms_lrw_eng08    |
| 고등           | 공통영어 1     | 강상구       | <CDB_PRIVATE_DOMAIN> |                   | 공통영어 1     |                                  | lms_lrm_eng09 / lms_cms_eng09 / lms_lrw_eng09    |
| 고등           | 공통영어 2     | 강상구       | <CDB_PRIVATE_DOMAIN> |                   | 공통영어 2     |                                  | lms_lrm_eng10 / lms_cms_eng10 / lms_lrw_eng10    |
| 고등           | 공통영어 1     | 조수경       | <CDB_PRIVATE_DOMAIN> |                   | 공통영어 1     |                                  | lms_lrm_eng11 / lms_cms_eng11 / lms_lrw_eng11    |
| 고등           | 공통영어 2     | 조수경       | <CDB_PRIVATE_DOMAIN> |                   | 공통영어 2     |                                  | lms_lrm_eng12 / lms_cms_eng12 / lms_lrw_eng12    |
| 초등           | 수학 3-1       | 박만구       | <CDB_PRIVATE_DOMAIN> |                   | 수학 3-1       | 수학통합…<CDB_DOMAIN> | lms_lrm_math01 / lms_cms_math01 / lms_lrw_math01 |
| 초등           | 수학 3-2       | 박만구       | <CDB_PRIVATE_DOMAIN> |                   | 수학 3-2       |                                  | lms_lrm_math02 / lms_cms_math02 / lms_lrw_math02 |
| 초등           | 수학 4-1       | 박만구       | <CDB_PRIVATE_DOMAIN> |                   | 수학 4-1       |                                  | lms_lrm_math03 / lms_cms_math03 / lms_lrw_math03 |
| 초등           | 수학 4-2       | 박만구       | <CDB_PRIVATE_DOMAIN> |                   | 수학 4-2       |                                  | lms_lrm_math04 / lms_cms_math04 / lms_lrw_math04 |
| 초등           | 수학 3-1       | 한대희       | <CDB_PRIVATE_DOMAIN> |                   | 수학 3-1       |                                  | lms_lrm_math05 / lms_cms_math05 / lms_lrw_math05 |
| 초등           | 수학 3-2       | 한대희       | <CDB_PRIVATE_DOMAIN> |                   | 수학 3-2       |                                  | lms_lrm_math06 / lms_cms_math06 / lms_lrw_math06 |

> 참고: 실제 엑셀에는 교과목군(영어/수학 등) 전체 매핑이 포함되어 있으며,  
> 본 문서에는 발췌본만 포함했습니다.

## 6. PoC 결과

- 영어(eng01~eng12) 구조 Import 후 테이블 생성 수 검증(일일 정리 기반):
  - `lms_lrm_engXX` : **136 tables**
  - `lms_cms_engXX` : **91 tables**
  - `eng01~eng12` : **12개 단위** 반복 검증 로그 확보(nohup + 로그 리다이렉션)

## 7. 산출물(증빙)

- [x] 통합안 표준 네이밍 규칙(교과목 suffix 체계) 정리
- [x] 스키마 덤프/복원 PoC 절차(스크립트/명령/검증 포인트 포함)
- [x] Import 결과 검증(테이블 수 등) 로그 확보
- [ ] 다음 단계: 실데이터 이관 리허설(부분 샘플) + 성능/용량 산정(종량제 대응)

