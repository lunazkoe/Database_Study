# S17. 트랜잭션이 동시에 실행될 때 발생 가능한 이상 현상들 / Isolation Level

## Dirty Read

### 상황
- x = 10 / y = 20
- tx1: x에 y를 더함
- tx2: y를 70으로 바꿈

### 발생할 수 있는 상황
- tx1
    - r1(x) = 10

- tx2
    - w2(y) = 70

- tx1
    - r1(y) = 70: tx2가 70으로 바꾸었기 때문
    - x+y
    - w1(x) = 80
    - c1

- tx2
    - abort: rollback(y) = 20

> 발생 현상: X는 80은 rollback한 tx2가 쓴 y=70의 값을 읽고 사용했기 때문에 나온 값. 70은 유효한 값이 아님(롤백되었기 때문) / x=80도 이상한 값임

> 이러한 현상을 Dirty Read라고 함

### 개념
- commit되지 않은 변화를 읽음

---

## Non-Repeatable Read

### 상황
- x = 10
- tx1: x를 두 번 읽음
- tx2: x에 40을 더함

### 발생할 수 있는 상황
- tx1
    - r1(x) = 10

- tx2
    - r2(x) = 10
    - +40
    - w2(x) = 50
    - c2

- tx1
    - r1(x) = 50
    - c1

> 발생 현상: 같은 데이터를 한 트랜잭션에서 두 번 읽었음에도 불구하고 10, 50 각각 다른 값으로 읽게 됨. 이것은 Isolation 관점에서 일어나면 안됨. 각 트랜잭션은 독립적으로 실행되는 것처럼 동작해야함. 

> 이러한 현상을 Non-Repeatable Read

### 개념 (or Fuzzy Read)
- 같은 데이터의 값이 달라짐

---

## Phantom Read

### 상황
- t1(..., v = 10)
- t2(..., v = 50)
- tx1: v가 10인 데이터를 두 번 읽음
- tx2: t2의 v를 10으로 바꿈

### 발생할 수 있는 상황
- tx1
    - r1(v=10) = t1: v가 10인 값이 t1 밖에 없음

- tx2
    - w2(t2, v=10)
    - c2

- tx1
    - r1(v=10) = t1, t2: v가 10인 값이 t2가 생김

> 발생 현상: 같은 조건으로 데이터를 두 번 읽었는데, 다른 결과가 나오게 됨. Isolation 위배

> 이러한 현상을 Phantom Read

### 개념
- 없던 데이터가 생김

---

## Isolation Level

### Dirty Read / Non-Repeatable Read / Phantom Read
- 이 현상들은 발생하지 않는게 좋음(이상 현상)
- 문제
    - 이러한 현상들이 모두 발생하지 않게 만들 수 있지만
    - 그러면 제약사항이 많아져서 동시 처리 가능한 트랜잭션 수가 줄어들어
    - 결국 DB의 전체 처리향이 하락하게 됨

- 해결
    - 일부 이상한 현상은 허용하는 몇 가지 **Level**을 만들어서 
    - 사용자(개발자)가 필요에 따라서 적절하게 선택할 수 있도록 하자!!
    - **Isolation Level**

### Isolation Level에 따른 이상 현상 허용 범위
- Dirty Read | Non-Repeatable Read | Phantom Read
- 다음 레벨은 아래로 내려갈수록 엄격함

- Read Uncommited
    - 커밋되지 않은 데이터를 읽을 수 있음
    - O | O | O
    - 성능은 가장 좋음

- Read Commited
    - 커밋되지 않은 데이터를 읽을 수 없음
    - X | O | O

- Repeatable Read
    - Read Commited를 포함하고 Non-Repeatable Read를 허용하지 않음
    - X | X | O

- Serializable
    - X | X | X
    - 위 세 가지 현상뿐만 아니라 아예 이상한 현상 자체가 발생하지 않는 Level을 의미함!!

### 결론
- 세 가지 이상한 현상을 정의하고, 어떤 현상을 허용하는지에 따라서 각각의 Isolation Level이 구분됨
- 애플리케이션 설계자는 Isolation Level을 통해 전체 처리량과 데이터 일관성 사이에서 어느 정도 거래를 할 수 있음

---

## Isolation Level 비판 논문

### 비판
1. 세가지 이상 현상의 정의가 모호함
2. 이상 현상은 세 가지 외에도 더 있음
3. 상업적인 DBMS에서 사용하는 방법을 반영해서 Isolation Level을 구분하지 않았음

### 상황
- x = 0
- tx1: x를 10으로 바꿈
- tx2: x를 100으로 바꿈

