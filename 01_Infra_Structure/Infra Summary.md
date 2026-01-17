# 인프라 요약

## 1. 개요
천재교육의 전사 IT 인프라 중 핵심 데이터베이스 시스템을 운영 및 관리하고 있습니다. 대규모 트래픽을 처리하는 메인 서비스부터 교육용 콘텐츠 DB까지, 고가용성과 성능 최적화를 최우선 가치로 두고 관리합니다.

## 2. 데이터베이스 환경 (Tech Stack)
현재 관리 중인 환경은 On-premise 기반이며, 차세대 클라우드 전환을 위한 PoC 및 교육을 병행하고 있습니다.

|구분|상세|내용|비고|
Main|RDBMS|MSSQL|(2019)|"메인 서비스, 결제, 회원 데이터 관리"
Open Source,MySQL / MariaDB,"LMS, T셀파, 쇼핑몰 등 서비스별 특화 DB"
High Availability,MHA (MySQL High Availability),무장애 자동 Failover 체계 구축
Monitoring,"Maxgauge, Zabbix (PoC)",실시간 지표 분석 및 알람 연동
Infrastructure,"IDC On-premise, AWS (Migration 준비)",하이브리드 환경 관리 역량 보유
