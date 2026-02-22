# 클라우드 포트폴리오 목차

## 1. 개요
NCP Cloud DB for MySQL 운영 전환 과정에서 수행한 업무를 절차와 증적 중심으로 정리했습니다.
목표는 운영 표준화, 관측 체계 구축, 통합 이관 PoC, 접근통제와 자동화입니다.

## 2. 먼저 보면 좋은 순서
1) 대표사례 3건
2) 관측 파이프라인 Active Session 샘플링, Slow Error 로그 수집 가공
3) 통합 이관 PoC Dump Restore 검증 자동화
4) VPC 접근통제와 장애 트러블슈팅
5) 운영 설정 표준화

## 3. 대표사례
- []

## 4. 목차
1) 자원 인벤토리
- [01_자원인벤토리][inv]
  
  [inv]: https://github.com/yskim1230/DBA_portfolio/blob/main/%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C/1.%20%EC%9E%90%EC%9B%90%20%EA%B4%80%EB%A6%AC/01_inventory.md

2) 자원 최적화 작업 계획 및 이관 Poc
- [02_자원 최적화 작업][Optimization]
  
  [Optimization]: https://github.com/yskim1230/DBA_portfolio/blob/main/%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C/2.%20%EC%9E%90%EC%9B%90%20%EC%B5%9C%EC%A0%81%ED%99%94%20%EC%9E%91%EC%97%85/02_resource_optimization.md

3) 실시간 세션 및 롱러닝 쿼리 모니터링
- [03_실시간 세션 및 롱러닝 쿼리 모니터링][mor]

  [mor]: https://github.com/yskim1230/DBA_portfolio/blob/main/%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C/3.%20%EC%8B%A4%EC%8B%9C%EA%B0%84%20%EC%84%B8%EC%85%98%20%EB%B0%8F%20%EB%A1%B1%EB%9F%AC%EB%8B%9D%20%EC%BF%BC%EB%A6%AC%20%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81/03_realtime_session_monitoring.md


4) 분산 DB 통합 이관 PoC Dump Restore
- ./04_통합이관_PoC/

5) 관측 성능 운영 체계
- ./05_관측/
  - Active Session 3초 샘플링
  - Slow Error 로그 수집 가공
  - PMM 운영 적용

6) 백업 배치 운영 자동화
- ./06_운영자동화/

7) 스키마 메타데이터 거버넌스 EXERD 변경관리
- ./07_변경관리/

## 5. 증적 자료 규칙
각 문서는 배경 절차 검증 결과 리스크 롤백 순서로 작성했습니다.
스크립트와 로그 캡처는 문서와 같은 폴더 또는 하위 evidence 폴더에 둡니다.
