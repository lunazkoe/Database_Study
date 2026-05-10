# S19. DB MVCC 개념 설명

## MVCC 등장 배경

### 저번 시간
- Lock-based concurrency control
    - read - write lock
    - 기본적으로 read-read는 허용하되, 나머지는 전부 허용하지 않음 

### MVCC (MultiVersion Concurrency Control)
- read-write lock까지는 허용할 수 있게 바꾸어보자는 아이디어에서 출발

---

## MVCC 설명 (예제와 함께)

### 상황
- x = 10
- tx1: x를 읽음
- tx2: x를 50으로 바꿈

- tx2
    - write_lock(x)
    - w2(x = 50)
        - 여기서 x = 50이라는 걸 써놓음(자신의 트랜잭션만 아는 공간에)

- tx1
    - read_lock(x)
    - r1(x) = 10

> 중요한 특징: MVCC: commit된 데이터만 읽음이 기본 전제

- tx2
    - c2
        - 이때 x = 50이라는 걸 써놓은 걸 실제 DB에 반영
        - 실제 x = 50으로 변경됨
    - unlock(x)
        - recoverability를 위해 commit할 때 write_lock을 unlock을 함

- tx1
    - read(x) = ??
        - isolation level에 따라 다름
        - read committed: read하는 시간을 기준으로 그전에 commit된 데이터를 읽음
            - read(x) = 50으로 읽음(MySQL, PostgreSQL)
        - repeatable read: tx 시작 시간 기준으로 그전에 commit된 데이터를 읽음
            - read(x) = 10으로 읽음
            - RDBMS마다 다름(트랜잭션 시작 시점 | 최초의 read 시점 | 어떤 operation 시작 시점)
            - MySQL / PostgreSQL 모두 동일하게 동작

### 특징
- 데이터를 읽을 때 특정 시점 기준으로 가장 최근에 commit된 데이터를 읽음
- 데이터 변화(write) 이력을 관리함
- read / write는 서로 block하지 않음

### MySQL의 Consistent Read
- 특정 시점 기준으로 가장 최근에 commit된 데이터를 읽는 경우

### Serializable에서는?
- MySQL
    - MVCC로 동작하기 보다는 Lock으로 동작함

- PostgreSQL
    - SSI 기번이 적용된 MVCC로 동작함

### Read Uncommitted
- MVCC는 committed된 데이터를 읽기 때문에 이 level에서는 보통 MVCC가 적용되지 않음
- MySQL
    - read committed / repeatable read에서 MVCC를 적용함

- PostgreSQL
    - read uncommitted level이 존재는 하지만
    - read committed level처럼 동작함

---

## PostgreSQL에서 lost update 문제와 해결

### 상황
- x = 50 | y = 10
- tx1: x가 y에 40을 이체함
- tx2: x에 30을 입금함
> 정상동작: x = 40 | y = 50

### MVCC를 적용한다면 (각 트랜잭션 레벨을 read committed)
- tx1
    - read_lock(x)
    - r1(x) = 50
    - -40
    - write_lock(x)
    - w1(x = 10)
        - 자신만의 공간에 x = 10

- tx2
    - read_lock(x)
    - r2(x) = 50
    - write_lock(x): 아직 tx1이 write_lock을 가지고 있음 / blocking

- tx1
    - read_lock(y)
    - r1(y) = 10
    - write_lock(y)
    - w1(y = 50)
        - 자신만의 공간에 y = 50
    - c1
        - 실제 DB에 x = 10 | y = 50 반영

- tx2
    - w2(x = 80)
        - 자신만의 공간에 x = 80
    - c2
        - 실제 DB에 x = 80 반영

> 최종 결과: x = 80 | y = 50 => 잘못된 결과(Lost Update!!)

### Isolation Level을 변경해서 해결해보자!! - tx2에 Repeatable Read tx1은 유지
- tx1
    - read_lock(x)
    - r1(x) = 50
    - -40
    - write_lock(x)
    - w1(x = 10)
        - 자신만의 공간에 x = 10

- tx2
    - read_lock(x)
    - r2(x) = 50
    - write_lock(x): 아직 tx1이 write_lock을 가지고 있음 / blocking

- tx1
    - read_lock(y)
    - r1(y) = 10
    - write_lock(y)
    - w1(y = 50)
        - 자신만의 공간에 y = 50
    - c1
        - 실제 DB에 x = 10 | y = 50 반영 => 여기까지는 동일함

- tx2
    - w2(x = 80): 이 부분이 실패함
        - **PostgreSQL은 같은 데이터에 대해서 먼저 update한 tx가 commit되면 나중 tx는 rollback됨**
    - rollback

