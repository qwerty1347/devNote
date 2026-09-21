# 클러스터 인덱스 vs 비클러스터 인덱스

> 한 줄 요약: **클러스터 인덱스는 "행 데이터 그 자체를 정렬해 담고 있는 트리"**, **비클러스터 인덱스는 "값 → 행 위치"만 담은 별도 트리**다.

---

# 1. 개념

우리가 평소 "인덱스 걸었다"고 말하는 것 — `CREATE INDEX idx_email ON users(email)` 처럼 **특정 컬럼을 지정해서 만드는 것** — 이 바로 비클러스터 인덱스다. MySQL/InnoDB에서는 **PK를 제외한 모든 인덱스**가 여기 해당한다: `UNIQUE KEY`, `KEY`, 복합 인덱스, SQLAlchemy의 `index=True` 전부.

다만 **"컬럼을 지정했냐"가 둘의 구분 기준은 아니다.** 클러스터 인덱스도 특정 컬럼(PK)을 지정한 것이다. 진짜 차이는 *지정한 다음 무엇이 만들어지느냐*다.

| | 지정하는 컬럼 | 만들어지는 것 | 개수 |
| --- | --- | --- | --- |
| 클러스터 | `PRIMARY KEY (id)` | `id` 순서로 **행 데이터 자체**를 정렬해 저장. 별도 구조물 없음 = 테이블이 곧 인덱스 | 1개 |
| 비클러스터 | `CREATE INDEX ... (email)` | `email` 정렬 순서 + PK값만 담은 **별도의 작은 트리**를 하나 더 생성 | 여러 개 |

```
users 테이블 (InnoDB)
├── 클러스터 인덱스 (id)     ← 행 전체가 여기 산다. 1개뿐
├── 비클러스터 idx_email     ← 'a@x.com' → id=30   (사본, 작음)
└── 비클러스터 idx_name      ← 'kim'     → id=10   (사본, 작음)
```

> **책 비유** — 클러스터 인덱스는 책을 어떤 순서로 꽂을지(본문 배열 그 자체), 비클러스터 인덱스는 뒤에 붙인 **목차**다.
> 목차에서 페이지 번호(PK)를 찾은 뒤 그 번호로 본문을 다시 펼쳐야 내용을 볼 수 있고 — 이게 아래 2번의 "2번 탐색"이다.
> 책을 가나다순과 출간일순으로 *동시에* 꽂을 수는 없으니 클러스터 인덱스는 1개뿐이고, 목차는 가나다순·저자순·주제별로 몇 개든 덧붙일 수 있다.

## 클러스터 인덱스 (Clustered Index)

- 인덱스의 **리프 노드가 곧 행(row) 전체**다. 인덱스 = 테이블 데이터.
- 그래서 테이블당 **딱 1개**만 존재할 수 있다 (물리적 정렬 순서는 하나뿐이니까).
- 행은 클러스터 키 순서로 디스크에 저장된다 → **범위 검색이 매우 빠르다**.

```
클러스터 인덱스 (키 = id)
        [ 50 ]
       /      \
  [10,30]    [70,90]
   /  |  \     ...
 리프: id=10 → (id=10, name='kim',  email=..., created_at=...)   ← 행 전체
       id=30 → (id=30, name='lee',  email=..., created_at=...)
```

## 비클러스터 인덱스 (Non-clustered / Secondary Index)

- 리프 노드에 **인덱스 컬럼 값 + 행을 찾아갈 포인터**만 있다.
- 테이블당 **여러 개** 만들 수 있다.
- 실제 행이 필요하면 포인터를 따라 한 번 더 찾아가야 한다(뒤의 2번).

```sql
-- 이 세 가지가 전부 비클러스터 인덱스다
CREATE INDEX idx_name ON users(name);              -- 일반
CREATE UNIQUE INDEX uq_email ON users(email);      -- 유니크
CREATE INDEX idx_name_email ON users(name, email); -- 복합
```

