# 1️⃣ B+Tree 인덱스

## A. 기본 구조

- **정의:** B+Tree는 균형 잡힌 트리 구조로, 검색/삽입/삭제 시 트리 높이가 유지되어 성능이 보장된다. 모든 실제 키(데이터 포인터)는 **리프 노드**에 있고, 내부 노드(비리프)는 검색용 키만 갖는다.
- **특징 요약**
    - 균형 트리(모든 리프가 같은 깊이)
    - 내부 노드는 키와 포인터(다음 노드 주소)
    - 리프 노드는 실제 레코드(또는 레코드의 포인터)를 가리키며, 순차 연결(Linked list)로 이어짐 → **Range Scan**에 유리

### B-Tree vs B+Tree 차이

- **B-Tree:** 키와 값이 내부/리프 모두에 존재. 탐색이 내부에서 끝날 수 있음.
- **B+Tree:** 모든 실제 값은 리프에만 존재. 내부 노드는 오직 탐색용 키만 보유. → 리프의 순차 연결과 높이 축소로 범위 연산 유리.

### 리프 노드 구성

- 리프 엔트리: (key, record pointer) 또는 (key, full row) — 스토리지 엔진에 따라 다름.
- 리프는 좌/우 링크로 연결되어 있어 연속 레코드 읽기가 빠름.

### Range Scan이 유리한 이유

- 리프가 순차 연결되므로, 범위 시작 키를 찾은 뒤 인접 리프를 순차적으로 읽으면 됨 → **sequential I/O**로 효율적.
- 랜덤 I/O(여러 포인터로 흩어진 읽기)보다 디스크/스토리지에서 훨씬 빠름.

### Fan-out & 노드 크기 (예: 16KB 페이지)

- **Fan-out** = 한 내부 노드가 가리킬 수 있는 자식 수. 페이지 크기와 각 엔트리 크기에 따라 결정됨.
- 예: 페이지 16KB, 키+포인터가 평균 16B라면 fan-out ≈ 16KB / 16B = 1024. 즉 트리 높이가 매우 낮아짐(수백만 레코드도 높이 3~4).
- 노드 크기 최적화는 탐색 깊이와 메모리/디스크 I/O에 직접 영향.

### MySQL(InnoDB)에서 B+Tree가 **컴팩트한 이유** (Record Format)

- InnoDB는 페이지 내에서 레코드를 **압축/정렬**하고, 가능한 PK만 저장(보조인덱스 제외)하며 레코드 헤더/필드 오프셋 등으로 공간을 절약.
- 레코드 포맷(Compact / Redundant)이 있고, Compact 포맷은 null/length 저장 최적화로 페이지당 더 많은 레코드를 넣을 수 있음.

---

## B. 내부 저장 구조 (InnoDB 중심)

### InnoDB의 페이지 구조

- **표준 페이지 크기:** 보통 16KB (innodb_page_size)
- 페이지 내부 레이아웃
    
    ```mathematica
    ┌──────────────────────┐
    │ File Page Header     │ 38 bytes
    ├──────────────────────┤
    │ Page Header          │ 56 bytes
    ├──────────────────────┤
    │ Infimum / Supremum   │ 26 bytes
    │  (Sentinel Records)  │
    ├──────────────────────┤
    │ User Records         │  (가변)
    │  (정렬된 B+Tree 레코드)│
    ├──────────────────────┤
    │ Free Space           │  (가변)
    ├──────────────────────┤
    │ Page Directory       │  2 bytes × N slots
    ├──────────────────────┤
    │ File Trailer         │  8 bytes
    └──────────────────────┘
    ```
    

### File Page Header / File Trailer

- File Page Header
    - 페이지 타입(root/leaf/internal), prev/next pointers, LSN 등 메타 정보.
    - 페이지 간 연결/일관성/복구 위한 정보
- File Page Trailer: 체크섬(페이지 손상 여부 확인), 복구/무결성 정보.