### 발생할 수 있는 상황: 이상 현상은 세 가지 외에도 더 있음
- tx1
    - w1(x) = 10

- tx2
    - w2(x) = 100

- tx1
    - abort: rollback(x = 10)
    - 문제: tx2가 x를 100으로 변경을 했었는데, 그게 사라짐

- tx2
    - abort: rollback(x = 10)
    - 문제: w2(x) = 100 시점에 10이라는 값이였음
        - 근데 10이라는 값이 abort가 되었기 때문에 롤백을 하면 안됨

### Dirty Write
- commit 안된 데이터를 write 함
- rollback 시 정상적인 recovery는 매우 중요하기 때문에 모든 isolation level에서 dirty write를 허용하면 안됨

### 상황
- x = 50
- tx1: x에 50을 더함
- tx2: x에 150을 더함

### 발생할 수 있는 상황: 이상 현상은 세 가지 외에도 더 있음
- tx1
    - r1(x) = 50
    
- tx2
    - r2(x) = 50
    - +150
    - w2(x = 200)
    - c2

- tx1
    - +50
    - w1(x = 100)
    - c1

> 문제 발생: tx2가 작성한 내용이 사라져버림

### Lost Update
- 업데이트를 덮어씀

---

## Dirty Read

### 문제점
- Commit 되지 않은 변화를 읽음
- 논문에서는 롤백이 발생하지 않아도 Dirty Read가 발생할 수 있음

### 상황
- x = 50, y = 50
- tx1: x가 y에 40을 이체함
- tx2: x와 y를 읽음

### 발생할 수 있는 상황
- tx1
    - r1(x) = 50
    - -40
    - w1(x = 10)

- tx2
    - r2(x) = 10
    - r2(y) = 50
    - c2

- tx1
    - r1(y) = 50
    - +40
    - w1(y = 90)
    - c1

> 문제: x + y = 100으로 유지되어야함. 근데 tx2는 x = 10, y = 50으로 읽음. 이것 또한 Dirty Read

> 결론: abort가 발생하지 않아도 dirty read가 될 수 있음

---

## Read Skew

### 상황
- x = 50, y = 50
- tx1: x가 y에 40을 이체함
- tx2: x와 y를 읽음

### 발생할 수 있는 상황
- tx2
    - r2(x) = 50

- tx1
    - r1(x) = 50
    - -40
    - w1(x) = 10
    - r1(y) = 50
    - w1(y) = 90
    c1

- tx2
    - r2(y) = 90

> 문제: tx2의 경우 x + y = 140 != 100 데이터 불일치 발생. 이런걸 Read Skew라고 부름

### 개념
- inconsistent한 데이터 읽기

---

## Write Skew

### 상황
- x = 50, y = 50 (x + y >= 0)
- tx1: x에서 80을 인출함
- tx2: y에서 90을 인출함

### 발생할 수 있는 상황
- x + y >= 0인지를 체크해야해서 x를 읽는 것
- tx1
    - r1(x) = 50
    - r1(y) = 50: 인출 가능하다 판단

- tx2
    - r2(x) = 50
    - r2(y) = 50: 인출 가능하다 판단

- tx1
    - -80
    - w1(x) = -30

- tx2
    - -90
    - w2(y) = -40

- tx1
    - c1

- tx2
    - c2

> 결과: x = -30, y = -40: 데이터 불일치 발생

### 개념
- inconsistent한 데이터 쓰기

---

## Phantom Read 확장판(논문)

### 상황
- t1(..., v = 7) cnt = 0 / cnt는 v가 10보다 큰 개수를 나타냄
- tx1: v > 10 데이터와 cnt를 읽음
- tx2: v = 15인 t2를 추가하고 cnt를 1 증가

### 발생할 수 있는 상황
- tx1
    - r1(v > 10) = .

- tx2
    - w2(insert t2: ..., v = 15)
    - r2(cnt) = 0
    - +1
    - w2(cnt = 1)
    - c2

- tx1
    - r(cnt) = 1: 여기가 문제 v1을 못 찾았는데, cnt는 1이 나옴!!
    - c1

> 즉, 같은 조건의 데이터를 두 번 읽는게 아니더라도 없던 데이터가 생길 수 있음

---

## 상업적인 DBMS에서 사용되는 방법을 반영해서 Isolation Level을 구분하지 않았다

### Snapshot Isolation
- Concurrency Control을 어떻게 구현을 할거냐를 반영해서 Level을 정의함

### 예제
- x = 50 | y = 50
- tx1: x가 y에 40을 이체함
- tx2: y에 100을 입금함

