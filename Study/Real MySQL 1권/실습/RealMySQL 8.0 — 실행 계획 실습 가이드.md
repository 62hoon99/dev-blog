## 환경 세팅

```sql
CREATE DATABASE IF NOT EXISTS study CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE study;

CREATE TABLE employees (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    emp_no      VARCHAR(20)  NOT NULL,
    name        VARCHAR(50)  NOT NULL,
    department  VARCHAR(30)  NOT NULL,
    position    VARCHAR(30)  NOT NULL,
    status      VARCHAR(10)  NOT NULL,  -- ACTIVE / LEAVE / RETIRED
    salary      INT          NOT NULL,
    join_date   DATE         NOT NULL,
    birth_date  DATE         NOT NULL,
    gender      CHAR(1)      NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```python
# generate_data.py — 100만 건 생성 (약 3~5분 소요)
import mysql.connector, random
from datetime import date, timedelta

conn = mysql.connector.connect(
    host="127.0.0.1", port=3306,
    user="root", password="study1234", database="study"
)
cursor = conn.cursor()

DEPARTMENTS = ["개발팀","인프라팀","데이터팀","기획팀","마케팅팀",
               "영업팀","인사팀","재무팀","법무팀","고객지원팀"]
POSITIONS   = ["사원","주임","대리","과장","차장","팀장","이사"]
STATUSES    = ["ACTIVE"]*75 + ["LEAVE"]*15 + ["RETIRED"]*10
SALARY_BASE = {"사원":3000,"주임":3500,"대리":4200,"과장":5500,
               "차장":6500,"팀장":8000,"이사":10000}

def rand_date(s, e): return s + timedelta(days=random.randint(0,(e-s).days))
def rand_name():
    return random.choice(["김","이","박","최","정","강","조","윤","장","임"]) \
         + random.choice(["민준","서연","지우","서준","하은","지민","수빈","도윤","지현","예린"])

SQL = """INSERT INTO employees
(emp_no,name,department,position,status,salary,join_date,birth_date,gender)
VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s)"""

TOTAL, BATCH = 1_000_000, 1_000
for b in range(0, TOTAL, BATCH):
    pos = random.choice(POSITIONS)
    rows = [(f"EMP-{b+i+1:07d}", rand_name(), random.choice(DEPARTMENTS), pos,
             random.choice(STATUSES), SALARY_BASE[pos]+random.randint(-300,500),
             rand_date(date(2000,1,1),date(2024,12,31)),
             rand_date(date(1965,1,1),date(2000,12,31)),
             random.choice(["M","F"])) for i in range(BATCH)]
    cursor.executemany(SQL, rows)
    conn.commit()
    if b % 100_000 == 0: print(f"{b+BATCH:,} / {TOTAL:,}")

cursor.close(); conn.close(); print("완료")
```

> **주의**: 실습 전 인덱스는 PK 외에 **아무것도 만들지 않는다**.

---

## 실습 1 — 복합 인덱스 설계와 filesort 제거

**시나리오**: "현재 재직 중인 개발팀 직원을 연봉 높은 순으로 조회"

```sql
SELECT id, emp_no, name, position, salary
FROM employees
WHERE department = '개발팀'
  AND status = 'ACTIVE'
ORDER BY salary DESC
LIMIT 50;
```

**실습 목표**

- 인덱스 없을 때 실행 계획과 실행 시간을 먼저 기록한다
- 인덱스를 직접 설계하고 적용해서 실행 계획이 어떻게 바뀌는지 확인한다
- `Using filesort` 를 제거하는 인덱스를 찾아본다

**힌트**

- 컬럼 순서를 다르게 바꿔가며 실행 계획을 비교해본다
- `ORDER BY salary DESC` 를 인덱스가 처리하려면 인덱스 선언 시 어떻게 해야 할까?
- `EXPLAIN` 의 `type`, `key`, `rows`, `Extra` 를 기록해두고 비교한다

---

## 실습 2 — 커버링 인덱스

**시나리오**: 실습 1에서 만든 인덱스를 그대로 두고, SELECT 절을 바꿔가며 실행 계획을 비교한다

**실습 목표**

- `Extra: Using index` 가 언제 나타나는지 확인한다
- 커버링 인덱스가 적용되는 조건을 직접 찾아낸다

**힌트**

- SELECT 절에 인덱스에 없는 컬럼이 포함되면 어떻게 달라지는가?
- `Using index` 와 `Using where; Using index` 는 어떻게 다른가?

---

## 실습 3 — 인덱스를 무력화하는 쿼리 패턴

**시나리오**: 아래 두 쌍의 쿼리를 각각 실행하고 실행 계획을 비교한다

```sql
-- (A) 날짜 조건
ALTER TABLE employees ADD INDEX idx_join_date (join_date);

