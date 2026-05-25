# S16. Concurrency Control 기초 이론: recoverability. 트랜잭션이 동시에 실행될 때 rollback이 발생할 경우
- recoverable schedule
- cascadeless schedule
- strict schedule

## 스케쥴 복습

### 상황: K가 H에게 20만원을 이체할 때, H도 본인 계좌에 30만원을 입금한다면 여러 형태의 실행이 가능할 수 있음
- Case 1
    - tx1
    - tx2

- Case 2
    - tx2
    - tx1
> Case 1 && Case 2: Serial Schedule: 트랜잭션을 각각 수행함

- Case 3
    - tx1
    - tx2
    - tx1

- Case 4
    - tx1
    - tx2
    - tx1

> Case 3 && Case 4: Nonserial Schedule: 트랜잭션을 동시에 실행시킴
- 물론 겹쳐서 실행하는 경우는 여러 경우가 더 존재할 수 있음
- I/O 작업은 CPU가 노는 중이므로 이때 다른 트랜잭션 처리를 수행할 수 있음
- 성능: Serial < Nonserial
    - 단점: 이상한 결과가 나올 수 있음(Case 4 = 220만원)

### 지금까지는 Commit
- 지금까지는 모든 트랜잭션이 Commit을 함
- 이제는 트랜잭션이 Rollback을 수행할 경우 발생하는 상황에 대해서 알아보기

---

## Unrecoverable Schdule

### 상황: K가 H에게 20만원을 이체할 때, H도 본인 계좌에 30만원을 입금한다면
- tx1
    - r1(K) => 100만원
    - -20만원
    - w1(K) => 80만원

- tx2
    - r2(H) => 200만원
    - +30만원
    - w2(H) => 230만원
    - 주의: 아직 commit하지 않음!!

- tx1
    - r1(H) => 230만원: **여기가 문제**
        - tx2가 abort 되었으므로 여기는 tx2가 쓴 데이터를 읽었으므로 유효한 작업이 아니게 됨
    - w1(H) => 250만원
    - c1

- tx2
    - abort: rollback 수행
    - rollback(H) => **200만원**
        - tx2가 수행되기 전으로 돌려야하므로 그 때 H의 잔액은 200만원이였음!!

- 결과: K(80만원) / H(200만원): 증발해버린 20만원...

> tx2는 더 이상 유효하지 않으므로 tx2가 write 했던 H_Balance를 읽은 tx1도 롤백해야함!!

### 문제
- tx1도 롤백을 수행해야하는데
- tx1은 이미 commit 했기 때문에 durability 속성 때문에 rollback을 할 수 없음!!

### Unrecoverable Schdule 개념
- 스케쥴 내에서 commit된 트랜잭션이 rollback된 트랜잭션이 write했었던 데이터를 읽은 경우
- 이런 스케쥴은 rollback을 해도 이전 상태로 회복 불가능할 수 있기 때문에 이런 스케쥴은 DBMS가 허용하면 안됨

---

## Recoverable Schedule

### 상황: K가 H에게 20만원을 이체할 때, H도 본인 계좌에 30만원을 입금한다면
- tx1
    - r1(K) => 100만원
    - -20만원
    - w1(K) => 80만원

- tx2
    - r2(H) => 200만원
    - +30만원
    - w2(H) => 230만원

- tx1
    - r1(H) => 230만원
    - +20만원
    - w1(H) => 250만원

> 자 이제 어떻게 해야할까? 바로 commit 순서가 중요함!!

- tx1이 tx2에 의존중
    - tx2가 commit
    - 이후 tx1 commit

- 만약 tx2에서 abort가 발생하면
    - tx2 rollback이 발생하게 되면서 
    - tx1도 같이 abort가 발생하면서 rollback 되어야함

> 결론 1: 이 스케쥴에서 tx1은 tx2에 의존을 하고 있으니 tx2에 대한 commit/rollback 여부를 반드시 기다려야함

> 결론 2: 스케쥴 내에서 그 어떤 트랜잭션도 자신이 읽은 데이터를 write한 트랜잭션이 먼저 commit/rollback 전까지는 commit하지 않는 경우 => **Recoverable Schedule**
- DBMS는 이런 스케쥴만 허용해야함!!

### Cascading Rollback
- 하나의 트랜잭션이 rollback하면 다른 트랜잭션도 rollback해야하는 경우
- 여러 트랜잭션의 rollback이 연쇄적으로 일어나면 처리하는 비용이 많이 들게 된다...

### 해결방법
- 데이터를 write한 트랜잭션이 commit/rollback 한 뒤에 데이터를 읽는 schedule만 허용을 하자!!
- 예:
    - w2(H)
    - r1(H)
    - 이런 경우 w2, 즉, tx2가 쓴 데이터를 tx1이 읽어야하는 경우 아직 commit되지 않았기 때문에 이런 스케쥴은 허용하면 안됨

- 예:
    - r2(H)
    - w2(H)
    - **c2**

    - r1(H)
    - w1(H)
    > 이런 스케쥴만 허용을 하자!!

### 이렇게 해서 나오는 결과
- tx1
    - ...

- tx2
    - ...
    - rollback

- tx1
    - 현재는 tx2가 쓴 데이터를 의존하고 있지 않으므로
    - tx2만 롤백하고
    - tx1은 그냥 수행하면 됨(rollback하지 않아도 됨) => Cascading Rollback 비용 해결!!

