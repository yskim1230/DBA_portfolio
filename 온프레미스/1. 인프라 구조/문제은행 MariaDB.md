# testbank (MariaDB) 스키마 서비스 파악 요약

## 1. 결론: DB 오브젝트 로직 없음, 애플리케이션 SQL 직접 수행 구조

* `testbank` 스키마에는 **프로시저/함수/트리거/뷰/이벤트가 존재하지 않음**.
* 실제 동작 로직은 **애플리케이션에서 직접 SELECT/INSERT/UPDATE를 수행**하는 패턴이 `performance_schema digest`로 확인됨.
* 따라서 서비스 파악은 “오브젝트”가 아니라 **테이블 모델 + 쿼리 패턴(Top digest)** 중심으로 진행해야 함.

---

## 2. 스키마 목적(기능 범위) 추정

테이블 구성과 컬럼 코멘트 기준으로, `testbank`는 “문제은행 전체”가 아니라 아래 기능에 집중된 **‘시험지(세팅지) 관리 서브도메인’**으로 보임.

* 시험지(세팅지) 생성/목록/삭제
* 시험지에 문항을 매핑(대용량 핵심)
* 시험지 HTML(문제/정답/전체) URL 저장 및 갱신
* 다운로드/편집 클릭 로그 적재
* 오류 신고(문항 오류 신고) 및 해당 페이지 접근을 위한 임시 패스워드

---

## 3. 데이터 규모(운영 중요도)

| 테이블                        |      Rows | Size(MB) | 의미                   |
| -------------------------- | --------: | -------: | -------------------- |
| t_question_paper_mapping   | 6,820,975 |   651.83 | 시험지-문항 매핑(핵심 대용량)    |
| t_paper_info               |   244,888 |   106.63 | 시험지(세팅지) 마스터         |
| setting_paper_download_log | 1,159,819 |    58.58 | 다운로드 로그(적재 중심)       |
| setting_paper_edit_hit_log |   282,228 |    13.52 | 편집 클릭 로그(적재 중심)      |
| t_subject_info             |       170 |     0.05 | 과목/교육과정 메타           |
| t_question_error           |        88 |     0.03 | 문항 오류 신고             |
| t_code                     |         6 |     0.02 | 공통코드                 |
| t_tmp_main_pwd             |         2 |     0.02 | 오류 신고 페이지 접근 임시 패스워드 |

운영상 병목/부하 포인트는 `t_question_paper_mapping`에 집중됨.

---

## 4. 테이블별 역할 정의(DDL 기반)

### 4.1 t_paper_info (시험지 정보 테이블)

* PK: `id` (AUTO_INCREMENT)
* 주요 컬럼

  * `subject_id` (varchar(15)) : 교재코드
  * `text_name` : 교재명(+교육과정)
  * `area_code/area_name` : 과목코드/과목명
  * `title` : 시험지 제목
  * `question_count` : 문제 수
  * `contentHml/questionHml/answerHml` (longtext) : 전체/문제/정답해설 URL 저장
  * `user_id` : T셀파 유저아이디
  * `delete_yn` : 소프트 삭제 플래그
  * `created_at/updated_at`

특징:

* URL 3종을 저장하고 이후 UPDATE로 갱신하는 패턴이 digest로 확인됨(아래 6절 참고).
* 인덱스는 PK만 존재 → `user_id`, `delete_yn`, `subject_id` 조건 검색이 많으면 성능 리스크. 하지만 현재는 리스크가 발견되지 않음

---

### 4.2 t_question_paper_mapping (문항-시험지 매핑)

* PK: `id` (AUTO_INCREMENT)
* Unique: `(paper_id, question_id)` : 시험지 내 동일 문항 중복 매핑 방지
* 주요 컬럼

  * `paper_id` : 시험지 ID (t_paper_info.id로 조인되는 것이 자연스러움)
  * `question_id` : 문항 ID (문항 마스터 테이블은 이 스키마에 없음 = 다른 DB/스키마일 가능성)
  * `question_no` : 시험지 내 문항 순번
  * `delete_yn` : 소프트 삭제
  * `created_at/updated_at`

특징/주의:

* 인덱스가 사실상 PK와 (paper_id, question_id)뿐.
* `question_id` 단독 조회/삭제가 잦다면 별도 인덱스 필요 가능성.
* `paper_id IN (?)`로 delete_yn 갱신하는 쿼리가 존재(digest 확인).

---

### 4.3 t_subject_info (초등 과목 정보)

* PK: `subjectId`
* 과목/교육과정/학년/학기 등의 메타
* `status`로 활성/비활성 필터링 존재(digest 확인)

---

### 4.4 t_question_error (오류신고 테이블)

* 문항 오류 신고 + 첨부파일 저장
* `question_id`, `subject_name`, `type`, `content`
* `tsherpa_user_id`로 신고자 관리
* `updated_at`은 ON UPDATE로 자동 갱신
* 목적이 명확: “오류신고 기능”이 DB에 구현되어 있음(단, 로직은 앱)

---

### 4.5 setting_paper_download_log / setting_paper_edit_hit_log

* 둘 다 PK는 `seq` AUTO_INCREMENT만 존재
* `setting_paper_id` + `user_id` + (download_log는 `differentiation` 추가)
* 적재량이 큰 로그 테이블인데 **보조 인덱스가 전무**

  * 특정 기간/유저/시험지 기준 조회가 많아지면 즉시 병목 가능

---

### 4.6 t_code (공통코드)

