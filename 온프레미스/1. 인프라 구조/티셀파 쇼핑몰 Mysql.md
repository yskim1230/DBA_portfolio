# T셀파 쇼핑몰 (MySQL 5.X) 스키마 서비스 파악 요약

## 1) 이 DB의 핵심 작동 구조(한 줄 요약)

**상품(goods)–카테고리–브랜드 관계 변경과 상품 코드/옵션 코드 변경을 트리거로 감지해서,**

1. **카테고리/브랜드별 상품 수 집계(fm_count)**
2. **카테고리–브랜드 매핑 테이블(fm_category_brand) 재구성**
3. **옵션/추가옵션의 바코드(=상품코드+옵션코드) 파생 컬럼 생성(full_barcode/sub_full_barcode)**
4. **물류/SCM 관련 여러 테이블의 goods_code를 옵션 바코드로 동기화(fm_scm_*)**
5. **상품 랭킹 요약값(ranking_point) 갱신(fm_goods_list_summary)**
   을 수행합니다.

---

## 2) 트리거별 역할 맵(데이터 흐름 관점)

### A. 카테고리/브랜드 “상품 수 집계” 및 “카테고리-브랜드 매핑” 유지

* 대상: `fm_brand_link`, `fm_category_link`
* 파생 테이블:

  * `fm_count` : `kind`(brand/category) + `code` 단위로 `totalcount` 누적
  * `fm_category_brand` : 카테고리와 브랜드 조합(필터/검색/리스트 페이지용으로 보임)

**브랜드 링크 변화 (`fm_brand_link`)**

* `tr_fm_brand_link_insert` (AFTER INSERT)

  * 특정 상품이 “노출 대상”이면 브랜드 카운트 +1
  * 해당 상품이 연결된 모든 카테고리에 대해 (category_code, brand_code) 매핑 생성/치환
  
* `tr_fm_brand_link_update` (AFTER UPDATE)
  * OLD 브랜드 카운트 -1, NEW 브랜드 카운트 +1
  * OLD 브랜드의 totalcount가 1 미만이면 `fm_category_brand`에서 제거
  * 매핑 REPLACE 수행, totalcount<1 정리
  
* `tr_fm_brand_link_delete` (AFTER DELETE)
  * 노출 대상이면 카운트 -1
  * totalcount<1이면 매핑 제거 및 fm_count 정리

**카테고리 링크 변화 (`fm_category_link`)**

* `tr_fm_category_link_insert/update/delete`

  * 위 브랜드 로직과 거의 대칭 구조로, 카테고리 카운트를 증감하고 `fm_category_brand` 매핑을 유지합니다.

**노출 대상 판정 로직(중요)**

* 트리거에서 “집계에 포함할 상품인지”를 아래 조건으로 판단합니다.

  * `goods_view='look'` 이거나
  * `display_terms='AUTO'` 이고, `display_terms_begin <= CURDATE() <= display_terms_end`
* 즉, **현재 노출 가능한 상품만 집계/매핑에 반영**하려는 의도입니다.

---

### B. 상품 노출 상태(goods_view) 변경 시, 집계 재조정

* `tr_fm_goods_update_view` (AFTER UPDATE ON `fm_goods`)
* `goods_view` 값이 바뀌면, 해당 상품이 물려 있는 **모든 카테고리/브랜드 코드 목록**을 가져와서:

  * `look`으로 바뀌면 `fm_count`에 +1
  * `notLook`으로 바뀌면 `fm_count`에서 -1
  * 그리고 totalcount<1 데이터 정리

이 트리거는 “상품의 노출/비노출”만 바뀌어도 **연관된 카테고리/브랜드 집계를 즉시 맞춰주는 역할**입니다.

---

### C. 상품 코드(goods_code) 기반 “바코드(파생 컬럼)” 생성/유지

* 목적: 옵션 바코드 = `goods_code + optioncode1..5` 를 **항상 최신 상태로 유지**
* 대상 테이블: `fm_goods`, `fm_goods_option`, `fm_goods_suboption`

**1) 상품 생성 시**

* `tr_fm_goods_insert` (AFTER INSERT ON `fm_goods`)

  * 해당 goods_seq의 모든 옵션/추가옵션에 대해 `full_barcode`, `sub_full_barcode`를 일괄 생성

**2) 상품 코드 변경 시**

* `tr_fm_goods_update` (AFTER UPDATE ON `fm_goods`)

  * `goods_code`가 바뀌면 옵션/추가옵션의 바코드를 일괄 재계산

**3) 옵션/추가옵션 INSERT/UPDATE 시**

* `tr_fm_goods_option_insert` (BEFORE INSERT)
* `tr_fm_goods_option_update` (BEFORE UPDATE)

  * INSERT/UPDATE 시점에 `NEW.full_barcode`를 계산해 컬럼에 저장
