# S14. 데이터베이스 트랜잭션과 ACID

## 예제를 통해 배워보는 트랜잭션 개념

### 상황: J가 H에게 20만원을 이체한다면 각자 계좌는 어떻게 변경되어야 할까?
- J: 100만원 / H: 200만원
- J의 계좌에서 -20만원
    - SQL `update account set balance = balance - 200000 where id = 'J';`
- H의 계좌에서 +20만원
    - SQL `update account set balance = balance + 200000 where id = 'H';`
> 주의할 점: 위 두개의 SQL문이 두 개다 성공해야 이체가 성공적으로 이루어졌다고 말할 수 있음!!
- 만약 하나라도 실패하면 돈이 복사 혹은 삭제되는 어마무시한 문제가 발생함!!
- **즉, 이체라는 작업은 둘 다 정상 처리되어야만 성공하는 단일 작업**

---

## 트랜잭션의 개념 정리

### 트랜잭션이란?
- 단일한 논리적인 작업 단위(a single logical unit of work)
- 논리적인 이유로 여러 SQL문들이 단일 작업으로 묶어서 나눠질 수 없게 만든 것이 transaction
- transaction의 SQL문들 중 일부만 성공해서 DB에 반영되는 일은 일어나지 않음

---

## SQL을 통한 트랜잭션 사용해 보기

### 예제 1: J가 H에게 20만원을 이체
```sql
select * from account;
start transaction; 
// - DB에 트랜잭션을 시작할거야라고 알려주는 것
update account set balance = balance - 200000 where id = 'J';
update account set balance = balance + 200000 where id = 'H';
commit;
// - 지금까지 작업한 내용을 DB에 영구적으로 저장하라는 의미
// - transaction을 종료한다는 의미
```

### 예제 2: J가 H에게 30만원을 이체
```sql
select * from account;
start transaction;
update account set balance = balance - 300000 where id = 'J';
select * from account;
// - J의 계좌에서 300000원이 빠져나간 상황
rollback;
// - 지금까지 작업들을 모두 취소하고 transaction 이전 상태로 되돌린다는 의미
// - transaction을 종료한다는 의미
select * from account;
// - start transaction을 하기 이전의 계좌 상황으로 돌아감
```

---

## AUTO COMMIT

### AUTO COMMIT이란
- 각각의 SQL문을 자동으로 transaction 처리 해주는 개념
- SQL문이 성공적으로 실행하면 자동으로 commit함
- 실행 중에 문제가 있었다면 알아서 rollback함
- MySQL에서는 default로 autocommit이 enabled되어 있음
- 다른 DBMS에서도 대부분 같은 기능을 제공함

### 예제
```sql
select @@autocommit
// - 현재 autocommit이 활성화되어있는지 여부를 확인
//      - 1: true / 0: false
insert into account values ('W', 100000); 
// - 현재 autocommit이 활성화되어있는 상태이기 때문에
// - insert문을 실행하면 자동으로 commit이 되면서 account 테이블에 ('W', 1000000) 데이터가 영구적으로 저장됨
select * from account;
// - 이제 반영된 것을 확인할 수 있음
```

### autocommit을 비활성화한다면?
```sql
set autocommit=0;
// - autocommit 비활성화
delete from account where balance <= 1000000;
// - 예상 결과: H의 계좌만 남아있어야 함
select * from account;
// - H의 계좌만 남아있음
// - 주의: 아직 commit이 된 상태가 아님
//      - autocommit을 off한 후에 delete를 했기 때문에
//      - rollback을 한다면 다시 이전 상태로 돌아갈 수 있음
rollback;
select * from account;
// - J, H, W의 계좌를 전부 확인할 수 있음
```

### start transaction의 진실
```sql
start transaction;
update account set balance = balance - 200000 where id = 'J';
update account set balance = balance + 200000 where id = 'H';
commit;
```
- **MySQL에서는**
    - start transaction 실행과 동시에 autocommit은 off됨
    - **commit / rollback과 함께 트랜잭션이 종료되면 원래 autocommit 상태로 돌아감!!**
    - 주의: 원래 autocommit 상태로 돌아감
        - 근데 autocommit을 수동으로(start transaction으로 하지 않고 set autocommit=0) 변경한 경우
        - commit을 해도 원래 상태로 돌아가지 않음 => 커넥션 풀을 재사용해야하기 때문에 다른 것도 그대로 반영될 수 있어서 이런 경우 반드시 `set autocommit=1`으로 변경해야함!!

