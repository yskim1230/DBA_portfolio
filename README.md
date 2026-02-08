# DBA Portfolio (On-Premise / Cloud) — 김영석

천재교육 DBA로서 **전사 핵심 DB 운영(온프레미스)**과 **클라우드 전환(통합/이관 PoC)** 과정에서 수행한 업무를  
**증빙 중심(Runbook/스크립트/로그/정량 지표)** 으로 정리한 포트폴리오입니다.

> 원칙: “해봤다”가 아니라 **반복 가능(Automation)** 하고 **검증 가능(Verification)** 한 산출물만 수록합니다.

---

## Portfolio Structure

- **On-Premise**: IDC 기반 운영(가용성/성능/보안/백업/복구/모니터링)
- **Cloud**: NCP 기반 클라우드 운영(통합 1안 PoC, DB 설정 표준화, 덤프/복구 자동화, 접근통제/트러블슈팅)

---

## 1. On-Premise (IDC)

### 1.1 인프라 요약
- 전사 핵심 DB(MSSQL) 및 서비스별 MySQL/MariaDB 운영
- All-Flash 스토리지 + dHCI 기반 가상화 인프라
- 백업/소산/DR 정책 운영 및 정기 복구 검증

### 1.2 고가용성(HA) / 장애 대응
- MySQL/MariaDB **MHA 기반 자동 Failover**
- 장애 대응 Runbook 및 복구 리허설(정기 테스트 포함)

### 1.3 Backup & Recovery
- Full/Incr/일간/주간/월간 등 다층 백업 체계
- 소산 백업 및 보관 정책(ISMS 기준) 운영
- 실제 복구/최신화 수행 내역 기반 절차서

### 1.4 성능/운영 최적화
- 인덱스/쿼리 튜닝, 리빌드, 실행계획 기반 개선
- 용량/로그성 테이블 관리 정책, 운영 자동화 스크립트

### 1.5 보안/감사(ISMS)
- 계정/권한 정책, 접근제어, 마스킹 등 보안 통제
- ISMS 증적/감사 대응 문서화(재현 가능한 절차 중심)

### 1.6 모니터링/관측(Observability)
- MaxGauge / Zabbix 기반 실시간 모니터링
- SMTP 알람 연계 및 장애 조기 탐지 체계

---

## 2. Cloud (NCP)

### 2.1 클라우드 자원 전환 및 목표
- 분산 DB 자원 회수 및 통합 운영 전환 준비
- 통합 설계안 기준, **덤프/복구 기반 통합 PoC** 수행

### 2.2 네트워크/접근통제 운영
- VPC/Subnet/ACG/NACL 구조 파악 및 접근 통제 기준 정리
- DB 커넥션 장애를 **네트워크/권한/인증(SSL)** 으로 분해해 트러블슈팅

### 2.3 DB 설정 표준화 (MySQL 8.4)
- lower_case_table_names, charset/collation, sql_mode 등 표준 설정
- 운영 리스크(이벤트 스케줄러, lock timeout, slow log 기준)와 정책화

### 2.4 상시 통합(1안) PoC — Dump/Restore 자동화
- 단위 분산 DB를 표준 네이밍 규칙(예: eng01)으로 재구성
- export/import 자동화 스크립트 + 검증 로그(완결성/테이블 수)로 증빙

### 2.5 ERD/메타데이터 갱신
- EXERD 기반 통합 ERD 갱신 및 컬럼 변경 반영
- 용어/물리/논리 스키마 일관성 유지

---

## 운영 원칙 (공통)
- **도메인 로직은 앱, DB는 무결성과 저장에 집중**
- 변경은 항상 **검증(정량 지표/로그) + 롤백 전략**을 포함
- 자격증명/민감정보는 문서에 미기재(Secret 분리)

---

## How to Read
1) On-Premise에서 운영 기본기(백업/복구/HA/보안/튜닝)를 확인  
2) Cloud에서 전환 과정(통합/이관 PoC, 설정 표준화, 접근통제, 자동화)을 확인  
3) 각 문서는 “배경 → 절차 → 검증 → 결과/리스크” 구조로 작성

---

## Quick Links
- On-Premise: `/on-prem/`
- Cloud: `/cloud/`
- Runbooks & Scripts: `/runbooks/`, `/scripts/`
- Evidence(Log/Screenshots): `/evidence/`

