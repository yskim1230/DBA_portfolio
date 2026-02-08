#  MSCS 기반 SQL Server Failover/Failback 전환 테스트 결과 (증빙)

Windows Failover Cluster(MSCS) 환경에서 SQL Server(MSSQLSERVER)와 DTC 리소스의 Failover/Failback 전환 및 전환 후 DML(INSERT/SELECT/DELETE) 검증 결과를 증빙 중심으로 정리

---

# 1. 테스트 개요
- **목적**: SQL Server 서비스 연속성 확보를 위해 MSCS 환경에서 **Failover/Failback 전환 가능 여부** 및 전환 후 **DB 동작 정상성(DML)**을 검증
- **테스트 범위**
  - 클러스터 Role 소유자(Owner Node) 전환 확인
  - 공유 디스크(데이터/로그/DTC/쿼럼) 소유자 전환 및 Online 상태 확인
  - 전환된 노드에서 SQL Server 서비스 기동 상태 확인
  - 전환 후 DML 테스트(INSERT/SELECT/DELETE) 수행

---

# 2. 환경 정보
## 2.1 노드
- Active(전환 전): `cjdb01p` (IP: `192.168.170.11`)
- Standby(전환 대상): `cjdb02p` (IP: `192.168.170.12`)

## 2.2 클러스터 Role
- `cjdbdtc01` (DTC)
- `SQL Server (MSSQLSERVER)` (DB 서비스)

## 2.3 공유 디스크(클러스터 자원)
- `MSSQL (S:)`  : 데이터
- `DB_Log (F:)` : 로그
- `DTC (U:)`
- `Quorum (Q:)`

---

# 3. 성공 기준(DoD)
- [ ] Role(DTC, SQL Server)의 **Owner Node 전환**이 정상 반영된다.
- [ ] 공유 디스크(S:, F:, U:)가 전환된 노드에서 **Online 유지**된다.
- [ ] 전환된 노드에서 **SQL Server 서비스가 실행 중**이다.
- [ ] 전환 후 **DML(INSERT/SELECT/DELETE)**이 정상 수행된다.
- [ ] Failback 후에도 동일하게 정상 동작한다.

---

# 4. 테스트 수행 및 결과(증빙 기반)

## 4.1 Failover 전 상태 (Owner: cjdb01p)
- Role 소유자 노드가 `cjdb01p`로 확인됨
- Disk 자원이 Online이며 소유자 노드가 `cjdb01p`로 확인됨

**Evidence**
- Role Owner(전환 전): `cjdb01p`  
<img width="602" height="103" alt="1" src="https://github.com/user-attachments/assets/25556063-d229-4061-99fa-2a1436e1e603" />

- Disk Owner/Online(전환 전): `cjdb01p`  
<img width="816" height="162" alt="2" src="https://github.com/user-attachments/assets/a56d5f7d-59cc-4f1c-a99e-b270c66836f5" />


---

## 4.2 Failover 수행 (cjdb01p → cjdb02p)
- Role 소유자 노드가 `cjdb02p`로 전환됨
- Disk 자원이 Online 유지되며 소유자 노드가 `cjdb02p`로 전환됨

**Evidence**
- Role Owner(FAILOVER 후): `cjdb02p`  
<img width="615" height="127" alt="3" src="https://github.com/user-attachments/assets/7a10d97f-379c-435c-aa5c-8ab9777716a5" />

- Disk Owner/Online(FAILOVER 후): `cjdb02p`  
<img width="837" height="208" alt="6" src="https://github.com/user-attachments/assets/4e648738-1f6f-4c00-abe8-4853c45d7d08" />
  

---

## 4.3 Failover 후 DB 동작 검증 (cjdb02p에서 DML)
- `192.168.170.12(cjdb02p)`에서 SQL Server 관련 서비스가 **실행 중**으로 확인됨
- SSMS에서 테스트 테이블 대상으로 DML 수행 결과 **정상**
  - INSERT: (1개 행 적용됨)
  - SELECT: 정상 조회
  - DELETE: (1개 행 적용됨)

**Evidence**
- (cjdb02p) 서비스 실행 상태  
  ![Services on cjdb02p](./evidence/04_services_on_cjdb02p.png)
- (cjdb02p) DML 테스트 성공  
  ![DML on cjdb02p](./evidence/05_dml_on_cjdb02p.png)

---

## 4.4 Failback 수행 (cjdb02p → cjdb01p)
- Role 소유자 노드가 `cjdb01p`로 원복됨
- 원복 후 `192.168.170.11(cjdb01p)`에서 DML 검증 결과 정상

**Evidence**
- Role Owner(FAILBACK 후): `cjdb01p`  
  ![Role Owner - After Failback](./evidence/07_role_owner_after_failback.png)
- (cjdb01p) DML 테스트 성공  
  ![DML on cjdb01p](./evidence/08_dml_on_cjdb01p.png)

---

# 5. 결론
본 테스트를 통해 MSCS 클러스터 환경에서
- DTC 및 SQL Server(MSSQLSERVER) Role의 **Failover/Failback 전환**이 정상 동작하고,
- 공유 디스크 자원이 전환된 노드에서 **Online 상태를 유지**하며,
- 전환 후에도 **DML(INSERT/SELECT/DELETE) 정상 수행**됨을 증빙하였다.

---

# 6. 운영 관점 보완(필수)
> 지금 상태도 “증빙”은 충분하지만, 채용 관점에서 강하게 만들려면 아래 2개가 꼭 필요합니다.

1) **RTO(전환 소요시간) 기록**
   - Failover 시작/완료, Failback 시작/완료 시각(분 단위) 타임라인 추가
2) **서비스 관점 스모크 테스트**
   - SSMS DML 외에, 실제 서비스 핵심 기능 1~2개(로그인/조회 등) 확인 로그 또는 캡처 추가

---