```
비클러스터 인덱스 (키 = email)
 리프: 'a@x.com' → [PK 30]
       'b@x.com' → [PK 10]
                     ↓  이 PK로 클러스터 인덱스를 다시 탐색
```

---

# 2. 핵심 차이: 세컨더리 인덱스 조회는 2번 탐색한다

InnoDB에서 비클러스터 인덱스 리프에 들어 있는 포인터는 **PK 값**이다.

```sql
SELECT * FROM users WHERE email = 'a@x.com';
```

1. `idx_email` B-Tree 탐색 → `PK = 30` 획득
2. `PK = 30`으로 **클러스터 인덱스를 다시 탐색** → 행 전체 읽기

이 2단계를 **북마크 룩업 / 테이블 룩업**이라 부른다. 그래서:

- **PK가 크면(예: UUID CHAR(36)) 모든 세컨더리 인덱스가 같이 뚱뚱해진다.** PK를 값으로 들고 있기 때문.
- 조회 컬럼이 인덱스 안에 전부 들어 있으면 2단계를 건너뛴다 → **커버링 인덱스**.

```sql
-- idx_email(email) 만 있으면 → 룩업 발생
SELECT * FROM users WHERE email = 'a@x.com';

-- idx_email_name(email, name) 이면 → 인덱스만 읽고 끝 (Extra: Using index)
SELECT name FROM users WHERE email = 'a@x.com';
```

---

# 3. DBMS별 동작 차이

| | MySQL / InnoDB | PostgreSQL | SQL Server |
| --- | --- | --- | --- |
| 클러스터 인덱스 | **항상 존재** | **없음** (모든 인덱스가 비클러스터) | 선택 (없으면 힙 테이블) |
| 클러스터 키 선정 | PK → 없으면 첫 UNIQUE NOT NULL → 없으면 숨은 6바이트 `GEN_CLUST_INDEX` | - | `CREATE CLUSTERED INDEX` 로 지정 |
| 세컨더리 인덱스 포인터 | **PK 값** | `ctid`(물리 위치) | 클러스터 키 값 / 없으면 RID |
| 비고 | PK 설계가 곧 물리 설계 | `CLUSTER` 명령은 1회성 재정렬, 유지 안 됨 | 테이블당 1개 |

> PostgreSQL에는 클러스터 인덱스 개념이 없다. `CLUSTER table USING idx`는 그 시점에 한 번 물리 정렬할 뿐, 이후 INSERT/UPDATE로 곧 흐트러진다.

---

# 4. PK 설계가 성능에 미치는 영향 (InnoDB)

## AUTO_INCREMENT vs UUID

```sql
-- 좋음: 항상 B-Tree 오른쪽 끝에 append → 페이지 분할 거의 없음
id BIGINT AUTO_INCREMENT PRIMARY KEY

-- 나쁨: 삽입 위치가 무작위 → 페이지 분할·단편화, 버퍼풀 적중률 하락
id CHAR(36) PRIMARY KEY  -- UUIDv4
```

UUID를 꼭 써야 한다면:

- **UUIDv7 / ULID** 처럼 시간순 정렬되는 값을 쓴다.
- `BINARY(16)` 으로 저장한다 (`UUID_TO_BIN(uuid, 1)` 은 타임스탬프 앞부분을 swap해 정렬성을 준다).
- 또는 내부 PK는 `BIGINT AUTO_INCREMENT`, 외부 노출용 UUID는 `UNIQUE` 세컨더리 인덱스로 둔다. ← 실무에서 가장 무난

## PK를 짧게 유지해야 하는 이유

세컨더리 인덱스 N개 = PK 값이 N번 복제 저장. PK가 8바이트에서 36바이트로 늘면 인덱스 전체 용량이 몇 배로 뛴다.

---

# 5. 실행계획으로 확인하기

```sql
EXPLAIN SELECT * FROM users WHERE email = 'a@x.com';
```