### 상황
- tx1
    - r1(x) = 50
        - **이때 스냅샷을 찍음**
            - 트랜잭션이 시작하는 시점에 스냅샷을 찍음
            - 현재 x = 50, y = 50
    - -40
    - w1(x = 10)
        - **이때 데이터베이스에 바로 반영하는 것이 아니라 스냅샷에 반영함**
            - 따라서 x = 10(스냅샷)
            - 이때 외부에서는 아직 x = 50

- tx2
    - r2(y) = 50
        - 스냅샷을 찍음
            - 현재 y = 50
    - +100
    - w2(y = 150)
        - 스냅샷에 적용
            - 현재 y = 150
    - c2
        - 이때 스냅샷에 있는게 적용됨
        - 실제 데이터베이스에 y = 150으로 적용

- tx1
    - r1(y) = 150...을 읽지 않음
        - 스냅샷을 봐야함(y = 50)
    - r1(y) = 50
    - +40
    - w1(y = 90)
        - 스냅샨의 y = 90
    - c1을 하게 되면 tx2에서 같은 데이터에 대해서 쓰기 작업을 해서 이 작업이 날아갈 수 있음
    > 그래서!! 첫번째로 반영된 스냅샷만 인정을 해줌
    - 그래서!! tx1은 abort로 처리가 됨

> 이것을 스냅샷 isolation이라고 하고, 이건 MVCC(Multi Version Concurrency Control)의 한 종류임

### 정리하자면
- tx 시작전 commit된 데이터만 보임
- First-Committer win!!

---

## 정리 및 실무에서의 Isolation Level

### MySQL (InnoDB)
- 표준 SQL에서 정의한 Isolation Level 4가지와 동일하게 정의하고 있음

### Oracle
- 표준 SQL에서 정의한 Isolation Level을 정의함
- 단, READ UNCOMMITED를 제공하지 않음
- 또한 Repeatable Read / Serializable을 할 때 그냥 Serializable로 적용을 하라고 가이드
- 또한 Serializable은 Shapshot Isolation Level로 동작함

### SQL Server
- 5가지로 적용함 (Snapshot 포함)

### PostgreSQL
- 표준 SQL에서 정의한 Isolation Level 4가지와 동일하게 정의하고 있음
- 표준에서 발생할 수 있는 이상현상 3가지 외에도 Serialization Anomaly를 정의함
    - 이건 Serializable만 가능하지 않게함 (그외 Level은 허용)
- 또한 Repeatable Read가 스냅샷에 해당함

### 정리
- 주요 RDBMS는 SQL 표준에 기반해서 isolation level을 정의함
- RDBMS마다 제공하는 isolation level이 다름
- 같은 이름의 isolation level이라도 동작 방식이 다를 수 있음
- **사용하는 RDBMS의 isolation level을 잘 파악하는 것이 중요함**
- 물론 Default로 제공해주는 것이 있음
    - 퍼포먼스 튜닝
    - 혹은 문제가 발생할 수 있는 부분에 대해서 고민해보고 적용
- 추가적으로 발생할 수 있는 이상 현상에 대해서 여러 가지를 알고 있는 것이 좋음(3가지 말고 더 많이 알고 있는 것이 좋음)

---

## 추가사항

### 기본 이상 현상(Dirty/Non-Repeatable/Phantom Read)
- 논문에서는 Lock 기반의 동시성 제어를 가정하고 느슨하게 정의했음
- Lock을 사용하지 않는 MVCC 환경에서는 정의만으로 이상 현상을 명확히 판별하기 어려움

### 논문 기반 확장된 이상 현상
- Dirty Write
    - 격리 수준이 낮아서 발생한다기 보나는 근본적인 물리적 정합성을 깨뜨림
    - 이것은 절대 허용해서는 안되는 현상 (Read Uncommitted도 이건 허용하지 않음)

- Read/Write Skew
    - Write Skew는 서로 다른 데이터를 수정하지만, 그 두 데이터 사이에 논리적인 제약 조건(x + y >= 0)이 있을 때 발생함

### 상업적인 DBMS의 Isolation Level 구현
- MySQL
    - 논리적인 4단계는 동일한
    - InnoDB의 기본 격리 수준은 Repeatable Read
        - 이 수준에서 Phantom Read가 거의 발생하지 않음
        - 표준에 따르면 발생해야함!!
    - 근데 InnoDB는 Next-Key Lock과 MVCC를 혼합하여 사용하여, Repeatable Read에서도 Phantom Read를 막아냄
    - 단, 순수 select 후 select for update를 섞어 쓰는 특수한 상황에서는 발생할 수 있음