> 결과: x = 10 | y = 50이 됨 => 예상했던 정상 결과는 아니지만 하나는 성공 / 하나는 실패 했기 때문에 정상적이라고 볼 수 있음. 다시 시도하면 성공하니깐 - *First Updater Win**

### 알아야할 것
- 각 트랜잭션마다 Isolation Level을 따로 줄 수 있음

### 그렇다면 tx1은 read committed로 두면 정말 괜찮은게 맞을까?
- tx2
    - r2(x) = 50

- tx1
    - r1(x) = 50

- tx2
    - w2(x = 80)
        - 자신만의 공간에 x = 80

- tx1
    - w1(x = 10): tx2가 write_lock을 쥐고 있음

- tx2
    - c2
        - x = 80이 실제 DB에 적용
        - write_lock 반환

- tx1
    - w1(x = 10)
        - 자신만의 공간에 x = 10
    - r1(y) = 10
    - w1(y = 50)
        - 자신만의 공간에 y = 50
    - c1
        - x = 10 | y = 50 실제 DB에 반영
        > Lost Update 발생
    
> 해결: tx1도 repeatable read로 변경

---

## MySQL에서 lost update 문제

### MySQL에서는?
- 같은 데이터에 대해서 먼저 update한 tx가 commit되면 나중 tx는 rollback됨
- 이 개념이 MySQL에서는 없음
- 따라서 실패하지 않음!! => 그렇다면 어떻게 해결??? => 이건 다음 시간에!!

---

## 추가사항
```txt
1. "자신만의 공간에 써놓음"에 대한 기술적 정의
피드백: 설명하신 "자신의 트랜잭션만 아는 공간"이라는 표현은 MVCC의 개념을 이해하는 데 아주 훌륭한 비유입니다.

엄밀한 보완: 실무나 면접에서는 이 '공간'이 실제로 어디인지를 아는 것이 중요합니다.

MySQL (InnoDB): Undo Log(언두 로그)라는 별도의 공간에 기존 데이터를 복사해 둡니다. 쓰기 작업은 실제 테이블(Buffer Pool)에 먼저 반영하고, 변경 전 과거 데이터를 Undo Log에 남겨서 다른 트랜잭션이 이를 읽게 만듭니다.

PostgreSQL: 별도의 Undo Log 공간을 쓰지 않고, 기존 테이블의 같은 페이지 내에 새로운 버전의 행(Tuple)을 하나 더 생성합니다. (이를 Append-only 방식이라고 하며, xmin, xmax라는 메타데이터로 버전을 관리합니다.)

2. 특정 시점(Snapshot)의 기준
피드백: Read Committed와 Repeatable Read가 데이터를 읽어오는 '시점'이 다르다는 것을 정확히 파악하셨습니다.

엄밀한 보완: 이를 데이터베이스 용어로 다음과 같이 부릅니다.

Read Committed: 각 쿼리(Statement)가 실행될 때마다 스냅샷을 새로 찍습니다. (Statement-level Read View)

Repeatable Read: 트랜잭션이 시작되고 첫 번째 쿼리(Read)가 실행되는 시점에 단 한 번만 스냅샷을 찍고, 트랜잭션이 끝날 때까지 그 스냅샷만 봅니다. (Transaction-level Read View)

3. PostgreSQL의 Lost Update 방어와 에러 메시지
피드백: PostgreSQL에서 Repeatable Read 레벨일 때 나중에 실행된 트랜잭션이 실패(Rollback)한다는 내용, 즉 First-Committer Wins (또는 First-Updater Wins) 원칙을 아주 정확히 짚으셨습니다.

엄밀한 보완: 이때 PostgreSQL은 나중에 업데이트를 시도한 트랜잭션(tx2)에게 다음과 같은 에러를 던지며 강제로 Abort 시킵니다.

ERROR: could not serialize access due to concurrent update

실무 애플리케이션 개발자는 이 에러를 캐치(Catch)하면 트랜잭션을 처음부터 다시 시도(Retry)하도록 로직을 짜야 합니다. (노트에 "다시 시도하면 성공하니깐"이라고 쓰신 부분이 바로 이 애플리케이션 레벨의 Retry 로직을 의미합니다.)

4. MySQL의 Lost Update와 다음 시간에 대한 기대감
피드백: "MySQL은 이 개념이 없어서 실패하지 않음! 어떻게 해결?"이라는 마지막 문단이 정말 예술입니다. 학습의 흐름을 주도하고 계시네요.

엄밀한 보완: 정확합니다. MySQL의 InnoDB는 Repeatable Read 레벨이라 하더라도 Lost Update를 자동으로 막아주지 못합니다. (PostgreSQL과 같은 충돌 감지 로직이 없습니다.) 따라서 개발자가 직접 다른 방식을 동원해야만 합니다.
```