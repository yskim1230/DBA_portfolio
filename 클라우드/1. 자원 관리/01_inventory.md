# 클라우드 DB 자원관리 인벤토리 (1년차 VPC CDB)

- 작성일: 2026-02-09
- 작성자: 김영석
- 범위: NCP VPC 기반 CDB(MySQL) **1년차 영역** 자원 인벤토리
- 근거 자료: `CloudDB_MySQL(VPC)_1년차 DB 리스트.xlsx`

> **보안 안내**: 본 문서는 포트폴리오 공개/공유를 고려하여 내부 도메인, 계정, 비밀번호, 티켓/시스템 식별자 등 민감정보를 **마스킹**했습니다.  
> 실제 운영 문서에는 사내 보안 정책(시크릿 매니저/권한분리/접근통제)에 따라 원본 정보를 별도 보관합니다.


## 1. 목적

- **운영/스테이징/개발** 환경별 DB 자원을 한눈에 파악하고,  
  (1) 통합/회수 대상 선정, (2) 백업·보안·성능 정책 적용 범위, (3) 장애 대응 범위(HA/StandAlone) 를 명확히 하기 위함.

## 2. 인벤토리 요약 (핵심 수치)

- DB 서버(인스턴스) 수: **288대**
- DB 서비스(서비스명 기준) 수: **167개**
- VPC 수: **8개**
- Subnet 수: **125개**
- MySQL 버전 분포: **MYSQL8.0.42** 272대, **MYSQL8.4.6** 9대, **MYSQL8.0.34** 7대

### 2.1 환경별 분포(서비스 기준)

| ENV   |   db_servers |   db_services |
|:------|-------------:|--------------:|
| prd   |          197 |           100 |
| stg   |           51 |            40 |
| dev   |           36 |            25 |
| etc   |            4 |             2 |

### 2.2 HA 구성 현황 (Role 기준)

| DB Role        |   count |
|:---------------|--------:|
| Master         |     118 |
| Standby Master |     118 |
| Stand Alone    |      49 |
| Slave          |       2 |
| Recovery       |       1 |

> 해석 가이드  
> - `Master + Standby Master` : **HA(이중화) 서비스**  
> - `Stand Alone` : 단독 구성(HA 아님)  
> - `Slave/Recovery` : 특수 목적(복제/복구)

- HA 서비스(서비스명 기준): **118개**
- Stand Alone 서비스(서비스명 기준): **49개**

### 2.3 VPC별 서비스 분포(서비스 기준)

| VPC             |   dev |   etc |   prd |   stg |
|:----------------|------:|------:|------:|------:|
| ai-dev-vpc-01   |     1 |     0 |     0 |     0 |
| lms-comm-vpc-01 |     0 |     2 |     0 |     0 |
| lms-dev-vpc-01  |    23 |     0 |     0 |     0 |
| lms-prd-vpc-01  |     0 |     0 |    98 |     0 |
| lms-stg-vpc-01  |     0 |     0 |     0 |    39 |
| stt-dev-vpc-01  |     1 |     0 |     0 |     0 |
| stt-prd-vpc-01  |     0 |     0 |     2 |     0 |
| stt-stg-vpc-01  |     0 |     0 |     0 |     1 |

## 3. 운영 관리 기준(Inventory 관점)

### 3.1 필수로 “정리해야 하는 것”

- **서비스명 표준화**: `서비스-환경-도메인-용도` 규칙으로 통일(예: `lms-prd-myd-xxx`)
- **Role 표준화**: HA 대상은 반드시 `Master/Standby Master` 쌍으로 관리
- **백업 보관 정책**: 운영 7일, 그 외(개발/스테이징) 3일 등 *환경별 기준*을 문서화
- **스토리지/암호화**: 암호화 적용 여부, 스토리지 타입/용량(증설 이력 포함) 관리

### 3.2 인벤토리 업데이트 루틴(권장)

- 주 1회: 신규 생성/삭제/Role 변경/VPC 이동 여부 반영
- 월 1회: 스펙(타입/스토리지) 및 백업 보관일 정책 점검(감사 대비)
- 변경 발생 시(수시): 통합/이관/스펙다운/버전업 등 변경 이벤트를 *인벤토리*에 즉시 반영

## 4. 인벤토리 샘플(발췌)

> 아래 표는 “전체 167개 서비스” 중 일부만 발췌했습니다.  
> Private/Public 도메인 등 민감정보는 별도 관리합니다.

