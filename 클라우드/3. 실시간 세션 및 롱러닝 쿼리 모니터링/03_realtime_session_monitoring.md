# NCP Cloud DB 관측 파이프라인 통합 문서

작성일 2026-02-19
작성자 김영석
범위 NCP Cloud DB for MySQL 다수 인스턴스 대상 실시간 Active Session 3초 샘플링과 Slow Error 로그의 일일 수집 가공을 통합 운영

보안 안내
본 문서는 포트폴리오 공개와 공유를 고려하여 내부 도메인, 계정, 키, 버킷명, 시스템 식별자 등 민감정보를 마스킹했다.
운영 원문은 사내 보안 정책에 따라 시크릿 분리, 권한 분리, 접근 통제 하에 별도 보관한다.

---

## 1. 배경과 목적

### 1.1 왜 필요한가

운영 환경에서는 짧은 시간에 폭증하는 쿼리나 락이 발생할 수 있고, Slow log만으로는 그 순간의 동시 세션 맥락이 누락될 수 있다.
다수 인스턴스 환경에서는 장애 징후가 어느 인스턴스에서 먼저 시작됐는지, 어떤 사용자와 호스트가 반복적으로 부하를 만들었는지 시간축 증적이 필요하다.

### 1.2 목표

1. 실시간 타임라인 증적 확보
   3초 간격으로 Active Session을 샘플링해 장애 성능 저하 시점에 그 순간 실행 중이던 세션과 대기 상태를 남긴다.

2. 일일 튜닝 후보군 자동 생산
   Slow Error 로그를 표준 수집하고 전일 Slow 로그를 자동 요약해 튜닝 후보를 매일 생성한다.

3. 접근통제와 감사 관점에 유리한 수집 체계
   DB 서버 OS에 직접 접속하지 않고 NCP 공식 Export to Object Storage 기능 기반으로 로그를 수집해 운영 감사 증적에 유리한 흐름을 갖춘다.

---

## 2. 전체 아키텍처 개요

### 2.1 구성 요소

작업서버 VMS
cron
실시간 샘플링 스크립트 main_active_3s.sh, active_3s.sh
로그 수집 스크립트 db_log.sh, db_prd_log.sh, 2026_stg_log.sh
로그 가공 스크립트 slow_sort.sh
Object Storage
로컬 파일 시스템 분석 디렉토리

### 2.2 데이터 흐름

실시간 Active Session
cron 매분 트리거
-> main_active_3s.sh
-> 3초 간격 3부터 57까지 active_3s.sh 호출
-> 각 인스턴스 processlist 수집
-> 로컬 디렉토리에 시간축 로그 적재

Slow Error 로그
Cloud DB Instances
-> NCP CLI exportDbServerLogToObjectStorage logType SLOW, ERROR
-> Object Storage 환경별 prefix
-> VMS list objects, download
-> 로컬 디렉토리에 raw 로그 적재
-> slow_sort.sh 병합 및 mysqldumpslow 요약
-> 일일 리포트 산출

---

## 3. 실시간 Active Session 3초 샘플링

### 3.1 한 줄 요약

작업서버에서 cron 1분 트리거로 3초 간격 3부터 57까지 Active Session을 샘플링하고, 각 DB 인스턴스의 information_schema.processlist를 수집해 시간축 증적 로그를 축적했다.

### 3.2 스케줄

매분 실행
main_active_3s.sh
필요 시 웹 전시용 별도 스크립트 main_active_3s_WEB.sh

### 3.3 실행 방식

cron은 1분 단위이므로 스크립트 내부에서 sleep 분할로 3초 주기 샘플링을 구현한다.
총 19회, 3초부터 57초까지 백그라운드 실행을 예약한다.

예시 개념
sleep 3  후 active_3s.sh 실행
sleep 6  후 active_3s.sh 실행
...
sleep 57 후 active_3s.sh 실행

### 3.4 수집 쿼리 요약 뷰