---

## 트랜잭션 사용패턴

### 일반적인 transaction 사용 패턴
1. 트랜잭션을 시작한다.
2. 데이터를 읽거나 쓰는 등의 SQL문들을 포함해서 로직을 수행한다.
3. 일련의 과정들이 문제없이 동작했다면 트랜잭션을 commit한다.
4. 중간에 문제가 발생했다면 트랜잭션을 rollback한다.

---

## Java에서 트랜잭션 사용 예제

### 예제
```java
try {
    connection.setAutoCommit(false); // == set autocommit=0; / != start transaction
    ... // update 1
    ... // update 2
    connection.commit(); // commit
}
catch {
    connection.rollback(); // rollback
}
finally {
    connection.setAutoCommit(true); // 위에서 말한 수동으로 조작할 경우 기본적으로 다시 활성화 시켜줘야함
}
```

### 위 코드의 문제점(이건 DB라기보다는 개발 관점)
- 현재 코드는 비즈니스 로직(이체)과 DB 트랜잭션 관련 코드가 함께 있음
- 트랜잭션과 관련된 코드는 따로 떼어내서 사용하고 싶음!! => Spring Framework 지원

---

## Spring에서 트랜잭션 사용 예제

### 예제
```
@Transactional
// - 이 메서드를 실행할 때 트랜잭션을 시작하고 자동으로 commit / rollback / autocommit=1까지 수행해줌
public void transfer(String fromId, String toId, int amount) {
    ... // update 1
    ... // update 2
}
```
- 비즈니스 로직(이체)만 작성하면 됨

---

## ACID

### ACID란
- Atomicity
- Consistency
- Isolation
- Durability
> 트랜잭션이 어떤 속성을 지녀야한느지 나타내는 개념들

---

## Atomicity

### 상황: J가 H에게 20만원을 이체
- 둘 다 성공해야만 의미있는 단일 작업
    - 모두 성공하거나
    - 모두 실패하거나
    - 이렇게 동작해야함

- All or Nothing
- 트랜잭션은 **논리적으로 쪼개질 수 없는 작업 단위(Atomicity)**이기 때문에 내부 SQL문들이 모두 성공해야함
- 중간에 SQL문이 실패하면 지금까지의 작업을 모두 취소하여 아무 일도 없었던 것처럼 rollback해야함

### DB와 개발자가 담당하는 부분 차이
- commit 실행 시 DB에 영구적으로 저장하는 것은 DBMS가 담당하는 부분
- rollback 실행 시 이전 상태로 되돌리는 것도 DBMS가 담당하는 부분
- **개발자는 언제 commit을 하거나 rollback을 할지를 챙겨야 함**

---

## Consistency

### 상황: J가 H에게 100만원을 추가로 이체한다면 어떻게 될까?
- J: 80만원 / H: 220만원 (100, 200만원 씩 들고 있을 때, 위에서 20만원을 이체한 후 상황)
```sql
update account set balance = balance - 1000000 where id = 'J';
// - 문제: -20만원이 됨(말이 안됨)
update account set balance = balance + 1000000 where id = 'H';
// - 320만원
```
- 문제 상황: balance는 데이블 조건에 0 이상이여야한다는 제약 사항이 있음
- 따라서 update 1번은 실패를 하게 됨!!  
- 이 상황에서 트랜잭션을 굳이 계속 실행시킬 이유가 없음(어차피 하나라도 실패하면 rollback을 해야함)

### Consistency란
- 트랜잭션은 DB 상태를 consistent 상태에서 또 다른 consistent 상태로 바꿔줘야함
- constraint, trigger 등을 통해 DB에 정의된 rules을 트랜잭션이 위반했다면 **rollback**을 해주어야함
- 트랜잭션이 DB에 정의된 rule을 위반했는지 DBMS가 commit 전에 확인하고 알려줌
- 그 외 application 관점에서 트랜잭션이 consistent하게 동작하는지 개발자가 챙겨야함

---

## Isolation

### 상황: J가 H에게 20만원을 이체할 때 H도 ATM에서 본인 계좌에서 30만원을 입금한다면?
- J
    - read(balance) => 100만원
    - -20만원
    - write(balance = 80만원)

