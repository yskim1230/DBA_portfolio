# 클라우드 포트폴리오 목차

## 1. 개요
NCP Cloud DB for MySQL 운영 전환 과정에서 수행한 업무를 절차와 증적 중심으로 정리했습니다.
목표는 운영 표준화, 관측 체계 구축, 통합 이관 PoC, 접근통제와 자동화입니다.

## 2. 먼저 보면 좋은 순서
1) 대표사례 3건
2) 관측 파이프라인 Active Session 샘플링, Slow Error 로그 수집 가공
3) 통합 이관 PoC Dump Restore 검증 자동화
4) 운영 설정 표준화

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


4) 스키마 변경 대응
- [04_스키마 변경][schema]

  [schema]: https://github.com/yskim1230/DBA_portfolio/blob/main/%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C/4.%20%EC%8A%A4%ED%82%A4%EB%A7%88%20%EB%B3%80%EA%B2%BD%20%EB%8C%80%EC%9D%91/04_schema_change_response.md

5) 슬로우 쿼리 튜닝.
- [슬로우 쿼리 개선 사례][slow]

  [slow]: https://github.com/yskim1230/DBA_portfolio/blob/main/%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C/5.%20%EC%8A%AC%EB%A1%9C%EC%9A%B0%EC%BF%BC%EB%A6%AC%2C%20%EB%A1%9C%EA%B7%B8%EB%A5%BC%20%ED%86%B5%ED%95%9C%20%ED%8A%9C%EB%8B%9D%20%EC%82%AC%EB%A1%80/%EC%8A%AC%EB%A1%9C%EC%9A%B0%20%EC%BF%BC%EB%A6%AC%20%ED%8A%9C%EB%8B%9D.md


## 5. 증적 자료 규칙
각 문서는 배경 절차 검증 결과 리스크 롤백 순서로 작성했습니다.
스크립트와 로그 캡처는 문서와 같은 폴더 또는 하위 evidence 폴더에 둡니다.