-- 비교 대상 ①
SELECT * FROM employees WHERE YEAR(join_date) = 2023;
-- 비교 대상 ②
SELECT * FROM employees WHERE join_date >= '2023-01-01' AND join_date < '2024-01-01';
```

```sql
-- (B) 타입 조건
ALTER TABLE employees ADD INDEX idx_emp_no (emp_no);

-- 비교 대상 ①
SELECT * FROM employees WHERE emp_no = 1234567;
-- 비교 대상 ②
SELECT * FROM employees WHERE emp_no = 'EMP-1234567';
```

**실습 목표**

- 같은 조건인데 `type` 이 왜 달라지는지 설명할 수 있다
- 실무에서 어떤 경우에 이 패턴이 나타나는지 생각해본다

**힌트**

- ①번 쿼리에서 인덱스 컬럼에 무언가가 적용되고 있다
- MySQL이 두 타입을 비교하려면 내부적으로 무엇을 하는가?

---

## 실습 4 — DEPENDENT SUBQUERY 탐지와 개선

**시나리오**: "자기 부서 평균 연봉보다 높은 직원 조회"

```sql
ALTER TABLE employees ADD INDEX idx_department (department);

-- 쿼리 A
SELECT id, name, department, salary
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department = e1.department
);

-- 쿼리 B
SELECT e.id, e.name, e.department, e.salary
FROM employees e
INNER JOIN (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_avg ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_salary;
```

**실습 목표**

- 두 쿼리의 `select_type` 차이를 확인한다
- 실행 시간을 비교하고 왜 차이가 나는지 설명할 수 있다

**힌트**

- `DEPENDENT SUBQUERY` 는 몇 번 실행되는가?
- `DERIVED` 는 무엇을 의미하는가?

---

## 실습 5 — 실행 계획의 rows와 filtered 해석

**시나리오**: 복합 조건 쿼리에서 `rows * (filtered / 100)` 이 실제로 무엇을 의미하는지 확인한다

```sql
ALTER TABLE employees ADD INDEX idx_dept_status (department, status);

EXPLAIN
SELECT * FROM employees
WHERE department = '개발팀'
  AND status = 'ACTIVE'
  AND salary > 5000;
```

**실습 목표**

- `rows` 와 `filtered` 수치를 읽고, 실제로 몇 건이 최종 반환될지 계산해본다
- `salary` 조건을 인덱스가 처리하지 못하는 이유를 설명할 수 있다
- 인덱스를 바꿔서 `filtered` 를 높이려면 어떻게 해야 하는지 직접 시도해본다

**힌트**

- `(department, status)` 인덱스 기준으로 `salary > 5000` 은 어느 단계에서 처리되는가?
- 인덱스를 `(department, status, salary)` 로 바꾸면 `filtered` 가 어떻게 달라지는가?

---

## Extra 실습

### Extra 1 — 인덱스 개수와 INSERT 성능

인덱스가 없는 테이블, 인덱스 3개짜리 테이블, 인덱스 6개짜리 테이블을 각각 만들고 1만 건씩 INSERT 해서 시간을 비교한다. 왜 차이가 나는지 InnoDB Change Buffer 관점에서 설명해본다.

### Extra 2 — 옵티마이저가 인덱스를 안 쓰는 경우

인덱스가 있음에도 옵티마이저가 풀 테이블 스캔을 선택하는 케이스를 만들어본다. `USE INDEX` / `FORCE INDEX` 힌트로 강제했을 때 오히려 느려지는지 확인하고, 왜 옵티마이저가 그 선택을 했는지 생각해본다.

> **힌트**: 카디널리티가 매우 낮은 컬럼(`gender`)에 단독 인덱스를 만들고 조회해보자.