## DDL - 테이블 생성 1

```sql
-- 고객 테이블
create table customers(
    customer_id int auto_increment primary key,
    name varchar(50) not null,
    email varchar(100) not null unique,
    password varchar(255) not null,
    address varchar(255) not null,
    join_date datetime default current_timestamp -- 자동으로 현재 시간이 들어감
);

-- 상품 테이블
create table products(
    product_id int auto_increment primary key,
    name varchar(100) not null,
    description text not null,
    price int not null,
    stock_quantity int not null default 0
);

-- 주문 테이블
create table orders(
    order_id int auto_increment primary key,
    customer_id int not null,
    product_id int not null,
    quantity int not null,
    order_date datetime default current_timestamp,
    status varchar(20) not null default '주문접수',

    constraint fk_orders_customers foreign key (customer_id) references customers(customer_id),
    constraint fk_orders_products foreign key (product_id) references products(product_id)
);

-- 테이블 재생성 시 먼저 실행
drop table orders; -- 연관관계를 가지고 있는 경우 먼저 삭제해야 됨
drop table products;
drop table customers;

``

### 고객 테이블
```sql
-- 고객 테이블
create table customers(
    customer_id int auto_increment primary key,
    name varchar(50) not null,
    email varchar(100) not null unique,
    password varchar(255) not null,
    address varchar(255) not null,
    join_date datetime default current_timestamp -- 자동으로 현재 시간이 들어감
);
```

### 날짜와 기본값 설정 옵션
```sql
create table test(
    ...
    created_at datetime default current_timestamp;
    updated_at datetime default current_timestamp on update current_timestamp
)
```
- default current_timestamp on update current_timestamp
    - 새로운 데이터 행(row)이 추가될 때는 물론이고, 같은 행의 컬럼 값이 변경되어 업데이트 될 때, 이 컬럼의 값은 현재 날짜와 시간이 자동으로 갱신됨

### 상품 테이블
```sql
-- 주문 테이블
create table orders(
    order_id int auto_increment primary key,
    customer_id int not null,
    product_id int not null,
    quantity int not null,
    order_date datetime default current_timestamp,
    status varchar(20) not null default '주문접수',

    constraint fk_orders_customers foreign key (customer_id) references customers(customer_id),
    constraint fk_orders_products foreign key (product_id) references products(product_id)
);
```
- 왜래키 제약조건을 걸었음 (constraint [제약조건이름])

### 외래키 제약조건
```sql
constraint [제약조건_이름]
foreign key ([자식_테이블의_컬럼명])
references [부모_테이블명]([부모_테이블의_컬럼명])
[on delete 옵션] [on update 옵션]
```

---

## DDL - 테이블 생성 2

### MySQL 워크벤티로 ERD 만들기
- Database => Reverse Engineer

### 테이블과 컬럼 이름 규칙
- 기본 규칙 (백틱 미사용(`))
    - 허용 문자
        - 영문 대소문자
        - 숫자
        - 밑줄
        - 달러 기호
        - UTF-8과 같은 다국어 문자

    - 제한 사항
        - 이름이 숫자로 시작할 수 없음
        - MySQL 예약어(SELECT, TABLE, ORDER)는 사용할 수 없음
        - 총 길이는 64자를 넘을 수 없음

- 확장 규칙 (백틱 사용)
    - 허용 문자: 거의 모든 문자가 허용됨
        - 공백: `user name`
        - 하이폰(-) 및 기타 특수문자: (`item-code`)
        - 숫자로 시작하는 이름: `2025_report`
        - MySQL 예약어: `order`

### 권장되는 명명 규칙(Best Practice)
- 영문 소문자와 밑줄 사용(스네이크 케이스)
- 일관성 유지
- 예약어 피하기
- 간결하고 명확하게
- 다국어 이름 피하기: 한글도 가능하지만 호환성 및 인코딩 문제를 예방하기 위해 영문으로 작성하는 것을 권장함

---

## DDL - 테이블 변경, 제거

