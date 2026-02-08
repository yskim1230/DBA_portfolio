# 실시간 세션 및 롱러닝 쿼리 모니터링 (3초 주기 수집)

- 작성일: 2026-02-09
- 작성자: 김영석
- 범위: MySQL(CDB) 다수 인스턴스 대상으로 **Active Session/Long Running Query**를 3초 간격으로 수집
- 근거 자료: `일일 슬로우 쿼리 수집 방법.txt`, `active_3s(마스킹).sh`, `main_active_3s.sh`

> **보안 안내**: 본 문서는 포트폴리오 공개/공유를 고려하여 내부 도메인, 계정, 비밀번호, 티켓/시스템 식별자 등 민감정보를 **마스킹**했습니다.  
> 실제 운영 문서에는 사내 보안 정책(시크릿 매니저/권한분리/접근통제)에 따라 원본 정보를 별도 보관합니다.


## 1. 배경 / 문제 정의

- 운영 환경에서 “짧은 시간에 폭증하는 쿼리/락”은 **Slow Log(분/초 단위)**만으로는 원인 지점이 누락될 수 있음.
- 특히 다수 인스턴스(교과목/서비스 단위로 분산)에서 장애 징후가 나타날 때,
  - **실시간 세션(processlist) 스냅샷**을 촘촘히 남겨야 “누가/어디서/무슨 SQL로” 부하를 유발했는지 역추적 가능.

## 2. 목표

- 3초 주기로 Active Session을 수집하여,
  - Long running query (TIME 증가)
  - Lock 대기(STATE)
  - 쿼리 텍스트(INFO 일부)
  를 빠르게 식별/증빙

## 3. 아키텍처

1) `cron`(매 분) → 2) `main_active_3s.sh`(sleep 분할) → 3) `active_3s.sh`(DB별 processlist 수집)

- cron 1분 단위 한계를 `sleep` 조합으로 우회
- 수집 로그를 **월/일 단위 디렉토리**로 자동 분리

## 4. 구현 상세

### 4.1 수집 SQL (핵심)

- `information_schema.processlist` 기반
- 필터 조건:
  - `STATE != ''`
  - `TIME != 0`
  - 시스템/복제 계정 제외(예: `root`, `repl_admin`, `lrm_infra`)

```sql
select  ID, USER, HOST, DB, COMMAND, TIME,
        substr(STATE,1,30) as STATE,
        substr(INFO,1,50)  as SQL
from information_schema.processlist
where STATE != ''
  and USER not in ('root','repl_admin','lrm_infra')
  and TIME != 0
order by TIME desc;
```

### 4.2 3초 간격 스케줄러 (`main_active_3s.sh`)

```bash
(sleep 3 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 6 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 9 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 12 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 15 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 18 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 21 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 24 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 27 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 30 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 33 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 36 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 39 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 42 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 45 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 48 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 51 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 54 && /DBA/script/3sec_mon/active_3s.sh) & 
(sleep 57 && /DBA/script/3sec_mon/active_3s.sh) &

#find /DBA/script/3sec_mon/202*/*/active_session_*LRM*.log -mtime 3 -exec rm -f {} \;
...
```

### 4.3 수집기(`active_3s.sh`) 핵심 로직(발췌)

- 로그 경로: `/DBA/script/3sec_mon/YYYYMM/DD/`
- 로그 파일: `active_session_<HOUR>_<INSTANCE>.log`
- 각 인스턴스에 대해:
  - 표(테이블 형식) 출력(`-t`)로 1차 확인/가독성 확보
  - FULL 출력(`\G`)로 원문 증빙

```bash
LANG=ko_KR.eucKR

export LANG



#v_USER="LRM_ADMIN"
v_USER="lrm_infra"
v_USER1="lrm_chunjae"

v_PASSWORD="[secret]"

v_date=`date +%Y%m%d_%H%M%S`



### Log Dir Created

v_dir1=`date +%Y%m`

v_dir2=`date +%d`

v_dir3=`date +%H`

v_act_dir=/DBA/script/3sec_mon/${v_dir1}/${v_dir2}



mkdir -p $v_act_dir







SQL_active_session="select  ID, USER, HOST, DB, COMMAND, TIME, SUBSTR(STATE,1,30) as 'STATE', SUBSTR(INFO,1,50) as 'SQL'
#, MEMORY_USED

from information_schema.processlist

where  1= 1

and STATE != ''

and USER NOT IN ('root','repl_admin','lrm_infra')
and TIME NOT IN (0)
order by TIME desc;"





SQL_actvie_full="select *

from information_schema.processlist

where  1= 1

and STATE != ''

and USER NOT IN ('root','repl_admin')
and TIME NOT IN (0)
order by TIME desc\G"



echo "#### `date '+%Y-%m-%d %H:%M:%S'` ############" >> ${v_act_dir}/active_session_${v_dir3}_LRM1.log
mysql --user=$v_USER1  --p<SECRET> -h<CDB_PRIVATE_DOMAIN> -t -e "${SQL_active_session}" | tee -a ${v_act_dir}/active_session_${v_dir3}_LRM1.log
mysql --user=$v_USER1  --p<SECRET> -h<CDB_PRIVATE_DOMAIN> -e "${SQL_actvie_full}" >> ${v_act_dir}/active_session_${v_dir3}_LRM1.log

echo "#### `date '+%Y-%m-%d %H:%M:%S'` ############" >> ${v_act_dir}/active_session_${v_dir3}_LRM1_1.log
mysql --user=$v_USER1  --p<SECRET> -h<CDB_PRIVATE_DOMAIN> -t -e "${SQL_active_session}" | tee -a ${v_act_dir}/active_session_${v_dir3}_LRM1_1.log
mysql --user=$v_USER1  --p<SECRET> -h<CDB_PRIVATE_DOMAIN> -e "${SQL_actvie_full}" >> ${v_act_dir}/active_session_${v_dir3}_LRM1_1.log
...
# (이하 동일 패턴으로 다수 인스턴스 반복)
```

### 4.4 로그 보존/롤링(디스크 Full 방지)

`main_active_3s.sh` 내에 다음과 같은 삭제 정책(주석 포함)이 존재:

```bash
#find /DBA/script/3sec_mon/202*/*/active_session_*LRM*.log -mtime 3 -exec rm -f {} \;
```

> 운영 반영 시 권장:  
> - 보존기간(mtime)과 삭제 대상을 **정책으로 확정**하고,  
> - 실행 로그/삭제 로그를 남겨 감사/증빙 가능하게 구성.

## 5. 운영 절차

### 5.1 설치/적용

1. `/DBA/script/3sec_mon/` 에 스크립트 배치
2. 실행 권한 부여: `chmod +x main_active_3s.sh active_3s.sh`
3. crontab 등록(예): `* * * * * /DBA/script/3sec_mon/main_active_3s.sh`

### 5.2 장애/성능 분석 시 활용 포인트

- 특정 시간대에 DB CPU 100%/응답 지연 발생 시:
  1) 해당 시각의 로그 디렉토리로 이동  
  2) `TIME` 내림차순으로 상위 세션 확인  
  3) 동일 USER/HOST 반복 여부 확인(애플리케이션 원인 추적)  
  4) STATE 기반으로 Lock/IO/metadata lock 추정

## 6. 개선 아이디어

- 대상 서버 목록을 하드코딩 대신 **서버 리스트 파일(csv/yaml) 기반**으로 관리
- `performance_schema` / `sys` 스키마 기반으로 digest(쿼리 지문) 단위 집계
- 슬로우 로그 + processlist 로그를 결합해 “원인 쿼리 → 코드/배포”까지 연결

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