* `group`(예약어 성격의 단어이므로 백틱으로 컬럼 사용) + `code` + `code_name`
* `delete_yn`
* digest에서 `WHERE delete_yn=? AND group=?` 조회 확인됨
* PK 외 보조 인덱스 없음

---

### 4.7 t_tmp_main_pwd (임시 오류 신고 목록 페이지 접속 패스워드)

* 테이블 코멘트상 “임시 오류 신고 목록 페이지 접속 패스워드”
* 실제로 password를 varchar(50) 평문 저장 구조로 보임
* 보안 관점에서 민감(암호화/해시/접근통제 여부 확인 필요)

---

## 5. 엔티티 관계(ERD 수준 추정)

현재 스키마 내부에서 확정 가능한 관계는 아래와 같습니다.

* `t_paper_info (id)` 1 ── N `t_question_paper_mapping (paper_id)`
* `t_question_paper_mapping (question_id)` ── (외부 문항 마스터 테이블로 연결됨 가능성 높음)
* `t_paper_info.subject_id (varchar)` ↔ `t_subject_info.subjectId (int)` 는 **타입이 달라 직접 FK 관계로 보기 어려움**

  * digest에서는 `t_paper_info.subject_id NOT IN (SELECT subjectId FROM t_subject_info)` 형태가 존재하므로, 실제로는 비교/캐스팅 이슈가 잠재되어 있음(아래 6절).

---

## 6. 실제 쿼리 패턴(Top Digest 기반)과 서비스 흐름

### 6.1 시험지 목록 조회(유저별)

가장 시간이 많이 소요된 digest가 아래 형태입니다.

* 유저별 시험지 목록 조회
* `delete_yn` 필터
* `subject_id IN/NOT IN (t_subject_info.subjectId)`로 분기
* `ORDER BY id DESC`
* `date_format(created_at, ?)`로 출력 포맷 변환

핵심 포인트:

* `t_paper_info`에 **(user_id, delete_yn, id)** 같은 인덱스가 없으면, 유저 데이터가 늘수록 정렬/필터 비용이 급증할 수 있음.
* `subject_id`(varchar) vs `subjectId`(int) 비교는 데이터 정합/성능 리스크(캐스팅 발생 가능, 조건 최적화 저해 가능).

---

### 6.2 매핑 테이블 전체 조회(SQL_NO_CACHE)

`SELECT SQL_NO_CACHE ... FROM t_question_paper_mapping`이 평균 3.62초로 잡혀 있습니다(5회).
이건 보통:

* 테이블 전체 스캔
* 또는 캐시 무시 + 대용량 결과셋 전송
  중 하나입니다.

즉시 점검 대상:

* 해당 쿼리가 “관리자 페이지 전체 다운로드” 같은 기능인지 확인 필요
* 운영 중 반복되면 DB I/O를 크게 소모할 가능성이 큼

---

### 6.3 시험지 URL 갱신 흐름

* 시험지 생성: `INSERT INTO t_paper_info (...)`
* 이후 URL 3종 갱신: `UPDATE t_paper_info SET contentHml=?, questionHml=?, answerHml=?, updated_at=NOW() WHERE id=?`
* 상세 조회: `SELECT id, contentHml, questionHml, answerHml ... WHERE delete_yn=? AND id=?`

즉, “시험지 생성 → HTML 생성/업로드 → URL 업데이트”의 2단계 프로세스가 앱단에 존재하는 구조로 보입니다.

---

### 6.4 시험지 삭제(소프트 삭제) 및 매핑 삭제(소프트 삭제)

* `UPDATE t_question_paper_mapping SET delete_yn=?, updated_at=NOW() WHERE paper_id IN (?)`
* 시험지/매핑 모두 delete_yn 기반 소프트 삭제 운영

주의:

* `paper_id IN (?)`가 대량 리스트로 들어오면, 인덱스 설계/쿼리 구성에 따라 잠금/부하가 커질 수 있음.

---

### 6.5 로그 적재

* 다운로드 로그: `INSERT INTO setting_paper_download_log (...)` 다수(1225회)
* 편집 히트 로그도 유사
* 인덱스가 PK만이라 조회 목적이 생기는 순간 병목이 빠르게 발생 가능

---

## 7. 운영 관점에서 부족한 부분

1. `t_paper_info` 조회 핵심 조건(`user_id`, `delete_yn`, `id desc`)에 맞는 보조 인덱스 부재
2. `subject_id`(varchar)와 `subjectId`(int) 비교로 인한 성능/정합 리스크
3. `t_question_paper_mapping` 전체 스캔성 쿼리(SQL_NO_CACHE)가 존재
4. 로그 테이블 2종이 대용량 적재인데 보조 인덱스 없음(향후 통계/감사 요구 시 즉시 문제화)
5. `t_tmp_main_pwd`의 평문 password 저장 가능성(보안 정책 이슈)

---

1. `t_paper_info`에서 실제 목록 조회가 느린 이유(인덱스 부재) 검증용

```sql
EXPLAIN
SELECT id, subject_id, text_name, area_code, area_name, title, question_count, user_id, created_at
FROM t_paper_info
WHERE user_id = '샘플유저' AND delete_yn='N'
ORDER BY id DESC
LIMIT 50;
```

2. 매핑 대용량 스캔성 쿼리 확인용

```sql
EXPLAIN SELECT SQL_NO_CACHE id, paper_id, question_id, question_no, delete_yn, created_at, updated_at
FROM t_question_paper_mapping;
```

3. `subject_id` 타입 미스매치 영향 확인(캐스팅 여부)

```sql
EXPLAIN
SELECT id
FROM t_paper_info
WHERE subject_id IN (SELECT subjectId FROM t_subject_info);
```