| 표시 | 의미 |
| --- | --- |
| `type: const` / `eq_ref` | PK·UNIQUE 단건 → 클러스터 인덱스 직접 접근 |
| `type: ref` + `key: idx_email` | 세컨더리 인덱스 탐색 후 룩업 |
| `Extra: Using index` | **커버링 인덱스** — 룩업 없음 |
| `Extra: Using index condition` | ICP(Index Condition Pushdown) — 룩업 전에 인덱스 단계에서 걸러냄 |
| `type: ALL` | 풀 스캔 (클러스터 인덱스 전체 순회) |

---

# 6. SQLAlchemy / 정의 예시

```python
from sqlalchemy import BigInteger, String, Index
from sqlalchemy.orm import Mapped, mapped_column, DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    # InnoDB: 이 PK가 클러스터 인덱스가 된다
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)

    # 비클러스터(세컨더리) 인덱스
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(50))

    __table_args__ = (
        # 복합 인덱스: 순서가 곧 성능 (왼쪽 접두사 규칙)
        Index("idx_name_email", "name", "email"),
    )
```

```sql
-- 위와 동등한 DDL
CREATE TABLE users (
  id     BIGINT      NOT NULL AUTO_INCREMENT,
  email  VARCHAR(255) NOT NULL,
  name   VARCHAR(50)  NOT NULL,
  PRIMARY KEY (id),                 -- 클러스터 인덱스
  UNIQUE KEY uq_email (email),      -- 비클러스터
  KEY idx_name_email (name, email)  -- 비클러스터(복합)
) ENGINE=InnoDB;
```

---

# 7. 언제 무엇을 쓰나

| 상황 | 선택 |
| --- | --- |
| PK 단건 조회가 대부분 | 클러스터 인덱스(PK)만으로 충분 |
| `WHERE created_at BETWEEN ...` 범위 스캔이 잦음 | 해당 컬럼을 클러스터 키로 두면 최고 (SQL Server). InnoDB라면 커버링 복합 인덱스로 대체 |
| 조회 조건 컬럼이 다양함 | 비클러스터 인덱스 여러 개 |
| 특정 쿼리가 핫패스 | 그 쿼리 전용 **커버링 인덱스** (`WHERE`컬럼 + `SELECT`컬럼) |
| 쓰기가 매우 많음 | 인덱스 개수를 줄인다 — 인덱스마다 INSERT/UPDATE 비용이 붙는다 |

---

# 8. 흔한 함정

- **인덱스를 많이 만들수록 좋다** → 아니다. 쓰기 비용과 저장공간이 인덱스 수만큼 늘고, 옵티마이저 선택도 흔들린다.
- **복합 인덱스 `(a, b)`는 `b` 단독 조건에 안 쓰인다** (왼쪽 접두사 규칙). → [[복합 인덱스 컬럼 순서 정하기]]
- **컬럼에 함수를 씌우면 인덱스를 못 탄다**: `WHERE DATE(created_at) = '2026-09-21'` → `WHERE created_at >= '2026-09-21' AND created_at < '2026-09-22'`.
- **PK를 UPDATE하면** 클러스터 인덱스에서 행이 물리적으로 이동하고, 모든 세컨더리 인덱스가 갱신된다. PK는 불변으로 둔다.
- **카디널리티가 낮은 컬럼**(`is_active`, `gender`)의 **단독** 인덱스는 대체로 무용. 복합 인덱스의 구성 컬럼으로 쓴다 — 항상 `=` 조건이면 선두여도 좋다. → [[복합 인덱스 컬럼 순서 정하기]]

---

## 관련 노트

- [[복합 인덱스 컬럼 순서 정하기]]
- [[인덱스가 많으면 쓰기가 느려지는 이유]]
- [[SQLAlchemy 쿼리 가이드 (Laravel 비교)]]
- [[관계(조인) CRUD 예제]]
- [[Elasticsearch 쿼리 가이드 (auth_apikey)]]
