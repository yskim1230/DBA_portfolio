
**회사명:** ㈜천재교육
**재직기간:** 2024.01 ~ 현재 (재직중)
**직무:** DBA / IT인프라팀
**주요 역할:** 대규모 교육 플랫폼(T셀파, 쇼핑몰, 밀크T 등)의 MSSQL/MariaDB 운영 관리, 성능 최적화, ISMS 보안 체계 구축

#### 1. 데이터베이스 성능 고도화 및 튜닝

 
**전사적 인덱스 최적화 전략 수립:** 응답 소요 시간별(1초 미만~5초 이상) 리스트를 세분화하여 총 5단계의 인덱스 리빌드 전략을 수립 및 수행, 데이터 페이지 밀도 최적화 및 조회 속도 개선.


 
**핵심 프로시저(SP) 튜닝:** `USP_Admin_Member_Excel`, `USP_New_Member_EasyJoin` 등 고부하 핵심 로직을 리팩토링하고 전용 인덱스를 추가하여 실행 시간 단축 및 CPU/IO 부하 경감.


 
**I/O 병목 해소:** 주요 운영 DB(Tsherpa2021 등)의 Slow Query를 상시 모니터링하여 인덱스 신규 생성 및 쿼리 리팩토링을 통해 디스크 I/O 과부하 문제 해결.



#### 2. 선제적 모니터링 체계 구축 및 장애 예방

 
**실시간 장애 대응 시스템 구축:** MaxGauge 이벤트 알람과 SMTP 메일 연동을 통해 장애 징후를 즉각 통보받는 체계를 확립, 장애 인지 및 대응 시간 단축.


 
**오픈소스 모니터링 확장:** MariaDB 및 MySQL 환경에 최적화된 Zabbix 모니터링 PoC를 성공적으로 수행 및 도입하여 이기종 DBMS에 대한 통합 관제 토대 마련.


 
**고가용성(HA) 아키텍처 운영:** AIDT 홍보관, 문제은행 등 핵심 서비스의 MariaDB MHA(Master High Availability) 클러스터 구성 및 무장애 리부팅/점검 수행으로 서비스 가용성 확보.



#### 3. 보안 컴플라이언스(ISMS) 대응 및 데이터 거버넌스

 
**ISMS 인증 심사 완벽 대응:** 암호화 키 대장 최신화, 패스워드 정책 강화, 취약점 조치 스크립트 작성 등 보안 요구사항을 100% 이행하여 인증 심사(인터뷰 및 실사) 완수.


 
**데이터 보안 프로세스 정립:** 운영 DB와 개발 DB 간의 링크 연결 제한 가이드라인을 직접 수립 및 전파하여 휴먼 에러로 인한 데이터 유출 리스크 차단.


 
**개인정보 보호 조치:** 테스트/개발 환경 이관 시 개인정보 마스킹 및 암호화 백업/복원 테스트 시나리오를 수립하고 정기적으로 이행.



#### 4. 인프라 운영 효율화 및 마이그레이션

 
**DB 서버 마이그레이션:** 노후 서버(150대역)에서 신규 서버(170대역)로 운영 DB 마이그레이션 계획 수립, 사전 복구 테스트, 스크립트 작성을 통해 안정적인 이관 완료.


 
**리소스 라이프사이클 관리:** 미사용 테이블, 임시 스키마, 1년 이상 된 로그 데이터를 5차에 걸쳐 전수 조사 및 삭제하여 스토리지 효율성 제고.


 
**개발 생산성 지원:** 신규 서비스(클래스보드 등)의 DB 환경(MariaDB)을 신속 구축하고, 운영-개발-스테이징 간 데이터 동기화(연간 40건 이상)를 적기에 지원하여 개발/QA 정합성 유지.


 
**비즈니스 요구사항 대응:** SMS 발송 업체 변경(CJ→다우)에 따른 신규 테이블 스키마 설계 및 대량 예약 발송 프로시저 개발로 서비스 중단 없는 전환 지원.



---

### [Tech Stack]

* **Database:** MS-SQL Server (Expert), MariaDB/MySQL (Advanced), PostgreSQL
* **Performance Tuning:** Query Optimization, Index Tuning, Execution Plan Analysis, Lock & Deadlock Troubleshooting
* **High Availability & DR:** MHA (Master High Availability), Replication, Backup & Recovery Strategy
* **Monitoring Tools:** MaxGauge, Zabbix
* **Security:** ISMS Compliance, TDE, Data Masking, DB Access Control
* **Language:** SQL, T-SQL (Stored Procedure/Trigger/Function), Python (Automation Scripting)

---