### AlTER TABLE: 이미 만든 테이블의 구조 변경
- 열 추가하기 - add column
```sql
-- 새로운 컬럼 생성
alter table customers
add column point int not null default 0;

-- 컬럼 변경
alter table customers
modify column address varchar(500) not null;

-- 컬럼 삭제
alter table customers
drop column point;

select * from customers;
```
- 실무 경고
    - alter table은 유용한 기능이지만, 조심해서 사용해야함
    - 거대한 테이블의 구조를 변경하는 작업은 엄청나게 많은 시간과 시스템 자원을 소모함
    - 작업 중에는 테이블이 잠겨서 서비스가 일시적으로 멈출 수 있음
    - 참고로 최신 버전의 MySQL에서는 열 추가하는 것 정도는 실시간으로 적용해도 크게 문제가 되지 않음
    - 라이브 변경 작업은 데이터베이스 버전에 맞는 메뉴얼을 확인하고 어느 정도 잠금이 발생하는지 확인하는 것이 중요

### Drop table vs Truncate table
- 두 개는 비슷해보이지만 아주 큰 차이가 있음
- drop table: 테이블의 존재 자체를 삭제함
- truncate table: 테이블의 구조는 남기고, 내부 데이터만 모두 삭제함
    - 특징
        - delete from orderes: where 절이 없는 delete와 결과적으로 같아보이지만,
        - truncate가 훨씬 빠름
        - delete는 한 줄 씩 삭제 기록을 남기는 반면, truncate는 테이블을 초기화하는 개념이라 내부 처리 방식이 더 간단하고 빠름
        - **truncate는 auto_increment 값도 초기화함**

### 제약 조건 무시하기
- products 테이블은 orders에서 참조하고 있음
- **이렇게 참조 당하는 테이블은 drop, truncate를 할 수 없음**

### 제약 조건을 무시하고 싶다면?
```sql
set foreign_key_checks = 0; -- 비활성화
set foreign_key_checks = 1; -- 활성화
```
- 필요한 작업을 마친 후에는 반드시 다시 활성화
- set xxx 방식의 설정은 데이터베이스 접속이 연결되는 동안만 유효함
    - 연결이 끊기면 설정이 사라짐
    - 따라서 다시 접속하는 경우 설정을 다시 해줘야함

---

## DML - 등록

### INSERT 문법
```sql
insert into table_name (column1, column2, ...)
values (value1, values2, ...);
```

```sql
-- 전체 열에 넣기
insert into customers values (null, '강감찬', 'kang@example.com', 'hased_password_123', '서울시 관악구', '2026-05-30 10:30:00');
insert into customers values (null, '이순신', 'lee@example.com', 'hased_password_123', '서울시 관악구', '2026-05-30 10:30:00');

-- 원하는 열에만 넣기
insert into customers (name, email, password, address)
values ('세종대왕', 'sejoing@example.com', 'hashed_password_456', '서울시');

select * from customers;

insert into products (name, price, stock_quantity)
values ('초록색 긴팔 티셔츠', 30000, 50);

select * from products;
```
- id 쪽에 null을 넣으면 auto_increment에 의해서 자동으로 갱신됨

### 실무 이야기
- 열 목록을 사용하자 (테이블 구조가 변경될 가능성을 열어두기)
- 항상 데이터를 추가할 열의 목록을 명시적으로 작성하는 것이 안전하고 좋은 습관

### 한 번에 여러 행에 데이터 추가하기
```sql
insert into products (name, price, stock_quantity) values
('검정 양말', 5000, 100),
('하얀 양말', 5000, 100);
```

---

## DML - 수정 (Update)

### UPDATE 문법
```sql
update table_name
set column1 = value1, column2 = value2, ...
where condition;
```
- where절을 주의해야함
    - where절이 없으면 모든 행이 영향을 받음
    - 또한 조건을 명시한다고 해도 주의해서 해야함

- 다행히도 MySQL은 안전 업데이트 모드를 제공함
    - where 절에 기본 키 컬럼을 반드시 지정해야함
```sql
select @@SQL_SAFE_UPDATES; -- 0: false 1: true
```
- 문제: 기본 키 컬럼을 사용하지 않은 다음과 같은 SQL도 실행이되지 않음
```sql
update products
set price = 900
where name = '베이직 반팔 티셔츠';
```

### 참고사항
- MySQL 서버는 기본적으로 제공하는 안전 업데이트 모드의 기본 설정은 비활성화 상태
- 다만, MySQL Workbench가 안전 업데이트 설정을 자동으로 활성화함
```sql
SET SQL_SAFE_UPDATES = 0; -- 안전 업데이트 모드 끄기
SET SQL_SAFE_UPDATES = 1; -- 안전 업데이트 모드 활성화
```