* `tr_fm_goods_suboption_insert/update` (BEFORE INSERT/UPDATE)

  * `NEW.sub_full_barcode` 계산

---

### D. 옵션 바코드 변경을 SCM(물류) 관련 테이블에 전파

* `tr_fm_goods_option_full_barcode_insert` (AFTER INSERT)
* `tr_fm_goods_option_full_barcode_update` (AFTER UPDATE, 바코드가 실제로 변한 경우)

여기서 핵심은:

* `fm_scm_autoorder_order`, `fm_scm_ledger*`, `fm_scm_order_goods`, `fm_scm_stock_*`, `fm_scm_warehousing_goods` 등 다수 테이블에
* `option_seq`를 키로 잡고 `goods_code = NEW.full_barcode`로 업데이트합니다.

즉, 이 DB는 **SCM 영역이 “상품코드(goods_code)”를 옵션 단위 바코드로 쓰는 구조**이고, 그 정합성을 트리거로 강제합니다.

---

### E. 상품 리스트 랭킹 요약값 갱신

* `tr_fm_goods_update` 내 로직:

  * `page_view`, `review_count`, `purchase_ea` 중 하나라도 바뀌면
  * `fm_goods_list_summary.ranking_point = purchase_ea + page_view + review_count` 로 갱신

즉, 리스트/검색 정렬용 점수를 **요약 테이블에 미리 계산해 저장**합니다.

---

## 3) 이 구조에서 “당신이 반드시 알아야 할 것” 체크리스트

### (1) 원본 vs 파생(권위 데이터) 구분

* **원본(트랜잭션의 근원)**: `fm_goods`, `fm_category_link`, `fm_brand_link`, `fm_goods_option`, `fm_goods_suboption`
* **파생/캐시(정합성은 트리거에 의존)**: `fm_count`, `fm_category_brand`, `fm_goods_list_summary.ranking_point`, 여러 `fm_scm_*`.`goods_code`

운영/장애 대응 시 “어디가 틀렸을 때 어디를 재생성해야 하는가”가 이 구분에서 결정됩니다.

### (2) 정합성 깨질 때의 “재빌드(복구) 전략”이 필요

트리거 기반 시스템은 아래 상황에서 불일치가 생길 수 있습니다.

* 트리거 비활성/누락된 데이터 이관
* 대량 배치 업데이트/데이터 수정(운영자 수동 작업 포함)
* 복제/리플리케이션 지연 또는 binlog 포맷 문제
* 트리거 내부 로직 버그로 음수 카운트/중복 매핑 발생

따라서 실무적으로는:

* `fm_count` 재생성 SQL(카테고리/브랜드별 “현재 노출 상품 수”를 다시 집계)
* `fm_category_brand` 재생성 SQL(현재 링크 테이블 기준으로 재구성)
* 옵션 바코드 재생성 SQL(옵션/추가옵션 전체 리빌드)
* `fm_scm_*` goods_code 재동기화 SQL(옵션 바코드 기준으로 일괄 동기화)
  같은 “복구 스크립트”를 런북에 포함시키는 것이 운영 안정성에 중요합니다.

### (3) 성능/락/부하 포인트

* 트리거는 **행 단위(FOREACH ROW)** 로 동작합니다.
  대량 INSERT/UPDATE 시 트리거가 같은 비율로 폭증합니다.
* 특히 `fm_goods_option_full_barcode_*`는 옵션 1건 변경이 **SCM 관련 다수 테이블 UPDATE**로 이어져 락/IO가 커질 수 있습니다.
* `fm_goods_update_view`는 `GROUP_CONCAT + WHILE 루프`로 문자열 파싱을 하므로, 상품이 많은 카테고리/브랜드에 묶여 있으면 비용이 커질 수 있습니다.

### (4) 이식/복구/감사 관점에서 중요한 메타정보

* 모든 트리거에 `DEFINER=\`gabia`@`172.21.112.62``가 붙어 있습니다.
  다른 환경으로 dump/restore 시 definer 계정이 없으면 생성 실패할 수 있어, 이관 절차에 주의가 필요합니다.

### (5) 코드 레벨에서 “잠재 리스크”로 보이는 지점(검토 권장)

* 일부 UPDATE에서 `fm_count`를 감소시키는 구문이 `kind` 조건 없이 실행되는 형태가 섞여 있어(카테고리/브랜드 모두에 영향 가능) **카운트 오염 가능성**이 있습니다.
  운영 중 `fm_count.totalcount`가 음수/비정상으로 내려가는 사례가 있다면 이 포인트를 1순위로 의심해야 합니다.
* 트리거에서 `@변수`(세션 변수)를 사용합니다. 트리거 내부에서는 보통 `DECLARE`로 지역 변수를 쓰는 것이 더 안전/명확합니다(기능상 큰 문제는 없더라도 유지보수성 이슈).

---

