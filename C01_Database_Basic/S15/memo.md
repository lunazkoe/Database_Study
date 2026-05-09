# S15. Concurrency Control 기초 이론: Schedule과 Serializability 설명, 트랜잭션 Isolation 보장을 위한 이론

## 트랜잭션이 동시에 실행될 때 실행 순서들

### 상황: K가 H에게 20만원을 이체할 때 H도 ATM에서 본인 계좌에 30만원을 입금한다면 여러 행태의 실행이 가능할 수 있음
- 트랜잭션 1(K -> H 20만원 이체)
- 트랜잭션 2(H -> H 30만원 이체)

### Case 1
- 트랜잭션 1 실행 완료
- 트랜잭션 2 실행 완료

### Case 2
- 트랜잭션 2 실행 완료
- 트랜잭션 1 실행 완료

### Case 3
- 트랜잭션 1 실행
    - read(K_balance) => 100만원
    - -20만원
    - write(K_balance = 80만원)

- 트랜잭션 2 실행 완료
    - read(H_balance) => 200만원
    - +30만원
    - write(H_balance = 230만원)
    - commit

- 트랜잭션 1 실행 완료
    - read(H_balance) => 230만원
    - +20만원
    - write(H_balance = 250만원)
    - commit

### Case 4
- 트랜잭션 1 실행
    - read(K_balance => 100만원)
    - -20만원
    - write(K_balance = 80만원)
    - read(H_balance) => 200만원

- 트랜잭션 2 실행 완료
    - read(H_balance) => 200만원
    - +30만원
    - write(H_balance = 230만원)
    - commit

- 트랜잭션 1 실행 완료
    - 문제발생: **트랜잭션 1은 트랜잭션 2가 실행 완료되었다는 사실을 모르고 이전에 read한 값을 기준으로 진행**
    - +20만원
    - write(H_balance = 220만원)
    - commit

- 문제: 30만원은 없어져버림
- 이런 상황을 **Lost Update**라고 함
    - 업데이트된 결과가 사라져버림

> **결론: 특정 상황에서 여러 실행 Case가 존재하고, 특정 Case의 경우, 문제가 발생할 수 있음을 확인**

---

## 실행 순서 간략화해서 나타내기

### Operation
- read / write 각각을 Operation이라고 부름
- 이거를 간소화해보자!! (Case 4 기준)
```
- 트랜잭션 1 실행
r1(K)
w1(K)
r1(H)

- 트랜잭션 2 실행 완료
r2(H)
w2(H)
c2

- 트랜잭션 1 실행 완료
w1(H)
c1
```

---

## Scheduled

### 개념
- 여러 트랜잭션들이 동시에 실행될 때 각 트랜잭션에 속한 operation들의 실행 순서
- 각 트랜잭션 내의 operation들의 순서는 바뀌지 않음
    - 이게 무슨 말이냐
    - r1(K) -> w1(K) -> r1(H) -> w1(H): 이 순서는 바뀌지 않는다라는 것

### Case 1 && Case 2
- 각 트랜잭션이 겹치지 않고, 한 번에 하나씩 실행되는 스케쥴
- 이것을 **Serial Schedule**이라고 부름!!
    - 순차적으로 실행된다는 의미

### Case 3 && Case 4
- 각 트랜잭션들이 겹쳐서 실행되는 스케쥴
- 이것을 **Nonserial Schedule**이라고 부름!!
    - 순차적으로 실행되지 않는다는 의미

### Serial Schedule의 성능
- r2(H)
    - I/O 작업
    - I/O 작업을 하는 동안 CPU는 놀고 있다는 의미
        - 이 때, 다른 트랜잭션을 시작할 수도 있음
        - 근데 Serial은 다른 트랜잭션을 시작하지 않음

- w2(H)
    - I/O 작업

- 트랜잭션 2가 끝나면 트랜잭션 1이 쭉 실행됨
> 성능: 이상한 상황이 발생할 경우는 없지만, 한 번에 하나의 트랜잭션만 실행되기 때문에 좋은 성능을 낼 수 없고, 현실적으로 사용할 수 없는 방식!!

### Nonserial Schedule의 성능
- r2(H): 220만원을 읽음
    - I/O 작업
    - r1(K): 트랜잭션 1을 실행: 100만원을 읽음
    - I/O 작업 완료

- w2(H): 250만원으로 씀
    - I/O 작업
    - w1(K): 트랜잭션 1을 실행: 80만원으로 씀

- c2
- 나머지 트랜잭션 1번을 실행
> 성능: 트랜잭션들이 겹처서 실행되기 때문에 동시성이 높아져서 같은 시간 동안 더 많은 트랜잭션들을 처리할 수 있음!!

> 결론: Nonserial 승!!

### Nonserail Schedule의 단점
- 트랜잭션들이 어떤 형태로 겹처서 실행되는지에 따라 **이상한 결과**가 나올 수 있음!!
- Case 4: H의 최종값이 220만원이 됨 (250만원이 되어야함!!)

### 고민거리
- 성능 때문에 여러 트랜잭션들이 겹쳐서 실행할 수 있으면 좋음
- 근데 이상한 결과가 나오는 건 싫음!!
- 이제 nonserial로 실행해도 이상한 결과가 나오지 않을 수 있는 방법을 연구하기 시작

### 아이디어
- serial schedule과 동일한 nonserial scehdule을 실행하면 됨!!
- 그렇다면 스케쥴이 동일하다는 의미가 무엇인지 정의하는 것이 필요함!!

---

## Conflict

### 개념
- of two operations
- 아래 세 가지 조건을 만족하면 conflict라고 부름
    1. 서로 다른 트랜잭션 소속
    2. 같은 데이터에 접근
    3. 최소 하나는 write operation