### 주의사항
- **변경 대상을 먼저 확인하는 습관을 반드시 들이자!!!**
```sql
-- 1. 먼저 변경 대상을 확인
select * from proudcts
where name = '베이직 반팔 티셔츠';

-- 2. 안전 모드 끄기
SET SQL_SAFE_UPDATES = 0;

-- 3. 데이터 수정
update products
set price = 19800
where name = '베이직 반팔 티셔츠'; -- 이 때 위에 select에서 사용했던 where를 복사해서 사용

-- 4. 안전 모드 활성화
SET SQL_SAFE_UPDATES = 1;
```

---

## DML - 삭제 (DELETE)

### DELETE 기본 문법
```sql
delete from table_name
where condition;
```
- delete는 더 중요한데, 왠만하면 조건을 반드시 붙여야함
- 모든 행이 사라지는 대재앙을 맛 볼 수 있음
- 안전 업데이트 모드도 동일하게 적용됨

### 전체 UPDATE / DELETE를 실행했을 때 대처법
1. 사고 발생: 사실을 인식한다. 심장이 내려앉는다.
2. 즉시 보고 및 서비스 점검: 즉시, DBA를 호출하고, 상급자에게 상황을 보고한다. 가능하다면 추가적인 피해를 막기 위해 서비스를 점검 상태로 전환. 전문적인 지식을 가진 DBA가 있다면 데이터베이스 복원 기능으로 빠르게 복구할 수도 있음
3. 백업 확인: 가장 먼저 확인해야할 것은 가장 최근의 데이터베이스 백업
4. 로그 분석: 백업이 없다면, 데이터베이스 로그를 분석하여 이전의 값을 추적하는 복잡한 작업을 시도해 볼 수 있으나, 이는 매우 어렵고 시간이 오래 걸림

### 다시 한 번 비교하기: DELETE vs TRUNCATE
- DELETE 
    - DML: 데이터 조작어
    - 한 줄 씩, 조건에 따라 삭제 가능
    - 초기화되지 않음
    - 트랜잭션 내에서 ROLLBACK 가능

- TRUNCATE
    - DDL: 데이터 정의어
    - 테이블 전체를 한 번에 삭제
    - 매우 빠름
    - 1부터 다시 시작하도록 AUTO_INCREMENT 초기화
    - ROLLBACK 불가능 (즉시 적용)

---

## 제약 조건 활용
```sql
set foreign_key_checks = 0; -- 외래키 제약 조건 비활성화
truncate table products;
truncate table customers;
truncate table orders;
set foreign_key_checks = 1;
```

### NOT NULL 
- 반드시 데이터에 null이 아닌 값이 들어있어야함
- insert를 거부함

### UNIQUE 제약 조건: 이미 사용 중인 이메일입니다.
- 컬럼의 내용이 중복되면 안될 때 사용함
- insert를 거부함

### 외래 키 제약 조건: 관계의 무결성 지키기
- 주문의 경우 "고객과 주문한 상품이 반드시 존재해야 함"!!
- 참고로, insert, update, delete 등 모든 상황의 데이터 불일치를 원천 방지함!!

---

## 문제와 풀이

### 문제 1: 데이터베이스와 테이블 생성하기
```sql
create database my_test;

use my_test;

create table members(
    id int not null primary key,
    name varchar(50) not null,
    join_date date
);

desc members;
```

### 문제 2: 데이터 추가 및 조회하기
```sql
insert into members(name, join_date) values
('션', '2025-01-10'),
('네이트', '2025-02-15');
```

### 문제 3: 데이터 수정 및 삭제하기
```sql
SET SQL_SAFE_UPDATES = 0;
update members set name = '네이트2' where id = 2;
delete from members where id = 1;
SET SQL_SAFE_UPDATES = 1;
```

### 문제 4: 제약 조건을 포함한 테이블 생성하기
```sql
create table products(
    product_id int primary key auto_increment
    product_name varchar(100) not null,
    product_code varchar(20) unique,
    price int not null,
    stock_count int not null default 0
);
```

### 문제 5: 외래 키로 테이블 관계 맺기
```sql
create table customers(
    customer_id int primary key auto_increment,
    name varchar(50) not null
);

create table orders(
    order_id int primary key auto_increment,
    customer_id int not null,
    order_date datetime default current_timestamp,
    
    constraint fk_orders_customers foreign key (customer_id) references customers(customer_id)
);

insert into customers(name) values('홍길동');
insert into orders(customer_id) values(1);
```