### Infimum / Supremum Record

- 페이지 정렬 시작/끝 지점. 실제 데이터가 아님.
- 정렬과 경계 체크에 사용

### Page Directory와 Slot

- Page Directory(=Slot Array)
    - 레코드들을 그룹으로 나누고 각 그룹의 첫 레코드 offset 저장
    - 탐색: O(log N)→O(k)
        - Slot Array에서 binary search → O(log N)
        - Slot이 가리키는 레코드 내부에서 linear search → O(k)
- Record heap: 실제 레코드는 끝쪽에서부터 쌓이고 slot은 앞쪽에서 관리되는 형태(가비지 생성 시 compaction 가능).

### 리프노드: 실제 데이터 저장 vs 보조 인덱스는 PK만 저장

- **Clustered Index**
    - 리프 노드에 **실제 전체 행(row)** 저장.
    - 반드시 하나 존재.
- **Secondary Index**
    - 리프노드에 (Secondary key, PK) 만 저장.
    - Double Read: 보조 인덱스에서 실제 레코드 접근 시 PK로 클러스터드 인덱스에서 찾음.
    - 필요할 때마다 여러 개 추가 가능.

### Non-leaf 노드의 역할

- 내부(비리프) 노드: **오직 검색용**. 키 값과 해당 자식 페이지 포인터만 유지. 실제 데이터는 없음.

---

## C. 탐색 알고리즘

### 인덱스 탐색 과정 시뮬레이션 (Root → Branch → Leaf)

1. Root 노드에서 시작.
2. 내부 노드의 키들을 바이너리/linear search로 비교해 적절한 자식 포인터 선택.
3. 선택한 자식(Branch)로 이동, 반복.
4. Leaf 도달 → 리프 엔트리에서 키를 찾고, 연관된 레코드 포인터(또는 실제 레코드)를 획득.

(실제 구현은 binary search, SIMD, prefix-compression 등 최적화 포함 가능)

### Random I/O vs Sequential I/O

- **Random I/O:** 여러 개의 페이지를 비연속적으로 읽어야 하는 경우 — 비용 큼(특히 디스크).
- **Sequential I/O:** 인접 블록을 순차적으로 읽음 — 더 효율적.
- Range Scan은 리프를 순차적으로 읽어 **sequential I/O**로 동작 → 효율적.

### Range Scan / Index Condition Pushdown(ICP) 동작

- **Range Scan:** 인덱스의 시작점에서 끝점까지 리프를 따라 순차 읽기.
- **Index Condition Pushdown (ICP):** MySQL 엔진이 인덱스 레벨에서 WHERE의 추가 조건을 적용하여 디스크/IO를 줄임.
    - 예: `WHERE idx_col BETWEEN a AND b AND other_col = c`에서 `other_col`도 인덱스에 포함돼 있거나 표현식으로 체크 가능하면 리프 레코드를 fetch하기 전에 필터링을 적용 → 불필요한 PK lookups 감소.

---

# 2️⃣ Clustered Index — PK 기반 저장 구조 깊게 이해하기

## A. InnoDB가 Clustered Index를 사용하는 이유

- **레코드 정렬 저장:** 클러스터드 인덱스는 PK 순으로 실제 데이터를 저장 → PK 기반의 범위 검색과 정렬이 빠름(추가 정렬 불필요).
- **성능 이점:** PK로 접근하면 인덱스 → 바로 레코드(같은 페이지)에 있어 1단계로 해결 가능한 경우가 많음.
- **테이블 자체가 B+Tree:** 테이블의 리프 레벨이 실제 데이터(행)를 포함.

### PK 기반으로 실제 데이터 페이지에 저장되는 구조

- 리프 노드 엔트리는 전체 행(컬럼 값들, 혹은 포인터로 표현된 레코드)으로 구성.
- 행 업데이트/삽입 시 페이지 오프셋/slot 재배치가 발생.