### 예시: Case 3
- r1(K)
- w1(K)
- r2(H): 
- w2(H)
- c2
- r1(H)
- w1(H): 
- c1

> r2(H), w1(H)는 conflict함
- 서로 다른 트랜잭션 소속
- 같은 데이터(H)에 접근
- 최소 하나는 write operation(w1(H))
- 이것을 **read-write conflict**라고 함

> w2(H), w1(H)는 conflict함
- 이것을 **write-srite conflict**라고 함

> Case 3: r2(H)/w1(H) // w2(H)/w1(H) // w2(H)/r1(H) 총 세 개의 conflict가 발생함

### 왜 중요할까?
- conflict operation은 순서가 바뀌면 결과도 바뀜
- w2(H = 230)/r1(H = 230)
    - 이 두개의 순서가 바뀌면
    - r1(H = 200)
    - w2(H = 230)

---

## Conflict Equivalent

### 개념
- for two schedules
- 아래 두 조건을 모두 만족하면 conflict equivalent
    1. 두 스케쥴은 같은 트랜잭션을 가진다
    2. 어떤 conflict operations의 순서도 양쪽 schedule 모두 동일함

### 예시: Case 3 && Case 2
- 둘 다 같은 트랜잭션을 가짐 (tx1, tx2): 1번 만족
- conflict operations들의 순서
    - r2(H)/w1(H)의 순서가 서로 같음
    - w2(H)/r1(H)의 순서가 서로 같음
    - w2(H)/w1(H)의 순서가 서로 같음

> Case 3 && Case 2는 conflict equivalent
- Case 2는 serial schedule!!

---

### Conflict Serializable

### 개념
- nonserial이 serial과 conflict equivalent일 때 이것을 **Conflict Serializable**라고 부름
- 때문에 Case 3는 Nonserial임에도 불구하고, 정상적인 결과를 낼 수 있었음

### Case 4 && Case 2
- 둘 다 같은 트랜잭션을 가짐(tx1, tx2): 1번 만족
- conflict operations들의 순서
    - Case 4
        - r1(H)/w2(H)
    - Case 2
        - w2(H)/r1(H)
    - 같은 순서가 아님 => Not Conflict Equivalent!!
    - 그래도 아직 단정하기 이름 => Serial 스케쥴인 Case 1이 남아있음
        - 이 친구와 Conflict Equivalent하면 괜찮음!!

### Case 4 && Case 1
- 둘 다 같은 트랜잭션을 가짐(tx1, tx2): 1번 만족
- conflict operations들의 순서
    - r1(H)/w2(H)의 순서가 서로 같음
    - Case 4
        - r2(H)/w1(H)
    - Case 1
        - w1(H)/r2(H)
    - 같은 순서가 아님 => Not Conflict Equivalent!!

### 결론
- tx1, tx2의 serial은 2개가 존재
- 근데 Case 4는 그 두개와 not conflict equivalent함
> 결론: Case 4는 Not Conflict Serializable!! => 이상한 결과가 나옴!!

---

## 정리

### 고민거리
- 성능 때문에 여러 트랜잭션이 겹쳐서 실행할 수 있으면 좋다
- 하지만 이상한 결과가 나오는 것은 싫다

### 해결책
- **conflict serializable한 nonserial schedule을 허용하자!!**

### 구현
- 여러 실행될 때마다 해당 스케쥴이 conflict serializable인지 확인!!
    - 하지만 이건 현실성이 부족함
    - 요청이 많이 들어오면(tx1, tx2, ....) 이걸 일일히 검증할 수 없음

### 실제 구현
- 여러 트랜잭션을 동시에 실행해도 스케쥴이 conflict serializable하도록 보장하는 **프로토콜**을 적용!!

---

## 주요 개념 최종 정리

### 스케쥴
- 어떤 스케쥴이
- 시리얼 스케쥴과 **동일하다면(conflict equivalent)**
- 그 스케쥴은 **conflict serializable**하다고 말할 수 있음(또는 serializability라는 속성을 가짐)

### 동일하다는 개념
- conflict equivalent
    - 위에서 설명
- view equivalent => view serializable(view serializability)
    - 이런 것도 있음
    - Conflict Serializable보다 더 많은 스케쥴을 허용하기 위한 것
    - 근데 사용은 안함
        - 이걸 판별하는 알고리즘은 NP-Complete

### Concurrency Control
- 그 어떤 스케쥴도
- Serializable하게 만드는 역할을 수행하는 것!!
- 이것과 밀접하게 연관된 속성이 **Isolation**
    - 여러 트랜잭션이 동시에 실행될 때도 혼자 실행되는 것처럼 동작하게 만드는 것

### Isolation Level
- isolation을 너무 엄격하게 지키면 성능이 줄어듦
- 이것을 완하시키기 위해서 **Isolation Level**이 등장함!!

---

## 이후 다룰 내용

### Recoverability
- Serializability만큼 중요한 개념
- Rollback을 했을 때에 대한 내용!!
- 지금까지는 Commit 했을 때에 대한 내용!!
- 이후 Isolation Level / Scheduling Protocol에 대해서 학습할 것

--- 

## 추가사항

### Lost Update에 대한 Spring Data JPA 방어책
- 주로 낙관적 락을 사용하여 해결함
- 엔티티에 @Version이 붙은 필드를 추가(version: 정수값)
    - 트랜잭션 1, 2가 version=1인 데이터를 읽어감
    - 트랜잭션 2가 먼저 업데이트를 치고 커밋하면 DB의 version=2가 됨
    - 이후 트랜잭션 1이 업데이트를 시도할 때, version=1로 업데이트를 침
    - 근데 DB에서 version=2로 업데이트가 되었기 때문에 Lost Update가 발생하였음을 인지하고 롤백시킴