| DB 서비스 이름       | ENV   | DB Role     | VPC            | Subnet                        | DB Server 타입                | 데이터 스토리지 용량   | DB 엔진 버전   | 백업 보관일(백업시간)   |
|:---------------------|:------|:------------|:---------------|:------------------------------|:------------------------------|:-----------------------|:---------------|:------------------------|
| ai-dev-myd           | dev   | Stand Alone | ai-dev-vpc-01  | ai-dev-sub-prv-myd-01         | G2 - [Standard]2vCPU, 8GB Mem | 10GB                   | MYSQL8.0.42    | 7일(02:00)              |
| lcms-dev-myd-002     | dev   | Stand Alone | lms-dev-vpc-01 | lcms-dev-sub-prv-db-01        | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| lcms-dev-myd-01      | dev   | Master      | lms-dev-vpc-01 | lcms-dev-sub-prv-db-01        | G2 - [Standard]2vCPU, 8GB Mem | 10GB                   | MYSQL8.0.42    | 7일(02:00)              |
| lms-dev-csb-01       | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-csb-01        | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| lms-dev-etb-01       | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-myd-etb-01    | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 1일(13:00)              |
| lms-dev-exh-01       | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-myd-exh-01    | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 1일(01:45)              |
| lms-dev-myd-2026-201 | dev   | Stand Alone | lms-dev-vpc-01 | lms-dev-sub-prv-myd-201       | G3 - [Standard]2vCPU, 8GB Mem | 30GB                   | MYSQL8.4.6     | 3일(00:30)              |
| lms-dev-myd-admin-01 | dev   | Stand Alone | lms-dev-vpc-01 | lms-dev-sub-prv-myd-admin-01  | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 1일(11:15)              |
| lms-dev-myd-ats-01   | dev   | Stand Alone | lms-dev-vpc-01 | lms-dev-sub-prv-ats-01        | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 3일(07:15)              |
| lms-dev-myd-ats-test | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-test-01       | G3 - [Standard]2vCPU, 8GB Mem | 3GB                    | MYSQL8.4.6     | 1일(07:30)              |
| lms-dev-myd-eng-01   | dev   | Stand Alone | lms-dev-vpc-01 | lms-dev-sub-prv-myd-lw-01     | G3 - [Standard]2vCPU, 8GB Mem | 8GB                    | MYSQL8.0.42    | 7일(02:00)              |
| lms-dev-myd-lm-01    | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-myd-lm-01     | G2 - [Standard]2vCPU, 8GB Mem | 28GB                   | MYSQL8.0.42    | 7일(02:00)              |
| lms-dev-myd-lm-02    | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-myd-lm-02     | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| lms-dev-myd-lw-01    | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-myd-lw-01     | G2 - [Standard]2vCPU, 8GB Mem | 10GB                   | MYSQL8.0.42    | 7일(02:00)              |
| lms-dev-myd-lw-02    | dev   | Master      | lms-dev-vpc-01 | lms-dev-sub-prv-myd-lw-02     | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| lms-dev-myd-math-01  | dev   | Stand Alone | lms-dev-vpc-01 | lms-dev-sub-prv-myd-lm-01     | G3 - [Standard]2vCPU, 8GB Mem | 10GB                   | MYSQL8.0.42    | 7일(02:00)              |
| mh-dev-myd-01        | dev   | Stand Alone | lms-dev-vpc-01 | mh-dev-sub-prv-db-01          | G2 - [High CPU]4vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:45)              |
| mh-dev-myd-02        | dev   | Stand Alone | lms-dev-vpc-01 | mh-dev-sub-prv-db-02          | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:45)              |
| mh-dev-myd-03        | dev   | Stand Alone | lms-dev-vpc-01 | mh-dev-sub-prv-db-03          | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(03:30)              |
| mh-dev-myd-04        | dev   | Stand Alone | lms-dev-vpc-01 | mh-dev-sub-prv-db-04          | G2 - [High CPU]4vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| mh-dev-myd-05        | dev   | Stand Alone | lms-dev-vpc-01 | mh-dev-sub-prv-db-05          | G2 - [High CPU]4vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| mh-dev-myd-06        | dev   | Stand Alone | lms-dev-vpc-01 | mh-dev-sub-prv-db-06          | G2 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| mh-dev-myd-07        | dev   | Stand Alone | lms-dev-vpc-01 | mh-dev-sub-prv-db-07          | G2 - [High CPU]4vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| portal-dev-myd-01    | dev   | Stand Alone | lms-dev-vpc-01 | lms-dev-sub-prv-myd-portal-01 | G3 - [Standard]2vCPU, 8GB Mem | 9GB                    | MYSQL8.0.42    | 7일(02:00)              |
| stt-dev-myd-01       | dev   | Master      | stt-dev-vpc-01 | stt-dev-sub-prv-myd-01        | G2 - [Standard]2vCPU, 8GB Mem | 10GB                   | MYSQL8.0.34    | 7일(02:00)              |

## 5. 산출물(증빙)

- [x] `CloudDB_MySQL(VPC)_1년차 DB 리스트.xlsx` 최신화(서비스/Role/버전/백업/스토리지)
- [x] 환경별/Role별/버전별 요약(본 문서)