수집 대상 information_schema.processlist
필터 조건
STATE 비어있지 않음
TIME 0이 아님
운영 계정, 복제 계정 등 노이즈 계정 제외

예시
SELECT ID, USER, HOST, DB, COMMAND, TIME,
SUBSTR(STATE,1,30) AS state,
SUBSTR(INFO,1,50)  AS sql
FROM information_schema.processlist
WHERE STATE <> ''
AND TIME <> 0
AND USER NOT IN ('root','repl_admin','lrm_infra')
ORDER BY TIME DESC;

### 3.5 수집 쿼리 상세 뷰

장애 분석 시 원문 증빙을 위해 full dump를 함께 남길 수 있다.

예시
SELECT *
FROM information_schema.processlist
WHERE STATE <> ''
AND TIME <> 0
AND USER NOT IN ('root','repl_admin','lrm_infra')
ORDER BY TIME DESC
\G

운영 부하를 고려해 평시에는 요약 뷰만, 임계치 초과 시 상세 뷰를 추가하는 방식이 권장된다.

### 3.6 저장 구조

디렉토리
/DBA/script/3sec_mon/YYYYMM/DD/

파일
active_session_HH_LRM*.log

각 샘플마다 타임스탬프 헤더를 append하여 시간축 증적을 남긴다.

---

## 4. Slow Error 로그 수집 및 가공 파이프라인

### 4.1 한 줄 요약

작업서버에서 cron으로 매일 NCP Cloud DB의 Slow Error 로그를 Object Storage로 Export하고 VMS로 다운로드한 뒤, 전일 Slow 로그를 병합 및 요약해 튜닝 후보 리포트를 생성한다.

### 4.2 환경 분리 원칙

Object Storage prefix 기준으로 환경과 날짜를 분리한다.
WEB 전시 DB
web_DB_log/YYYYMMDD/

PRD 운영 DB
PRD_DB_log/YYYYMMDD/

STG 2026 DB
STG2026_DB_log/YYYYMMDD/

로컬 저장도 동일 원칙으로 환경별 경로 분리한다. 분석, 감사, 증적 효율을 높이기 위함이다.

### 4.3 실행 스케줄

작업서버 VMS cron 기준

23:30 WEB 전시 DB Export plus Download db_log.sh
23:35 PRD 운영 DB Export plus Download db_prd_log.sh
23:53 STG 2026 Export plus Download 2026_stg_log.sh
08:30 전일 PRD slow 병합 및 요약 slow_sort.sh

### 4.4 스크립트 역할 요약

db_log.sh WEB 전시

* web_DB_log 날짜 prefix 하위 기존 파일이 있으면 선삭제 후 재적재로 리런 보장
* 인스턴스 배열을 순회하며 slow error 로그를 Object Storage로 export
* 반영 대기 후 list objects로 파일 확인, VMS로 다운로드
* 다운로드 경로 예시
  /backup/DB_logs/YYYYMMDD/slow_log/
  /backup/DB_logs/YYYYMMDD/error_log/
* 실행 로그는 별도 파일로 남김
  script_db_log/script_db_log_timestamp.log

db_prd_log.sh PRD 운영

* PRD_DB_log 날짜 prefix로 동일한 export download 수행
* 다운로드 경로 예시
  /backup/DB_logs/YYYYMMDD/prd_slow_log/
  /backup/DB_logs/YYYYMMDD/prd_error_log/

2026_stg_log.sh STG 2026

* STG2026_DB_log 날짜 prefix로 동일한 export download 수행
* 다운로드 경로 예시
  /backup/DB_logs/2026stg_log/YYYYMMDD/slow_log/
  /backup/DB_logs/2026stg_log/YYYYMMDD/error_log/

slow_sort.sh 전일 PRD slow 가공

* 전일 PRD slow log 조각을 병합해 slow_all.log 생성
* mysqldumpslow로 요약 리포트 생성
  현재 빈도 기준 리포트
  개선 권장 지연 기준 Top N 리포트 병행

