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
