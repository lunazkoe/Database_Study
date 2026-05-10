# S18. Lock을 활용한 Concurrency Control 
- write-lock
- read-lock
- 2PL(two-phase locking) protocol

## lock 설명 및 예제

### 상황 1
- x = 10
- tx1: x를 20으로 바꿈
- tx2: x를 90으로 바꿈

- tx1
    - w1(x = 20)
        - 이 과정은 단순히 값 하나 바꾸는 것보다 더 복잡한 과정(인덱스 처리 같은)

- tx2
    - w2(x = 90)

> 즉, 같은 데이터에 또다른 read/write가 있다면 예상치 못한 동작을 할 수 있음

> 해결: Lock을 사용해서 해결할 수 있음!! (운영체제의 그 락과 거의 동일)

### Lock을 사용
- tx1
    - write_lock(x): 쓰기 락을 획득

- tx2
    - write_lock(x): tx1이 쓰기 락을 가지고 있기 때문에 진행하지 못하고 기다림

- tx1
    - w1(x = 20)
    - unlock(x): 쓰기 락 해제 => 이제 tx2가 쓰기 락을 획득할 수 있음

- tx2
    - w2(x = 90)
    - unlock(x): 쓰기 락 해제

### 상황 2
- x = 10
- tx1: x를 20으로 바꿈
- tx2: x를 읽음

### Lock을 사용
- tx1
    - write_lock(x)

- tx2
    - read_lock(x): 읽기 위한 목적으로 락 획득 시도
        - 근데 tx1이 락을 가지고 있기 때문에 진행하지 못하고 기다림

- tx1
    - w1(x = 20)
    - unlock(x)

- tx2
    - r2(x) = 20
    - unlock(x)

---

## write-lock

### 개념 (or exclusive lock)
- 이름은 write-lock이지만
- **read/write(insert, modify, delete)를 할 때 사용함**
- 다른 tx가 같은 데이터를 read/write 하는 것을 원천 봉쇄

---

## read-lock

### 개념 (or shared lock)
- read할 때만 사용함
- 다른 tx가 같은 데이터를 read 하는 것을 **허용함**
    - 읽기를 하는 것은 전혀 문제될게 없음
- **다른 tx가 같은 데이터에 write 하는 것을 허용하지 않음**

---

## lock 관련 추가 예제

### 예제 1
- x = 10
- tx1: x를 20으로 바꿈
- tx2: x를 읽음

- tx2
    - read_lock(x)

- tx1
    - write_lock(x): read_lock이 잡혀있기 때문에 기다려야함

- tx2
    - read(x) = 10
    - unlock(x)

- tx1
    - write(x = 20)
    - unlock(x)

### 예제 2
- x = 10
- tx1: x를 읽음
- tx2: x를 읽음

- tx2
    - read_lock(x)
    - read(x): 
    - unlock()

- tx1
    - read_lock(x): read_lock은 다른 트랜잭션이 같은 데이터 읽기는 허용함!!
    - read(x): 위 두개는 동시에 됨(여기서는 따로 해야하는 것처럼 보임)
    - unlock(x)

---

## read-lock, write-lock 호환성

### 호환성
-  | read__lock | write_lock
- read_lock | O | X
- write_lock | X | X

---

## lock을 써도 생기는 이상한 현상 예제

### 상황 1
- x = 100 | y = 200
- tx1: x와 y의 합을 x에 저장
- tx2: x와 y의 합을 y에 저장

### 각 트랜잭션의 실행 순서
- tx1
    - read_lock(y)
    - read(y)
    - unlock(y)
    - write_lock(x)
    - read(x)
    - write(x = x + y)
    - unlock(x)

- tx2
    - read_lock(x)
    - read(x)
    - unlock(x)
    - write_lock(y)
    - read(y)
    - write(y = x + y)
    - unlock(y)

### 만약 serialization으로 동작한다면
- tx1, tx2
    - x = 300
    - y = 500

- tx2, tx1
    - x = 400
    - y = 300

### 이제 동시에 하면
- tx2
    - read_lock(x)
    - read(x) = 100
    - unlock(x)

- tx1
    - read_lock(y)

- tx2
    - write_lock(y): read_lock이 걸려있어서 대기함

