# 인프라 요약

## 1. 개요
천재교육의 전사 IT 인프라 중 핵심 데이터베이스 시스템을 운영 및 관리하고 있습니다. 대규모 트래픽을 처리하는 메인 서비스부터 교육용 콘텐츠 DB까지, 고가용성과 성능 최적화를 최우선 가치로 두고 관리합니다.

## 2. 데이터베이스 환경 (Tech Stack)
현재 관리 중인 환경은 On-premise 기반이며, 차세대 클라우드 전환을 위한 PoC 및 교육을 병행하고 있습니다.

| 구분 | 분류 | 기술 스택 | 주요 역할 및 특징 |
| :--- | :--- | :--- | :--- |
| **Main RDBMS** | Commercial | **MSSQL (2019)** | 메인 서비스, 결제, 회원 데이터 등 전사 핵심 데이터 관리 |
| **Open Source** | Open Source | **MySQL / MariaDB** | LMS, T셀파, 쇼핑몰 등 서비스별 특화 데이터베이스 운영 |
| **High Availability** | Availability | **MHA** | MySQL High Availability 기반 무장애 자동 Failover 체계 구축 |
| **Monitoring** | Observability | **Maxgauge / Zabbix** | 실시간 지표 분석 및 SMTP 연동을 통한 선제적 장애 알람 |
| **Infrastructure** | Hybrid | **IDC / AWS** | IDC On-premise 운영 및 AWS 마이그레이션 전략 수립(PoC) |


## 3. Storage Infrastructure
고성능 All-Flash 스토리지와 dHCI 가상화 인프라를 통해 서비스 데이터의 물리적 안정성과 성능을 보장합니다.
| 서비스명 | 위치 | 총 용량(TB) | 사용량(TB) | 사용량(%) | 남은 용량(TB) | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Nimble 스토리지(DB스토리지) | 목동 KT IDC | 11 | 2.856 | 26.0% | 8.144 | - |
| DHCI 스토리지_Datastore01 | 목동 KT IDC | 10 | 5.5 | 55.0% | 4.5 | - |
| DHCI 스토리지_Datastore02 | 목동 KT IDC | 10 | 4.1 | 41.0% | 5.9 | - |
| DHCI 스토리지_Datastore03 | 목동 KT IDC | 10 | 5.5 | 55.0% | 4.5 | - |
| DHCI 스토리지_Datastore04 | 목동 KT IDC | 10 | 6.8 | 68.0% | 3.2 | - |
| 백업 스토리지#1 | 목동 KT IDC | 18.1 | 11.56 | 63.9% | 6.54 | - |
| 백업 스토리지#2 | 목동 KT IDC | 25.7 | 18.6 | 72.4% | 7.1 | DB 소산 백업 전용 |

## 4. Backup & Recovery Policy
데이터 유실 방지 및 재해 복구(DR)를 위해 다중화된 백업 체계를 운영하며, ISMS 인증 기준에 따른 엄격한 보관 정책을 준수합니다.
| 백업 명 | 소스 | 타겟 | 백업 주기 | 총 용량(GB) | 보관주기 | 방식 | 데이터 유형 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 천재교육 DB | 천재교육DB | 천재교육DB | 주간(Full/Incr) | 600 | 5주 | SQL 백업 | DB Data SQL |
| 천재교육 DB 소산 | 천재교육 DB 로컬 | 백업스토리지#2 | 주간(Full/Incr) | 19,046 | 2년 | SQL 백업본 복사 | DB Data SQL |
| 티셀파 문제은행 DB | 티셀파 문제은행 DB | 티셀파 문제은행 DB | 일간(Full) | 5.6 | 9일 | SQL 백업 | DB Data SQL |
| 티셀파 문제은행 DB 소산 | 티셀파 문제은행 DB | 백업스토리지#1 | 일간(Full) | 40 | 3개월 | SQL 백업본 복사 | DB Data SQL |
| 티셀파 쇼핑몰 DB | 티셀파 쇼핑몰 DB | 티셀파 쇼핑몰 DB | 일간(Full) | 110 | 1주 | SQL 백업 | DB Data SQL |
| 티셀파 쇼핑몰 DB 소산 | 티셀파 쇼핑몰 DB 로컬 | 백업스토리지#1 | 일간(Full) | 1,402 | 3개월 | SQL 백업본 복사 | DB Data SQL |
| 운영 윈도우 OS(25EA) | Windows OS | 백업스토리지#1 | 일간(Full) | 3,952 | 1일 | Image Backup | OS 및 Data |
| 운영 OS 아크로니스 백업 | Windows OS/Data | 백업스토리지#1 | 주간(Full) | 4,819 | 3개월 | 아크로니스 | OS 및 Data |
| DBsec 백업 | DB 접근제어 | 백업스토리지#1 | 일간(Incr) | 1,382 | 1년 | 원본 백업 | SQL Data 및 로그 |
| Netapp 데이터 백업(1~3) | Netapp Storage | 백업스토리지#2 | 월간(Full) | 약 4,500 | 6개월 | 컴볼트 | 서비스 콘텐츠 |

## 5. 구조도 요약
![천재교육](https://github.com/user-attachments/assets/5a7b4dd6-e162-4bf6-8944-614745dbb88b)


## 6. 핵심 관리 지표 (Operational Excellence)
데이터 가용성: MHA(MySQL High Availability) 구성을 통한 핵심 서비스 무장애 운영 및 자동 Failover 체계 유지.

복구 신뢰성: 연간 40건 이상의 DB 복원 및 최신화(LMS_TSTI, Chunjae_ACA 등)를 통해 백업 데이터의 정합성을 상시 검증.

보안 준수: ISMS 보안 감사 요구사항에 따른 암호화 백업 및 접근 제어 정책 100% 이행.