예시
mysqldumpslow -s c  slow_all.log > report_by_count.txt
mysqldumpslow -s at -t 20 slow_all.log > report_top_time_20.txt

---

## 5. 운영 절차

### 5.1 최초 적용

1. 스크립트 배치
   실시간 샘플링
   /DBA/script/3sec_mon/

로그 수집 가공
운영 표준 경로에 배치

2. 실행 권한 부여
   chmod +x main_active_3s.sh active_3s.sh
   chmod +x db_log.sh db_prd_log.sh 2026_stg_log.sh slow_sort.sh

3. crontab 등록
   실시간 샘플링 예시
   매분 main_active_3s.sh

로그 수집 가공 예시
지정 시간대에 각 스크립트 실행

### 5.2 장애 성능 분석 시 활용

1. 장애 시각을 기준으로 실시간 Active Session 로그 확인
   해당 날짜 디렉토리에서 시간 단위 로그 파일을 열고 TIME 내림차순 세션 확인
   동일 USER, HOST 반복 여부 확인
   STATE 기반으로 Lock, IO, metadata lock 등 추정

2. 같은 날짜의 Slow 리포트 확인
   전일 리포트에서 후보 SQL을 확인하고, 필요 시 raw slow 로그에서 원문 확인
   실시간 로그와 slow 리포트를 시간대 관점에서 교차 검증

사례 문서
별도 문서 참조

---

## 6. 운영화 개선 항목

### 6.1 중복 실행 방지

실시간 샘플링과 로그 수집 모두 동시 실행 방지가 필요하다.
lockfile 또는 flock 적용을 권장한다.
60초 초과 시 다음 실행은 스킵하고 스킵 로그를 남기는 방식이 운영 안정성에 유리하다.

### 6.2 시크릿 제거

스크립트 내 Access Key, Secret Key, DB password 평문 저장을 제거한다.
권한 기반 접근, 시크릿 매니저, 환경변수 분리 등 사내 표준 방식으로 전환한다.

### 6.3 부하 통제

실시간 샘플링
평시 요약 뷰 중심 수집
임계치 초과 시 상세 뷰 추가

로그 수집
export download 수행 시간과 다운로드 용량을 감안해 시간대를 분리 운영
인스턴스별 실패만 재시도 가능한 구조 도입 권장

### 6.4 보관과 롤링

로그 디스크 full 방지를 위해 보존기간과 삭제 대상을 정책으로 확정한다.
mtime 기반 삭제, 압축, 로테이션을 적용한다.
삭제 작업도 실행 로그를 남겨 감사와 증빙이 가능하도록 한다.

---

## 7. 산출물과 증적

1. 실시간 Active Session 시간축 로그
   /DBA/script/3sec_mon/YYYYMM/DD/active_session_HH_LRM*.log

2. Object Storage raw 로그
   환경별 prefix 하위 raw 로그 파일

3. VMS 로컬 raw 로그
   환경별 날짜별 raw 로그 디렉토리

4. 일일 요약 리포트
   mysqldumpslow 결과 리포트 파일

5. 선택 증적
   PMM, performance_schema 기반 전후 지표 스냅샷

---

## 8. 개선 로드맵

1. 대상 서버 목록 파일화
   하드코딩 제거, csv 또는 yaml 기반으로 인스턴스 관리

2. digest 단위 집계
   performance_schema 또는 sys 스키마 기반으로 쿼리 지문 단위 상위 후보를 정량화

3. 실시간과 일일의 연결 강화
   실시간 타임라인과 slow 리포트의 교차 검증 절차를 표준화
   장애 시각 기준으로 자동 링크되는 형태의 분석 보조 스크립트 도입

4. 알림 체계
   실패 시 Slack webhook 메일 알림
   부분 실패 재시도 큐 도입

---

원하면 다음 단계로, 너가 말한 스타일을 유지하면서도 더 강하게 보이게 문서 상단에 면접용 5줄 요약과 운영화 체크리스트 10줄을 붙여서 완전 포트폴리오 모드로 마감해줄게.
