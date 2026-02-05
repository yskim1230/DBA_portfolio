# 김영석 | DBA / Database Reliability Engineer  
**SQL Server · MariaDB/MySQL · HA/DR · 보안/컴플라이언스**

- Email: vastys@naver.com  
- Mobile: 010-8731-3771  
- Location: 서울/수도권  
- Git/Portfolio: (링크 입력)

---

## 경력 요약 (Summary)

- **SQL Server 기반 운영/개발/스테이징 DB 운영 및 성능 최적화** (인덱스/쿼리/IO)
- **MariaDB/MySQL 기반 MHA 고가용성(HA) 구성/점검 및 무중단 운영 지원**
- **백업/복구(시나리오·리허설) 및 마이그레이션(사전 테스트·검증·작업계획) 수행**
- **ISMS 및 보안감사 대응**: 계정/권한 정비, 개인정보 마스킹, 암호화 백업 테스트, 취약점 조치
- **모니터링/알람 고도화**: MaxGauge SMTP 연동, Zabbix PoC로 지표/알림 체계 확장
- **운영 자동화(스크립트/스케줄)**: 반복 작업 표준화, 백업/정리 스크립트 운영

---

## 핵심 기술 (Core Skills)

### DBMS
- SQL Server 2019, MariaDB, MySQL

### HA/DR
- MHA(Replication/Failover), 복구 리허설, 백업/보존 정책

### Performance
- 실행계획 분석, 인덱스 설계/리빌드, Slow Query/IO 개선

### Security / Compliance
- 계정·권한 관리, ISMS 증적 대응, 마스킹, 백업 암호화 테스트

### Monitoring
- MaxGauge, Zabbix(PoC), SMTP 알람, 장애 징후 탐지/대응

### Automation / Tools
- SQL Agent/Job, 스크립트(PowerShell/Python: 업무 자동화 수준)

---

# 경력 (Experience)

## ㈜천재교육 | IT인프라팀 DBA | 2024.01 - 재직중

### 주요 업무
- 운영/개발/스테이징 DB(메인 MSSQL, 서비스별 DB) **백업·복구·최신화 및 계정/권한 관리**
- 전사 성능 최적화: **인덱스 리빌드 전략 수립·단계별 시행**, 쿼리 튜닝 및 IO 개선
- 보안/컴플라이언스: **ISMS 대응(마스킹·증적)**, 암호화 백업 테스트, 취약점 조치 및 가이드 배포
- 모니터링/알림: **MaxGauge 이벤트 알람 + SMTP 연동**, Zabbix PoC로 지표/알림 범위 확장
- 가용성: 핵심 서비스 DB **MHA 구성/점검 및 리부팅 등 무중단 운영 지원**

### 대표 성과 (Highlights)
- **5단계 인덱스 리빌드 완수 (3~5월)**  
  - 응답시간 구간별 대상 리스트를 분리해 5차로 수행  
  - 데이터 페이지 밀도 최적화 및 조회 성능 개선
- **핵심 프로시저 튜닝 및 검증 (4/8/9월)**  
  - `USP_Admin_Member_Excel`, `USP_New_Member_EasyJoin` 등 고부하 SP 로직 개선  
  - 전용 인덱스 추가로 CPU/IO 부하 경감 확인
- **Slow Query/디스크 IO 과다 쿼리 상시 개선**  
  - `Tsherpa2021` 등 주요 DB 대상 병목 쿼리의 인덱스 신규 생성 및 SQL 리팩토링
- **자원 정리 5차 수행 (4~6월)**  
  - 미사용 테이블/임시 스키마/1년 초과 로그 데이터 전수 조사·삭제  
  - 스토리지 효율 및 백업 용량 최적화
- **운영-개발-스테이징 최신화 연간 약 40건 수행**  
  - `LMS_TSTI`, `Chunjae_ACA`, `edubankmain` 등 복원·동기화로 테스트 정합성 유지
- **MHA 무장애 유지(6월 등)**  
  - AIDT 홍보관, 문제은행 등 핵심 DB 점검·리부팅을 안정적으로 수행  
  - 장애 예방 및 가용성 확보
- **선제적 알람 체계 구축 (2월)**  
  - MaxGauge 이벤트 알람 및 SMTP 연동으로 장애 인지/대응 시간 단축 및 운영 표준 프로세스 정립
- **Zabbix 모니터링 PoC 성공**  
  - MariaDB/MySQL 환경 오픈소스 기반 모니터링 확장 가능성 입증
- **보안 가이드라인 수립**  
  - 운영/개발 DB 간 링크 연결 제한 가이드 작성·전파로 휴먼에러 기반 유출 리스크 차단

### 주요 수행 내역 (Selected Projects)
- **SMS 문자 발송 업체 변경(CJ → 다우)**  
  - 신규 테이블 스키마 생성, 전송/조회 프로시저 수정, 개발-운영 최종 테스트 및 전환 완료
- **운영 DB 마이그레이션 사전 작업**  
  - 백업 파일 생성 스크립트 및 스케줄 작성, 복구 테스트 수행, 작업계획서 작성 및 일정 조율
- **MariaDB 기반 AIDT 홍보관 DB MHA 구성(5월)**  
  - Replication 구성 및 장애 시나리오 점검
- **MaxGauge 도입 관련 보안/인프라 협의체 참여**  
  - 운영 알람 정책 정리 및 내부 표준화 지원

---

## ㈜시스웨어 | 운영컨설팅팀 DB 엔지니어 | 2018.06 - 2023.01 (4년 8개월)

- K-System ERP(영업/구매/생산/인사) **데이터베이스 운영 및 고객사 유지보수/컨설팅** (SQL Server)
- 고급 SQL 작성(다중 JOIN, 서브쿼리, CTE, 윈도우 함수) 및 실행계획 기반 튜닝으로 병목 개선
- DB 로그 기반 오류 추적/원인 분석 및 데이터 정제(불일치/무결성 검증) 스크립트 작성
- 신규 비즈니스 프로세스에 맞춘 테이블 스키마 설계 및 데이터 아키텍처 업데이트
- 고객 커뮤니케이션 전담: 요구사항 수집 → 분석 → 개발 협업 → 산출물 제공까지 End-to-End 수행

---

# 교육 / 자격 (Education & Certifications)

## 교육
- 코드스테이츠 AI 부트캠프 (2023.02 - 2023.09)  
  - 데이터 분석/ML-DL, Docker·Flask 기반 간단한 데이터 파이프라인 실습, AWS RDS MySQL 구축 실습
- AWS 마이그레이션 집체 교육 수강 (2024.03)
- 세미나: TANIUM World Tour Converge Seoul, Encore DATA x AI Seminar (참석)

## 자격증
- SQLD (한국데이터산업진흥원) — 2023.12
- ADsP (한국데이터산업진흥원) — 2023.09
- 네트워크관리사 2급 — 2017.10
- Oracle OCP 11g — 2017.04
