## 데이터베이스 시작하기 1

### 데이터의 집합, 데이터베이스 생성과 선택
```sql
-- 데이터베이스 생성
create database my_shop;

-- 데이터베이스 선택
use my_shop;

-- 테이블 생성
create table sample (
    product_id int primary key,
    name varchar(100), -- 최대 100글자
    price int, 
    stock_quantity int,
    release_date date -- YYYY-MM-DD
);

-- 테이블이 잘 만들어졌는지 확인
desc sample; -- 또는 describe

-- 현재 RDBMS에 존재하는 데이터베이스 목록 확인
show databases;

-- 현재 사용하고 있는 데이터베이스의 테이블 목록 확인
show tables;

-- 현재 사용하고 있는 데이터베이스의 테이블 삭제
drop table sample;

-- 현재 RDBMS에 존재하는 my_shop이라는 데이터베이스 삭제
drop database my_shop;
```

---

## 데이버베이스 시작하기 2

### 데이터베이스의 기본, CRUD 맛보기
```sql
-- 데이터 생성 (Create - Insert)
insert into sample (product_id, name, price, stock_quantity, release_date)
values (1, '프리미엄 청바지', 59900, 100, '2026-05-30');

-- 데이터 읽기 (Read - Select)
select * from sample;

-- 데이터 수정 (Update - Update)
update sample
set price = 40000
where product_id = 1;

-- 데이터 삭제 (Delete - Delete)
delete from sample
where product_id = 1;
```

### 대소문자 구문
- select, from, create 같은 SQL 키워드들은 대소문자를 구분하지 않음
- 하지만 우리가 직접 지정한 테이블 명, 컬럼 명은 DBMS 종류나 서버 환경, 또는 설정에 따라 대소문자를 구분하기도 함. (sample vs SAMPLE)

- 이러한 혼란을 피하기 위해서
    - SQL 키워드는 대문자
    - 우리가 직접 지정한 키워드는 소문자
    - 또한 소문자 + _ 조합으로 만듦: my_shop
    - **데이터 자체는 대소문자를 구분함**
        - select * from customer where name = 'kim';
        - select * from customer where name = "KIM";

--- 

## SQL이란?

### SQL(Structured Query Language)
- 관계형 데이터베이스의 표준 언어
- 여러 RDBMS마다 약간의 차이가 있긴하지만, 기본적인 표준은 거의 동일함
> 데이터베이스 방언(Dialect)
    - SQL 표준을 넘어서 각 DMBS는 자신들만의 추가 기능이나 추가 문법을 제공함

### 선언적 언어
- 데이터베이스에게 어떻게 데이터를 가져올지 지시하는 것이 아니라
- 무엇을 원하는지만 선언적으로 명시함
- ex. 서울에 사는 고객 명단을 찾아줘라고 SQL로 요청하면
- RBDMS가 내부적으로 가장 효율적인 방법을 찾아서 결과를 가져다줌 - 내부 동작방식을 알 필요가 없음
```sql
select name from customers where address = '서울';
```

### SQL 명령어의 4가지 종류
- DDL(Data Definition Language, 데이터 정의어)
    - 데이터의 구조를 정의하고 관리하는 언어
    - 데이터 그 자체가 아니라, 데이터를 담을 그릇이나 창고의 설계도를 만들고 수정하고, 제거하는 역할을 함
    - 주요 명령어
        - create
        - alter
        - drop

- DML(Data Manipulation Language, 데이터 조작어)
    - 테이블 안에 들어있는 실제 데이터를 직접 조작(추가, 조회, 수정, 삭제)하는 언어
    - 주요 명령어
        - insert: 테이블에 새로운 데이터를 추가
        - select: 테이블에서 데이터를 조회
        - update: 기존 데이터를 수정
        - delete: 기존 데이터를 삭제

- DCL(Data Control Language, 데이터 제어어)
    - 데이터에 대한 접근 권한을 부여(GRANT)하거나 회수(REVOKE)하는 등, 데이터의 보안과 관련된 권한을 제어함
    - 주요 명령어
        - GRANT: 특정 사용자에게 특정 작업에 대한 수행 권한을 부여함
        - REVOKE: 특정 사용자에게서 이미 부여한 권한을 회수함

- TCL(Transaction Control Language, 트랜잭션 제어어)
    - DML에 의해 수행된 데이터 변경 작업들이 하나의 거래 단위로 묶어서 관리하는 언어
    - 작업의 일관성을 보장하여 데이터가 잘못되는 것을 방지함
    - 주요 명령어
        - commit: 트랜잭션의 모든 작업을 최종적으로 데이터베이스에 확정, 저장함
        - rollback: 트랜잭션의 모든 작업을 취소하고 이전 상태로 되돌림

---

## 데이터 타입

- 저장 공간의 효율적을 관리 및 데이터의 정확성을 높일 수 있음