- tx1
    - read(y) = 200
    - unlock(y)

- tx2
    - read(y) = 200
    - write(y = 300)
    - unlock(y)

- tx1
    - write_lock(x)
    - read(x) = 100
    - write(x = 300)
    - unlock(x)

> 결과: x = 300 / y = 300 => 이상한 현상 발생 => 우리가 방금 본 스케쥴은 nonserializable

### 발생 이유
- tx2는 y를 업데이트
- tx1은 x를 업데이트

- tx2가 x가 업데이트 되기 전에 읽음
- tx1도 y가 업데이트 되기 전에 읽음

---

## 이상한 현상 해결하기

### 해결 방안
- y에 대한 쓰기 락을 먼저 얻고, x에 대한 읽기 락을 풀어준다면?
- tx1이 read_lock(y)을 얻으려고 해도, 획득할 수 없음

> 즉, 쓰기 목적을 위한 락을 획득하고 읽기 락을 해제!!

---

## 2PL protocol 설명

### 개념
- tx에서 모든 locking operation이 최초의 unlock operation보다 먼저 수행되도록 하는 것
- 이것은 **2PL Protocol**(two-phase locking)

### why 2-phase
- Phase 1   
    - read_lock(y)
    - write_lock(x)

    - read_lock(x)
    - write_lock(y)

- 위 두개는 lock을 취득하기만 하고 반환하지 않는 Phase(Expanding Phase(growing phase))

- Phase 2
    - unlock(y)
    - unlock(x)

    - unlock(x)
    - unlock(y)

- 위 두개는 lock을 반환만 하고 취득하지는 않는 Phase(Shrinking phase(contracting phase))
> 2PL Protocol은 Serializability를 보장함!!

---

## 2PL과 deadlock

### 상황 1
- tx2
    - read_lock(x)

- tx1
    - read_lock(y)
    - read(y) = 200
    - write_lock(x): tx2가 read_lock(x)을 획득한 상태여서 block

- tx2
    - read(x) = 100
    - write_lock(y): tx1이 read_lock(y)를 획득한 상태여서 block

> DEAD LOCK 발생!!
- OS에서 해결하는 방식과 동일함

--- 

## 2PL 종류 설명하기 위한 예제

### 2PL Protocol
- tx에서 모든 locking operatin이 최초의 unlock operation보다 먼저 수행되도록 하는 것

### 종류 예제
- x | y | z
- tx1: x와 y와 z를 더해서 y에 씀 / x에 2를 곱해서 z에 씀

- tx1
    - read_lock(x)
    - read(x)
    - write_lock(y): y에 씀을 위함
    - read(y)
    - write_lock(z): z에 씀을 위함
    - unlock(x)
    - read(z)
    - write(y = x+y+z)
    - unlock(y)
    - write(z = 2 * x)
    - unlock(z)

---

## conservative 2PL

### 개념
- 모든 lock을 획득한 뒤 트랜잭션을 시작
- 위 예제를 반영하려면
    - read_lock(x)
    - write_lock(y)
    - write_lock(z)

> Dead Lock Free / 실용적이지 않음(모든 락을 다 얻는게 힘들 수 있음)

---

## strict 2PL (S2PL)

### 개념
- strict schedule을 보장하는 2PL
- recoverability 보장
- write-lock을 commit/rollback 될 때 반환

- 위 예제에 반영하려면
    - commit
    - unlock(y)
    - unlock(z)

---

## strong strict (SS2PL)

### 개념
- strict scheudle을 보장하는 2PL
- read_lock도 commit/rollback될 때 반환

- 위 예제에 반영하려면
    - commit
    - unlock(x)
    - unlock(y)
    - unlock(z)

> S2PL보다 구현이 쉬움

> 단점: 그만큼 락을 오래 쥐고 있음

---

## lock 호환성 방식의 약점

### 문제점
- read-read를 제외하고는 한 쪽이 block이 되니깐 전체 처리량이 좋지 않음!!

### 해결 방안
- read와 write가 서로를 block하는 것이라도 해결해보자
- write와 write는 되면 이상현상이 발생이 됨
- 그 해결 방법이 => **MVCC**