### PK가 길면 왜 성능이 저하되는지

- PK 저장되는 곳: Clustered Index의 브랜치 노드, Secondary Index의 리프 노드
- PK가 길어지면 branch page에 들어갈 수 있는 key 수가 줄어들고 모든 Secondary Index가 커짐.
- Secondary index는 PK를 포인터로 저장하기 때문에 PK가 길면 보조 인덱스 엔트리 크기도 커져 더 많은 공간 사용 및 I/O 증가.
- **긴 PK** → 내부/리프 노드의 키 크기 증가 → fan-out 감소 → 트리 높이 증가 → 더 많은 디스크/메모리 페이지 접근.
- fan-out: 한 브랜치 노드가 몇 개의 자식을 가질 수 있는지

## B. Secondary Index(보조 인덱스)의 내부 구조

- 보조 인덱스의 **leaf**는 (secondary_key, PK) 형태로 저장.
- PK는 보조 인덱스에서 '레코드를 찾기 위한 포인터' 역할.
- **Secondary Index Lookup (Double Read)**:
    1. 보조 인덱스에서 secondary_key로 검색 → PK 획득.
    2. PK로 클러스터드 인덱스(테이블 B+Tree)에서 실제 행을 재탐색 → 실제 데이터 획득.
- 이중 읽기 비용을 줄이는 방법: **covering index**(쿼리에 필요한 모든 컬럼을 인덱스에 포함).

---

# 3️⃣ EXPLAIN문 — 식별 정보부터 실행 흐름까지

EXPLAIN문: MySQL이 쿼리를 어떻게 실행할지 보여주는 도구.

## A. EXPLAIN 주요 컬럼 해석 (기본 `EXPLAIN SELECT ...`)

```sql
EXPLAIN SELECT * FROM user WHERE id = 10;
```

- **id:** 서브쿼리/조인 단계 식별자
- **select_type:** 어떤 종류의 SELECT인지
- **table:** 해당 행이 검사하는 테이블
    - **SIMPLE:** 서브쿼리 없는 단순 SELECT
    - **PRIMARY:** 가장 바깥 SELECT
    - **SUBQUERY:** 서브쿼리
    - **DERIVED:** FROM (sub-select) 로 만들어진 임시 테이블
    - **UNION:** UNION의 두 번째 SELECT
- **type:** 테이블을 스캔하는 방식
    
    
    | type | 의미 |
    | --- | --- |
    | **const** | PK 또는 UNIQUE=값 하나 (최고빠름) |
    | **ref** | 일반 인덱스 탐색 |
    | **range** | 인덱스 범위 스캔 (>, <, BETWEEN) |
    | **index** | 인덱스 전체 스캔 |
    | **ALL** | 테이블 전체 스캔 (최악) |
- **possible_keys:** 사용 가능한 인덱스 후보
- **key:** 실제로 선택된 인덱스. NULL→인덱스 전혀 사용 안 함. 성능이 기대 이하라면 체크해보기.
- **key_len:** 사용된 인덱스의 길이(바이트). 인덱스를 부분/전 사용하는지 체크.
    - `key_len`이 작으면 옵티마이저가 인덱스의 일부만 사용함. 예: prefix 인덱스, 타입 크기, NULL 플래그 등 고려.
- **ref:** 어떤 값이 인덱스와 비교되는지 (const: WHERE 절의 고정값, user.id: 조인에서 다른 테이블 컬럼)
- **rows:** 옵티마이저가 예상하는 읽을 행 수(추정치)
- **filtered:** WHERE로 필터링되는 비율(%)

## B. EXPLAIN FORMAT=JSON 해석 (핵심 필드)