### 숫자 타입
- 숫자 타입이 왜 많을까?
- 나이를 저장하는 열과 국가 총인구 저장하는 열에 똑같은 크기의 데이터 타입을 쓰는 것은 낭비
- 사람의 나이의 경우, 아무리 많아도 현실적으로 0~150을 넘기기 힘듦
- 외우는 것이 아니라, 이런게 있다만 알고 사용할 때 찾아보고 적용!!

### 문자열 타입
- VARCHAR(n): 최대 n글자까지 저장되는 가변 길이 문자열
    - 이름, 주소처럼 길이가 제각각인 데이터에 사용하면 저장 공간을 효율적으로 사용할 수 있음
    - 홍길동(3글자)는 VARCHAR(10)에 저장하면 실제 데이터 길이인 3글자만큼의 공간만 사용함
    - 김네이트(4글자)을 VARCHAR(10)에 저장하면 실제 데이터 길이인 4글자만큼의 공간만 사용함

- CHAR(n): 항상 n글자 길이를 차지하는 고정 길이 문자열
    - 남(1글자)을 CHAR(2)에 저장하면, 나머지 1글자는 공백으로 채워져 무조건 2글자의 공간을 차지함
    - **항상 길이가 정해져있는 데이터에 사용하면, VARCHAR보다 아주 약간의 이점을 가질 수 있음**
    - 길이가 항상 같으니, 데이터를 찾기 위해 길이를 확인할 필요가 없음

- TEXT: 매우 긴 텍스트를 저장할 때 사용함
    - 상품의 상세 설명, 장문의 리뷰, 블로그 게시글 본문처럼 긴 글 작성에 적합

- VARCHAR vs CHAR
    - 기본적으로 varchar를 사용함
    - varchar는 데이터 길이를 확인하기 위해 길이 저장 공간을 차지함
    - VARCHAR: **데이터의 길이가 자주 변경되면, 행의 크기가 변해 데이터 단편화나 페이지 분할이 발생하여 성능 저하의 원인이 될 수 있음**
    - CHAR: **데이터 길이가 고정되어 있어 업데이트 시에도 행의 크기가 변하지 않으므로, 데이터 수정이 빈번할 때 VARCHAR보다 유리할 수 있음**
    - **애매하면 VARCHAR를 우선적으로 고려**

### 날짜와 시간 타입
- DATE: 날짜 정보(YYYY-MM-DD)만 저장됨
- DATETIME: 날짜와 시간 정보(YYYY-MM-DD HH:MM:SS)를 함께 저장함, 입력된 값을 그대로 저장함
- TIMESTAMP: DATETIME과 유사하게 날짜와 시간을 저장하지만, 특별한 기능이 있음
    - 타임존 자동 변환: TIMESTAMP는 데이터를 저장할 때 현재 서버의 타임존을 기준으로 UTC로 변환하여 저장하고, 데이터를 조회할 때는 현재 서버의 타임존에 맞춰 다시 변환해서 보여줌

- **DATETIME을 사용하자**
    - 저장할 때 UTC로 변환후 가져올 때 타임존에 맞춰서 다시 변환해서 가져오도록 하면 됨

### 기본 타입
- BLOB: 이미지, 오디오, 비디오 같은 이진 대용량 데이터를 저장함
- ENUM: 단일 선택 타입: 미리 정의한 값 목록 중 하나만 선택하여 저장할 수 있음
- SET: 다중 선택 타입, 미리 정의한 값 목록 중 여러 개를 동시에 선택하여 저장할 수 있음
- 실무에서는 ENUM / SET은 잘 사용하지 않음

--- 

## 제약 조건
- 일단 타입으로 먼저 거름

### NOT NULL: 필수 입력 항목 지정

### UNIQUE: 중복 불가 항목 지정

### PRIMARY KEY: 테이블의 대표 식별자
- 테이블의 모든 행을 유일하게 식별하는 열(NOT NULL과 UNIQUE의 특징을 모두 가짐)
- 모든 테이블에는 반드시 PRIMARY KEY가 있어야함
- AUTO INCREMENT: MySQL에서 PRIMARY KEY에 자주 사용하는 옵션
    - 정수 타입의 PRIMARY KEY 열에 이 옵션을 설정하면 새로운 데이터가 추가될 때마다 1씩 자동으로 증가하는 번호를 할당해줌

### FOREIGN KEY (FK): 테이블 간의 관게 설정
- 예를 들어 주문 테이블에 customer_id(FK) 값이 99가 입력된다고 가정
- customers 테이블에는 99번 고객 번호가 없음
- 이런 경우 데이터베이스는 데이터의 입력을 막음 (오류 발생)

### DEFAULT: 기본값 설정
- 데이터를 insert 시 특정 열의 값을 명시하지 않으면 기본 값이 들어감

### CHECK: 컬럼에 입력되는 값이 특정 조건을 만족하는지 검사
- 조건에 맞지 않는 데이터의 입력을 막음
    - 10보다 작은 숫자가 입력되지 않도록 막음