- H
    - read(balance) => 200만원
    - 근데 이 타이밍에 이 타이밍에 H가 본인 계좌에 30만원을 입금 (30만원 입금에 대한 트랜잭션이 발생)
        - read(balance) => 200만원
        - +30만원
        - write(balance = 230만원)
    - 이전 트랜잭션은 이 트랜잭션이 실행되었는지 모름!! (read(balance) => 200만원인 상태)
    - +20만원
    - write(balance = 220만원)
    - ???: 30만원은 어디로??

- 즉, 여러 트랜잭션들이 동시에 실행하니 문제가 발생!!

### Isolation이란!!
- 여러 트랜잭션이 동시에 실행될 때도 혼자 실행되는 것처럼 동작하게 만듦
- DBMS는 여러 종류의 isolation level을 제공함
    - **level이 높으면(엄격하게 하면) DB의 퍼포먼스가 줄어들어버림**
- 개발자는 ioslation level 중에 어떤 level로 트랜잭션을 동작시킬지 설정할 수 있음
    - default로 설정된게 있음
    - 상황에 따라서 튜닝
- concurrency control의 주된 목표가 isolation!!

---

## Durability

### 영존성
- commit된 트랜잭션은 DB에 **영구적으로 저장**
- 즉, DB system에 문제(power fail or DB crash)가 생겨도 commit된 트랜잭션은 DB에 남아 있음
- 영구적으로 저장한다라고 할 때 일반적으로 비휘발성 메모리(HDD, SSD)에 저장함을 의미함
- 기본적으로 트랜잭션의 durability는 DBMS가 보장함

---

## 참고사항
1. 트랜잭션을 어떻게 정의해서 쓸지는 개발자가 정하는 것. 구현하려는 기능과 ACID 속성을 이해해야 잘 정의할 수 있음
2. 트랜잭션의 ACID와 관련해서 개발자가 챙겨야하는 부분들이 있음. DBMS가 모든 것을 알아서 해주는 것이 아님(디폴트가 있기해도 상황에 따라 튜닝)
3. 위 내용은 MySQL 기준임!!

---

## 추가사항

### Spring @Transactional의 숨겨진 함정
- 모든 예외를 롤백하지 않음 => 기본적으로 RuntimeException이 발생했을 때만 롤백함
- 파일 입출력에서 오류가 나면 체크 예외여서 커밋해버림
- 보완책: @Transactional(rollbackFor = Exception.class)로 롤백 대상 예외를 지정할 수 있음

### Isolation의 한계
- DBMS의 Isolation Level을 올린다고만해서 해결되지 않는 경우가 있음
- 격리 수준 튜닝 외에도 명시적인 락을 활용
    - 비관적 락: 내가 수정하는 동안 남들은 읽지도 못하게 잠금
    - 낙곽적 락: 데이터에 @Version을 매겨 갱신 충돌을 감지

### Durability를 보장하는 마법: WAL(Redo Log)
- 트랜잭션이 커밋될 때마다 무건운 하드디스크에 쓰는 건 성능상 불가능함
- MySQL은 변경사항을 빠르게 기록할 수 있는 Redo Log(WAL: Write-Ahead-Logging)에 먼저 기록을 함
- 커밋 시점에 이 로그만 안전하게 디스크에 기록해두면 나중에 전원이 나가도 재부팅 시 이 로그르 읽어와서 그대로 데이터를 복구함

### Connection Pool과 autocommit 첨언
- HikariCP는 커넥션을 반납할 때 남아있는 미완성 트랜잭션을 강제로 rollback()하고, autocommit=true 상태로 초기화한 뒤 풀에 집어넣음
    - 미완성 트랜잭션: 쿼리는 다 날렸는데 마지막에 commit을 하지 않은 경우
    - 이 내용은 HikariCP를 사용할 경우를 말하는 것임!!
    - Spring Framework의 @Transactional은 이걸 알아서 해줌


---

## 해본 것
```sql
start transaction;
update users set balanced = balanced - 1000000 where name='J';
update users set balanced = balanced + 1000000 where name='H';
commit;
```
- update 1번 쿼리에서 오류가 발생
- 이후 진행하고 commit하면 반영이됨
- 즉, start transaction을 진행하다가 오류가 난다고 롤백을 해주는게 아니라
- 예외(오류)를 만들어서 애플리케이션에서 이걸 잡아서 직접 롤백을 입력해주어야함
    - DBMS에서 처리하는 방법이 있는 것 같은데
    - 보통 애플리케이션에서 처리 => Spring은 @Transactional에서 RuntimeException일 경우 알아서 처리해줌
- 결론: start transaction이 롤백을 해주는 것은 아니다!!