- `cost_info`: 옵티마이저가 계산한 비용(상대적 비용 점수)
- `chosen_range`: 어떤 range가 선택되었는지
- `attached_condition`: 인덱스로 처리하지 못한 조건 (레코드를 읽은 후 서버 레벨에서 필터링하는 조건)
- `index`: 스캔 자세한 정보(시작 키, 끝 키, direction)
- `scanning`/`reading` 상세: 고급 최적화 사용 여부
    
    
    | 최적화 | 의미 |
    | --- | --- |
    | **MRR (Multi-Range Read)** | 랜덤 I/O → 순차 I/O로 묶어서 읽기 |
    | **BKA (Batched Key Access)** | 조인 시 PK lookup을 묶어서 읽기 |
    | **ICP (Index Condition Pushdown)** | 인덱스 레벨에서 WHERE 일부 평가 |
- JSON은 더 많은 내부 정보(프로파일링/추정값/추가적 조건)를 제공 → 복잡한 쿼리의 디버깅에 유용.

---

# 4️⃣ 옵티마이저 전략 — MySQL이 어떻게 실행 계획을 고르는가

## A. 비용 기반 옵티마이저 (CBO)

- **목표:** 전체 비용(추정 I/O, CPU)을 최소화하는 실행 계획 선택.
- **핵심 요소**
    - **Cardinality estimation:** 특정 조건에서 얼마나 많은 행을 읽을지 예측→낮을수록 좋은 조건
    - **Index statistics:** 인덱스별 분포 정보 (유니크 값 개수, 가장 작은/큰 값, NULL 비율, 페이지 당 레코드 수, 분포도 정보). 통계 정보가 오래되면 엉뚱한 인덱스 선택하게 됨.→어떻게선택?
    - **Histograms:** 컬럼값이 편향된 정도 파악. 데이터 분포가 쏠려 있는 경우 최적화 가능.

## B. Access Method 전략

- **index range scan:** 범위 조건에서 사용(효율적)
- **index ref / eq_ref:** 조인에서 상수/유일키에 따른 접근
- **full table scan 판단:** 테이블이 작거나(테이블 전체 스캔 비용 < 인덱스 사용 비용), 인덱스 선택성이 낮을 때
- **index skip scan (MySQL 8+):** 첫 컬럼이 없는 경우에도 일부 컬럼 조합으로 인덱스 활용(Oracle에 있던 개념 유사)
- **index merge:** 여러 인덱스의 결과를 합쳐서 사용 (intersection/union) — 비용 대비 효과가 있을 때 사용.

## C. Join Optimizer

- **Nested Loop Join (NLJ):** MySQL의 기본 조인 방식. 바깥쪽 루프의 각 행에 대해 내부 루프에서 인덱스로 검색.
- **join order 결정:** 비용 기반으로 결정. 작은(또는 driving) table을 바깥으로 두는 것이 일반적.
- **driven table 선택 기준:** 테이블 크기, 인덱스 유무, 필터 셀렉티비티 등.
- **Multi-Range Read (MRR) 최적화:** 인덱스 lookup 시 다수의 랜덤 I/O를 모아 정렬/재배치하여 순차 I/O로 전환 → 성능 향상.
- **Batched Key Access (BKA):** 조인에서 바깥쪽 여러 키를 모아 내부 테이블을 일괄로 조회해 I/O 효율화(특히 MRR과 연동).

## D. Query Rewrite 단계

- 옵티마이저는 실행 전 **재작성(Rewrite)** 단계에서 쿼리를 변환하여 더 좋은 계획을 유도.
    - **subquery → semi join 변환:** 원래 탐색(서브쿼리 먼저 실행→결과를 비교)보다 빨라짐.
    - **constant propagation:** 상수 값 전파로 표현식 간소화.
    - **predicate pushdown:** 뷰/서브쿼리 내부 조건을 바깥 레벨로 끌어올리거나 반대로 밀어넣어 필터링 단계를 앞당김→읽는 페이지 줄어듦
    - **view merge / derived table merge:** 부모 쿼리에 병합시켜 인덱스 활용 (뷰나 파생테이블은 별도 임시 테이블로 만들어짐→인덱스 사용X→속도 느려짐.)

---
