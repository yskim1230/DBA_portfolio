# 온프레미스 포트폴리오 목차

## 1. 개요
IDC 기반 운영 환경에서 MSSQL과 서비스별 MySQL MariaDB를 운영하며 수행한 업무를 정리했습니다.
가용성, 백업 복구, 보안 감사, 성능 튜닝, 모니터링과 장애 대응에 집중합니다.

## 2. 먼저 보면 좋은 순서
1) 대표사례 3건
2) 백업 복구 표준과 복구 리허설
3) HA 장애 대응 런북 MHA 포함
4) 보안 감사 ISMS 증적
5) 성능 튜닝 실행계획 인덱스 전략
6) 모니터링 관측 MaxGauge PMM 알람

## 3. 대표사례
- [SQL서버-마이그레이션](https://github.com/yskim1230/DBA_portfolio/blob/main/%EC%98%A8%ED%94%84%EB%A0%88%EB%AF%B8%EC%8A%A4/3.%20%EC%9E%90%EC%9B%90%EA%B4%80%EB%A6%AC%2C%20%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98%20%EC%9E%91%EC%97%85%2C%20MHA%20%EA%B5%AC%EC%84%B1/SQL%EC%84%9C%EB%B2%84%20%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98.md)

- [인덱스 튜닝](https://github.com/yskim1230/DBA_portfolio/blob/main/%EC%98%A8%ED%94%84%EB%A0%88%EB%AF%B8%EC%8A%A4/2.%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%20%ED%8A%9C%EB%8B%9D%2C%20%EB%A6%AC%EB%B9%8C%EB%93%9C%2C%20%EC%BF%BC%EB%A6%AC%20%EA%B0%9C%EC%84%A0%20%EB%B0%8F%20%EA%B0%80%EC%9D%B4%EB%93%9C/%ED%8A%9C%EB%8B%9D%EC%82%AC%EB%A1%805.%20%ED%8B%B0%EC%85%80%ED%8C%8C%20%EC%B4%88%EB%93%B1%20%EC%9D%8C%EC%95%85%20%EC%B9%B4%ED%85%8C%EA%B3%A0%EB%A6%AC%EB%B3%84%20%EC%BD%98%ED%85%90%EC%B8%A0%20%EC%A1%B0%ED%9A%8C%EC%88%98%2C%EC%82%AC%EC%9A%A9%EC%9E%90%EC%88%98%20%EC%A7%91%EA%B3%84%20%EC%BF%BC%EB%A6%AC%20%EC%86%8D%EB%8F%84%20%EC%A0%80%ED%95%98%20%ED%95%B4%EA%B2%B0.md)

- [SQL서버-인덱스리빌드](https://github.com/yskim1230/DBA_portfolio/blob/main/%EC%98%A8%ED%94%84%EB%A0%88%EB%AF%B8%EC%8A%A4/2.%20%EC%9D%B8%EB%8D%B1%EC%8A%A4%20%ED%8A%9C%EB%8B%9D%2C%20%EB%A6%AC%EB%B9%8C%EB%93%9C%2C%20%EC%BF%BC%EB%A6%AC%20%EA%B0%9C%EC%84%A0%20%EB%B0%8F%20%EA%B0%80%EC%9D%B4%EB%93%9C/%EC%9D%B8%EB%8D%B1%EC%8A%A4%EB%A6%AC%EB%B9%8C%EB%93%9C.md)


## 4. 목차
1) 인프라 요약
- ./01_인프라요약/ (https://github.com/yskim1230/DBA_portfolio/tree/main/%EC%98%A8%ED%94%84%EB%A0%88%EB%AF%B8%EC%8A%A4/1.%20%EC%9D%B8%ED%94%84%EB%9D%BC%20%EA%B5%AC%EC%A1%B0)

2) 고가용성 HA 장애 대응
- ./02_HA_장애대응/

3) 백업 복구 최신화
- ./03_백업복구/

4) 성능 운영 최적화 튜닝 리빌드
- ./04_성능튜닝/

5) 보안 감사 ISMS
- ./05_보안감사/

6) 모니터링 관측
- ./06_모니터링/

7) 마이그레이션 업그레이드
- ./07_마이그레이션/

## 5. 증적 자료 규칙
각 문서는 배경 절차 검증 결과 리스크 롤백 순서로 작성했습니다.
스크립트 로그 캡처는 문서와 같은 폴더 또는 하위 evidence 폴더에 둡니다.