### Cascadeless Schedule(avoid cascading rollback)
- 스케쥴 내에서 어떤 트랜잭션도 commit되지 않은 트랜잭션들이 write한 데이터를 read 않는 경우

---

## Cascadeless Schedule에서 발생할 수 있는 문제

### 상황: H 사장님이 3만원이던 피자 가격을 2만원으로 낮추려는데 K 직원도 동일한 피자의 가격을 실수로 1만원으로 낮추려 했을 때 발생할 수 있는 스케쥴
- 피자 가격: 3만원
- tx1
    - w1(p) = 1만원

- tx2
    - w2(p) = 2만원
    - c2

- tx1
    - abort: rollback 수행
    - tx1이 시작되기 전의 가격으로 원복을 시켜줘야함
    - 문제 발생: 피자 가격이 3만원으로 rollback됨

> 문제: 위 스케쥴은 cascadeless schedule을 위반하지 않았음!!. 근데 문제가 발생함!!
- 스케쥴 내에서 어떤 트랜잭션도 commit되지 않은 트랜잭션들이 write한 데이터를 read하지 않았음
- 보강이 필요함
    - commit되지 않은 트랜잭션들이 write한 데이터는 쓰지도 않아야함

> 이것을 **Strict Schedule**이라고 함

---

## Strict Schedule

### 예시
- tx1
    - w1(p) = 1만원
    - c1 / rollback1

- tx2
    - w2(p) = 2만원
    - c2

> 이런 스케쥴을 strict schedule이라고 함

### 장점
- 롤백을 할 때 recovery가 쉬움
- 그냥 트랜잭션 이전 상태로 돌아가면 됨
- 위의 예시를 보면
    - tx1이 abort => p = 3만원으로 롤백
    - tx2는 그냥 p = 2만원으로 commit 가능!!

---

## 최종 정리

### unrecoverable schedule
- 중간에 롤백이 발생하면 회복 불가능할 수도 있기 때문에 DBMS 자체에서 막아야함

### recoverable schedule
- 중간에 롤백이 발생하더라도 회복 가능하기 때문에 DBMS에서 허용
    - 근데 문제가 있음
    - Cascading Rollback 비용이 발생함

### Cascadeless Schedule(avoid cascading rollback)
- Cascading Rollback 비용을 막기 위해
- 스케쥴 내에서 어떤 트랜잭션도 commit되지 않은 트랜잭션들이 write한 데이터를 read하지 않음
- 근데 문제가 발생할 수도 있음 (읽지 않고 쓰기만 한 경우!!)

### Strict Schedule
- 스케쥴 내에서 어떤 트랜잭션도 commit되지 않은 트랜잭션들이 write한 데이터를 write/read하지 않음

> Recoverable > Cascadeless > Strict 순으로 작아지는 범위

### 결론
- **concurrency control은 serializability와 recoverability를 제공해줌!!**
- 이와 관련된 트랜잭션 ACID 속성이 **Isolation**

---

## 추가사항

### 읽고 쓰는 것을 막는다의 실제 구현 => Lock
- 커밋되지 않은 트랜잭션이 쓴 데이터를 읽거나 쓰지 않는다는 규칙은 Lock을 사용함
    - Cascadeless 보장: 한 트랜잭션이 데이터에 write를 하면 해당 데이터는 배타적 락(Exclusive Lock)을 걸어버림
    - 이 데이터를 다른 트랜잭션이 read하려고 하면 락이 풀릴 때(commit/rollback 될 때가지)까지 대기
    - Strict 보장: write 뿐만 아니라 다른 트랜잭션이 write를 시도하는 것조차 X-Lock으로 막아버림

- 성능과의 타협
    - Lock을 걸어 대기시키기 때문에 필연적으로 동시 처리 성능이 떨어짐

### 피자 가게 예시
- 데이터를 읽지 않고, 무작정 써버리는 경우를 **Blind Write**라고 함
- 스프링/JPA 관점:
    - JPA를 사용하면 기본적으로 엔티티를 먼저 조회(findById)한 뒤에 변경 감지를 통해 수정함
    - 순수한 의미로 Blind Write는 잘 발생하지 않음
    - 근데 JPQL로 Update ... 를 날리면 Blind Write가 발생할 수 있어 주의가 필요함

### MVCC 등장
- 문제 제기: Strict를 지키려다 보니 락 때문에 성능이 느려짐
- 현재 DB의 해결책이 바로 MVCC
    - 커밋 안 된 데이터를 락 걸고 못 읽게 하는 대신
    - 데이터의 이전 버전(이미 커밋된 안전한 과거 데이터)를 읽게 해주면 대기 시간도 없고 Cascading Rollback도 안 생기지 않을까에서 시작
    - **근데 이 때문에 문제가 발생할 수 있는데(이전 데이터를 읽으므로) 이것은 Isolation Level에서 다룸

### Isolation Level과의 관계
- 격리 수준(Isolation Level) 중 READ UNCOMMITTED가 바로 노트의 처음에 등장한 Cascading Rollback을 유발하는 원인입니다. 커밋되지 않은 데이터를 읽는 것을 허용(Dirty Read)
- READ COMMITTED 격리 수준부터는 커밋된 데이터만 읽도록 강제하므로 이론상 Cascadeless Schedule을 보장