# 튜닝 사례: 고빈도 조회 쿼리의 불필요 정렬 및 임시테이블 제거(즉시 적용) 및 인덱스 최적화는 데이터 성장 시점에 단계적 적용 권고

## 1. 배경

* 상품과 브랜드 연동 정보를 조회하는 특정 쿼리가 서비스 트래픽에서 호출 빈도가 매우 높아 누적 부하 관점에서 개선 대상으로 선정됨
* 1주일 관측치 기준 단건 지연은 매우 짧지만(마이크로초 단위) 총 호출량이 크므로 작은 비효율도 누적 비용으로 전환되는 유형

### 1주일 지표(PMM Query Analytics)
```
Metric: Query Count (sum) 6.85M
Metric: QPS 11.32
Metric: Count share 13.94%
Metric: Avg latency 203.60 us
Metric: Total time 0:23:14
Metric: Time share 2.17%
```
<img width="1880" height="959" alt="0  문제은행 과도한 호출쿼리 내역 확인(1주일통계)" src="https://github.com/user-attachments/assets/3aa1dfa8-f4ca-42f4-9ef0-cf70a0912b83" />


## 2. AS-IS 쿼리 및 문제점

### AS-IS

```sql
SELECT
  c.title,
  c.brand_goods_code,
  c.title_eng,
  l.*,
  d.charge
FROM fm_brand_link l
INNER JOIN fm_brand c ON l.category_code = c.category_code
INNER JOIN fm_goods g ON l.goods_seq = g.goods_seq
LEFT JOIN fm_provider p ON g.provider_seq = p.provider_seq
LEFT JOIN fm_provider_charge d
  ON (l.category_code = d.category_code AND p.provider_seq = d.provider_seq)
WHERE l.goods_seq = '189663'
GROUP BY l.category_link_seq;
```

<img width="1943" height="487" alt="1  개선전 explain" src="https://github.com/user-attachments/assets/3eecfd31-0d94-4c2d-a670-5911b3d1683e" />



### 문제 인식

* 실행계획 Extra에 Using temporary, Using filesort 발생
  GROUP BY로 인한 임시테이블과 정렬은 고빈도 쿼리에서 누적 CPU 비용으로 이어질 수 있음
* fm_provider는 SELECT 결과에 사용되지 않는 중간 조인으로 조인 단순화 여지가 있음
* fm_provider_charge는 조인 조건이 provider_seq와 category_code의 복합 조건인데 단일 인덱스만 사용될 가능성이 있어 향후 데이터 증가 시 리스크가 될 수 있음



## 3. 개선 방향 및 TO-BE 적용(즉시 적용 범위)

### 핵심 전략(운영 리스크가 낮은 범위부터)

1. 불필요한 GROUP BY 제거로 임시테이블 및 파일소트 제거
2. 불필요한 중간 조인 제거로 조인 단계 축소
3. 인덱스 최적화는 데이터 크기와 분포에 따라 단계적으로 적용하며 현 상태에서는 보류 가능


### 적용전 데이터 정합성 확인


 
[Group by 제거 전 데이터 단일성 확인]

<img width="534" height="118" alt="2  개선 전 category_link_seq 중복확인" src="https://github.com/user-attachments/assets/63ace8ff-bf9b-4026-bc9c-07aa00d3bb26" />
<img width="522" height="90" alt="3  개선 전 provider_charge가 provider+category 조합당 다건인지 확인" src="https://github.com/user-attachments/assets/aa1afd44-973f-4880-93b3-6ebe14cdbb88" />




### TO-BE 쿼리

```sql
SELECT
  c.title,
  c.brand_goods_code,
  c.title_eng,
  l.*,
  d.charge
FROM fm_brand_link l
JOIN fm_brand c
  ON c.category_code = l.category_code
JOIN fm_goods g
  ON g.goods_seq = l.goods_seq
LEFT JOIN fm_provider_charge d
  ON d.category_code = l.category_code
 AND d.provider_seq = g.provider_seq
WHERE l.goods_seq = 189663;
```

<img width="1859" height="387" alt="4  개선 후 쿼리 explain" src="https://github.com/user-attachments/assets/82dada8a-132c-4ec6-b4dd-1873e504ff9c" />



## 4. 적용 결과(실행계획 관점)

* Using temporary 및 Using filesort 제거 확인
* 조인 단계 감소로 쿼리 구조 단순화
* fm_brand_link는 ix_goods_seq_category_code 인덱스 기반으로 범위를 좁히고 fm_goods는 PK const 형태로 단건 접근

이 사례는 1회 실행시간을 크게 줄이는 목적이라기보다 고빈도 쿼리에서 불필요한 정렬과 임시테이블을 제거해 누적 리소스 소비를 줄이는 운영형 튜닝 사례로 정리 가능

## 5. 인덱스 개선(보류) 판단 근거

### fm_provider_charge 현황(운영 데이터 기준)

* Row 수 491
* 인덱스 PRIMARY(charge_seq), provider_seq(단일), link(단일)
* provider_seq의 Cardinality가 rows에 근접(약 482/491)하여 현 시점 분포에서는 단일 인덱스만으로도 후보 row가 거의 1건 수준으로 좁혀질 가능성이 높음

<img width="1405" height="534" alt="5  fm_provider_charge 테이블 상태 확인(이 근거로 인덱스 추가는 보류)" src="https://github.com/user-attachments/assets/fab2a41a-a71e-46b6-9d87-b15a46eade6b" />


### 결론

* 현재 데이터 규모와 분포 기준으로는 복합 인덱스(provider_seq, category_code) 추가의 체감 성능 이득이 제한적일 수 있음
* 다만 향후 fm_provider_charge 데이터가 증가하거나 provider당 category 매핑이 다건화되는 경우, Using where 비용이 누적되며 병목 후보로 전환될 수 있으므로 성장 시점에 인덱스 추가를 권장함

## 6. 추후 적용 권장안(데이터 성장 및 분포 변화 시)

### 권장 인덱스(추후)

```sql
ALTER TABLE fm_provider_charge
  ADD INDEX ix_provider_seq_category_code (provider_seq, category_code);
```

### 적용 트리거(운영 기준)

* fm_provider_charge row 수가 의미 있게 증가(예: 수만에서 수십만 단위)
* 특정 provider에 매핑되는 category가 다건화되어 provider_seq 단일 인덱스로 후보 row가 커지는 경우
* PMM에서 해당 쿼리의 Rows examined 증가, CPU 상승, latency 상승이 관측되는 경우
* Replica, 복제 지연, DDL 리스크를 감안해 변경 윈도우 확보가 가능한 시점

## 7. 마무리 요약

* 고빈도 쿼리에서 GROUP BY로 인한 임시테이블 및 파일소트를 제거하고 불필요 조인을 줄여 누적 비용을 낮춤
* charge 테이블은 현재 소규모라 복합 인덱스는 효과 대비 리스크와 필요성이 낮아 보류했고 데이터 증가 시점에 (provider_seq, category_code) 인덱스를 단계적으로 적용하는 전략으로 정리함
