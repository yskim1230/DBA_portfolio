# 튜닝 사례
## 개요 : 메인 db sql 서비스가 1초 정도 중단
## 내역 : 오류 로그에 쌓인 기록은 chunjae_pass.dbo.USP_New_Member_EasyJoin 실행 시 EXCEPTION_ACCESS_VIOLATION(메모리 접근과 관련된 문제) 오류 발생으로, 순간적인 메모리 사용량 초과(서버내 메모리 할당 기준 90% 초과)로 인해 메모리 오류가 발생한 것으로 보임
## 발생 원인 분석
- SQL Server 2019 RTM 버전 (CU 미적용)
- EXISTS + AND + Linked Server View 조합 시 내부 처리 방식 문제
  <img width="900" height="610" alt="image" src="https://github.com/user-attachments/assets/1e7296e3-a1d7-4ee1-91a6-837e04002eeb" />
  <img width="900" height="529" alt="image" src="https://github.com/user-attachments/assets/641783fb-aa23-481b-b601-37374e0cc4db" />

- 링크드 서버 처리 최적화 필요

[서버 내 오류내역 확인]
<img width="1329" height="748" alt="image" src="https://github.com/user-attachments/assets/b19bc2ff-0700-4d07-978b-f096970ef60c" />

[오류 로그 파일 첨부 사진]
<img width="2063" height="926" alt="image" src="https://github.com/user-attachments/assets/af324905-b921-49e9-8752-a4e38977e089" />

## 추가 우려 사항
DB 에서 실행되는 USP_New_Member_EasyJoin 은 USP_MemberPoint_Ins 를 실행하고 최종적으로 USP_Schedule_MemberPoint_RP 실행하는 중첩 프로시저 구조.
최종 실행되는 USP_Schedule_MemberPoint_RP 에서 부하가 발생된다고 판단했으며, 해당 SP 실행시 과도한 연산 작업.
<img width="1176" height="798" alt="image" src="https://github.com/user-attachments/assets/29730d97-ee7d-48ee-8b72-985a849aeeef" />

<img width="1176" height="798" alt="image" src="https://github.com/user-attachments/assets/4e42dec4-b020-43dc-85e0-674387be8136" />

실제 USP_Schedule_MemberPoint_RP 내에서 우려 쿼리 문장은 아래와 같습니다.
<img width="1330" height="746" alt="image" src="https://github.com/user-attachments/assets/bd495cfe-f5b0-4770-ace7-0320b958449b" />

## 조치 사항
USP_New_Member_EasyJoin 저장 프로시저 리팩토링 변경 내역서 작성 및 운영팀에 적용 요청 내역 전달
<img width="670" height="849" alt="image" src="https://github.com/user-attachments/assets/a5cdcdd1-0a48-4607-9726-520381371e44" />

[적용된 개선 쿼리]
<img width="900" height="533" alt="image" src="https://github.com/user-attachments/assets/50a01006-2660-4ccb-a963-d7d00cbfd7b9" />

### 개선 쿼리 적용 후 추가 오류 발생하지 않음을 확인


## 추가 조치
아래와 같이 읽기횟수 감소를 위해 인덱스 생성을 진행했습니다.
<img width="1329" height="833" alt="image" src="https://github.com/user-attachments/assets/dcbbfc9e-cb30-48a9-8bb3-011d1dbaf3a6" />


