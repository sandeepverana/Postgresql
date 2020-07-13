# Postgresql concepts 

**1. [ Constraint types ](#constraint-types)  
  2. [Logical Operators](#logical-operators)        
  3. [Joins](#joins)                
  4. [Aggregate Functions - Group By - Having](#aggreagate-functions---group-by---having)                    
  5. [Subqueries](#subqueries)            
  6.[ Case + Coalesce ](#case---coalesce)  
  7. [Distinct - Distinct ON ](#distinct-and-distinct-on)    
  8. [ COPY to load datasets ](#copy)  
  9. [Temporary tables](#temporary-tables)   
  10.  [ Views, Materialized views  ](#views)           
  11.  [Common Table Expressions's (CTE)](#common-table-expressions-ctes)    
  12.  [Window functions](#window-functions)    
  13.  [ Full text search ](#full-text-search)          
  14.  [ Transaction isolation levels  ](#transaction-isolation-levels)         
  15. [ Manual locking ](#locking)  
  16. [Indexes - partial/functional, explain, analyze ](#indexes)  
  17. [Triggers](#triggers)**


## Constraint types 
***
Constraints are a part of the table definition that limits the values inserted into its columns.
                  
**CHECK constraint**

A check constraint allows you to specify that the value in a certain column must satisfy a Boolean (truth-value) expression.

```sql
create table orders(order_id int,price numeric check(price>0),description text);
CREATE TABLE
```

Giving a name to constraint

``` sql 
create table orders(order_id int,price numeric constraint order_check_constraint check(price>0),description text);
CREATE TABLE 
```


 A check constraint can also refer to several columns.

```sql
create table orders(order_id int,price numeric,discounted_price numeric,description text,
check(price>0),check(discounted_price>0),check(price>discounted_price));
CREATE TABLE


\d orders        
         Table "public.orders"
      Column      |  Type   | Modifiers 
------------------+---------+-----------
 order_id         | integer | 
 price            | numeric | 
 discounted_price | numeric | 
 description      | text    | 
Check constraints:
    "orders_check" CHECK (price > discounted_price)
    "orders_discounted_price_check" CHECK (discounted_price > 0::numeric)
    "orders_price_check" CHECK (price > 0::numeric) 
```

**NOT NULL constraint**   

Column constraint which ensures that column in the table cannot take any null value

No name is required to define a null constraint..

```sql
create table orders(order_id int not null,price numeric not null check(price>0),description text not null);
CREATE TABLE

\d orders 
       Table "public.orders"
   Column    |  Type   | Modifiers 
-------------+---------+-----------
 order_id    | integer | not null
 price       | numeric | not null
 description | text    | not null
Check constraints:
    "orders_price_check" CHECK (price > 0::numeric)

	
insert into orders(order_id,price,description) values(null,10.50,'A test order');
ERROR:  null value in column "order_id" violates not-null constraint
DETAIL:  Failing row contains (null, 10.50, A test order). 
```


**UNIQUE constraint**

Unique constraints ensure that the data contained in a column, or a group of columns, is unique among all the rows in the table.


```sql
create table orders(order_id int not null UNIQUE,price numeric not null check(price>0),description text not null);
CREATE TABLE


insert into orders values(100,500.505,'A test description');
INSERT 0 1

insert into orders values(100,600,'A test description');
ERROR:  duplicate key value violates unique constraint "orders_order_id_key"
DETAIL:  Key (order_id)=(100) already exists.
```

 Even in the presence of a unique constraint it is possible to store duplicate rows that contain a null value in at least one of the constrained columns.

```sql 
\d orders
       Table "public.orders"
   Column    |  Type   | Modifiers 
-------------+---------+-----------
 order_id    | integer | 
 price       | numeric | 
 description | text    | 
Indexes:
    "orders_order_id_key" UNIQUE CONSTRAINT, btree (order_id)
Check constraints:
    "orders_price_check" CHECK (price > 0::numeric)

select * from orders;
 order_id |  price  |    description     
----------+---------+--------------------
      100 | 500.505 | A test description
          |     300 | 
          |     400 | 
(3 rows)
```

**Primary Keys**

A primary key constraint indicates that a column, or group of columns, can be used as a unique identifier for rows in the table. 

Adding a primary key will automatically create constraints **NOT NULL** and **UNIQUE**.

```sql
create table prod(prod_no int primary key,price int);
CREATE TABLE

\d prod;
      Table "public.prod"
 Column  |  Type   | Modifiers 
---------+---------+-----------
 prod_no | integer | not null
 price   | integer | 
Indexes:
    "prod_pkey" PRIMARY KEY, btree (prod_no)


alter table prod alter prod_no drop not null;
ERROR:  column "prod_no" is in a primary key
```

A table can have at most one primary key. (There can be any number of unique and not-null constraints, 
which are functionally almost the same thing, but only one can be identified as the primary key.) 

Relational database theory dictates that every table must have a primary key. 

This rule is not enforced by PostgreSQL, but it is usually best to follow it.



**Foreign Keys**

A foreign key constraint specifies that the values in a column (or a group of columns) 
must match the values appearing in some row of another table.  
We say this maintains the referential integrity between two related tables.

A table can contain multiple foreign keys which is used to implement many to many relationships between tables. 

```sql
create table prod(prod_no int primary key,name text,price numeric);
CREATE TABLE

create table orders(order_id integer primary key,prod_no int references prod(prod_no),quantity int);
CREATE TABLE

create table order_sample(order_id int references prod);
CREATE TABLE

```
In absence of a column list the primary key of the referenced table is used as the referenced column(s).


**ON DELETE CASCADE**


CASCADE specifies that when a referenced row is deleted, row(s) referencing it should be automatically deleted as well. 

```sql
\d prod
      Table "public.prod"
 Column  |  Type   | Modifiers 
---------+---------+-----------
 prod_no | integer | not null
 name    | text    | 
 price   | numeric | 
Indexes:
    "prod_pkey" PRIMARY KEY, btree (prod_no)
Referenced by:
    TABLE "order_items" CONSTRAINT "order_items_prod_no_fkey" FOREIGN KEY (prod_no) REFERENCES prod(prod_no)


\d orders
         Table "public.orders"
      Column      |  Type   | Modifiers 
------------------+---------+-----------
 order_id         | integer | not null
 shipping_address | text    | 
Indexes:
    "orders_pkey" PRIMARY KEY, btree (order_id)
Referenced by:
    TABLE "order_items" CONSTRAINT "order_items_order_id_fkey" FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE


\d order_items
   Table "public.order_items"
  Column  |  Type   | Modifiers 
----------+---------+-----------
 prod_no  | integer | not null
 order_id | integer | not null
 quantity | integer | 
Indexes:
    "order_items_pkey" PRIMARY KEY, btree (prod_no, order_id)
Foreign-key constraints:
    "order_items_order_id_fkey" FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
    "order_items_prod_no_fkey" FOREIGN KEY (prod_no) REFERENCES prod(prod_no)


select * from order_items;
 prod_no | order_id | quantity 
---------+----------+----------
      10 |      100 |        2
      11 |      100 |        3
      12 |      101 |        2
(3 rows)

select * from orders;
 order_id | shipping_address 
----------+------------------
      100 | Hyd
      101 | Pune
      103 | Chennai
      102 | Banglore
(4 rows)

select * from order_items;
 prod_no | order_id | quantity 
---------+----------+----------
      10 |      100 |        2
      11 |      100 |        3
      12 |      101 |        2
      10 |      102 |        4
      11 |      103 |        2
(5 rows)


delete from orders where order_id=102;
DELETE 1

select * from orders;
 order_id | shipping_address 
----------+------------------
      100 | Hyd
      101 | Pune
      103 | Chennai
(3 rows)

select * from order_items;
 prod_no | order_id | quantity 
---------+----------+----------
      10 |      100 |        2
      11 |      100 |        3
      12 |      101 |        2
      11 |      103 |        2
(4 rows)
```

Analogous to ON DELETE there is also ON UPDATE which is invoked when a referenced column is changed (updated)

In this case, CASCADE means that the updated values of the referenced column(s) should be copied into the referencing row(s).

## Logical Operators
***

Types of operators:  
+ UNION
+ UNION ALL
+ INTERSECT
+ AND 
+ OR

```sql
select * from basket_a; 

id |  fruit   
----+----------
  1 | Apple
  2 | Orange
  3 | Banana
  4 | Cucumber

select * from basket_b;

 id |   fruit    
----+------------
  1 | Orange
  2 | Apple
  3 | Watermelon
  ```

**UNION**       
Union operator fetches all the unique rows from the two select queries.   
Combines the results of two or more result statements.
```sql
select fruit from basket_a union select fruit from basket_b;
   fruit    
------------
 Cucumber
 Watermelon
 Apple
 Pear
 Banana
 Orange
```
**UNION ALL**       
Fetches all the rows from two or more result statements which includes duplicates.   
In UNION operator it won't consider the duplicate values (only unique values) but UNION ALL fetches all the values including duplicates.

```sql
select fruit from basket_a union all select fruit from basket_b;

   fruit    
------------
 Apple
 Orange
 Banana
 Cucumber
 Orange
 Apple
 Watermelon
 Pear

```
**INTERSECT**       
Matches the two tables which only displays the matched row value with the other query.
```sql
select fruit from basket_a intersect select fruit from basket_b;
 fruit  
--------
 Apple
 Orange
(2 rows)
```

**EXCEPT**      
Fetches all the rows except the ones matched or present on the other side.
```sql
select fruit from basket_a except select fruit from basket_b;
  fruit   
----------
 Cucumber
 Banana
(2 rows)
```

**AND**     
Select all the rows which matches both the conditions (both conditions have to be true)
```sql
select * from emp where emp.sal>=8000 and emp.sal<20000;

 empno | empname | hourpay |   sal    | commission | numsales | dno | dep_id 
-------+---------+---------+----------+------------+----------+-----+--------
   766 | smit    |         |  8000.00 |            |    20.00 |   2 |     40
   101 | Neena   |         | 10000.00 |            |          |   2 |    110
   102 | lex     |   30.00 | 12000.00 |            |          |   1 |    100
   732 | kane    |   20.00 | 10000.00 |    2000.00 |    12.00 |   1 |     40

```

**OR**          
Selects all the rows which matches the either of the conditions when or is used it checks if any one of the condition is met.
```sql
select * from emp where emp.sal>=8000 or  emp.commission<=3000;

 empno | empname | hourpay |   sal    | commission | numsales | dno | dep_id 
-------+---------+---------+----------+------------+----------+-----+--------
   766 | smit    |         |  8000.00 |            |    20.00 |   2 |     40
   784 | alen    |         |  1600.00 |    3000.00 |     4.00 |   3 |     40
   100 | Steve   |   30.00 | 24000.00 |            |          |   1 |     90
   101 | Neena   |         | 10000.00 |            |          |   2 |    110
   102 | lex     |   30.00 | 12000.00 |            |          |   1 |    100
   104 | bruce   |         | 28000.00 |            |          |   2 |    120

```
**AND with OR**         
Using AND, OR operator 
```sql
select * from emp where (emp.sal>=8000 and emp.sal<20000) or (emp.sal>1000 and emp.sal<2500);

 empno | empname | hourpay |   sal    | commission | numsales | dno | dep_id 
-------+---------+---------+----------+------------+----------+-----+--------
   766 | smit    |         |  8000.00 |            |    20.00 |   2 |     40
   784 | alen    |         |  1600.00 |    3000.00 |     4.00 |   3 |     40
   791 | ward    |   40.00 |  1250.00 |    5000.00 |    10.00 |   3 |     40
   101 | Neena   |         | 10000.00 |            |          |   2 |    110
   102 | lex     |   30.00 | 12000.00 |            |          |   1 |    100
   732 | kane    |   20.00 | 10000.00 |    2000.00 |    12.00 |   1 |     40

```
**LIKE**        
Like operator is allowed to match with string of any length or charcters.  
SQL query to find empnames starting with s (empname words)
```sql 

select * from emp where empname like '%s%';

 empno | empname | hourpay |   sal   | commission | numsales | dno | dep_id 
-------+---------+---------+---------+------------+----------+-----+--------
   766 | smit    |         | 8000.00 |            |    20.00 |   2 |     4

---word with second letter as a
select * from emp where empname like '_a%';

 empno | empname | hourpay |   sal    | commission | numsales | dno | dep_id 
-------+---------+---------+----------+------------+----------+-----+--------
   791 | ward    |   40.00 |  1250.00 |    5000.00 |    10.00 |   3 |     40
   203 | mark    |   23.00 | 31000.00 |    4000.00 |    21.00 |   3 |     90
   732 | kane    |   20.00 | 10000.00 |    2000.00 |    12.00 |   1 |     40

---words ending with a..
select * from emp where empname like '%a';

 empno | empname | hourpay |   sal    | commission | numsales | dno | dep_id 
-------+---------+---------+----------+------------+----------+-----+--------
   101 | Neena   |         | 10000.00 |            |          |   2 |    110
```

## Joins
***
Joins are used to retrieve data (combine columns from one or more tables based on the values of the common columns between tables)

```sql
select * from category;
 cat_id | cat_name 
--------+----------
      1 | COMPUTER
      2 | MOBILE
      3 | FASHION
(3 rows)

select * from products;
 prod_id |     prod_name     |  price   | cid 
---------+-------------------+----------+-----
    1001 | ACER ASPIRE       | 33000.00 |   1
    1002 | DELL INSPIRON     | 43000.00 |   1
    1003 | LENOVE IDEAPAD    | 18000.00 |   1
    1004 | SAMSUNG GALAXY S9 | 35000.00 |   2
    1005 | IPHONE 7          | 60000.00 |   2
    1006 | HONOR 6X          |  9000.00 |   2
    1007 | NOKIA 1100        |  4000.00 |    
(7 rows)
```	

Types of Joins
* INNER JOIN
* LEFT JOIN or LEFT OUTER JOIN
* RIGHT JOIN or RIGHT OUTER JOIN
* FULL JOIN or FULL OUTER JOIN
* CROSS JOIN

![Joins](https://www.dofactory.com/Images/sql-joins.png)  
**INNER JOIN**

Returns all rows from multiple tables where the join condition is met.

```sql
select p.prod_name,p.price,c.cat_name from products p inner join category c on p.cid=c.cat_id;

     prod_name     |  price   | cat_name 
-------------------+----------+----------
 ACER ASPIRE       | 33000.00 | COMPUTER
 DELL INSPIRON     | 43000.00 | COMPUTER
 LENOVE IDEAPAD    | 18000.00 | COMPUTER
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE
 IPHONE 7          | 60000.00 | MOBILE
 HONOR 6X          |  9000.00 | MOBILE
(6 rows)
```
**LEFT OUTER JOIN or LEFT JOIN** 

Returns all rows from the LEFT-hand table specified in the ON condition
 and only those rows from the other table where the joined fields are equal (join condition is met).
 
 ```sql
select p.prod_name,p.price,c.cat_name from products p left join category c on p.cid=c.cat_id;

     prod_name     |  price   | cat_name 
-------------------+----------+----------
 ACER ASPIRE       | 33000.00 | COMPUTER
 DELL INSPIRON     | 43000.00 | COMPUTER
 LENOVE IDEAPAD    | 18000.00 | COMPUTER
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE
 IPHONE 7          | 60000.00 | MOBILE
 HONOR 6X          |  9000.00 | MOBILE
 NOKIA 1100        |  4000.00 | 
(7 rows)
```

**RIGHT OUTER JOIN or RIGHT JOIN**

Returns all rows from the RIGHT-hand table specified in the ON condition
and only those rows from the other table where the joined fields are equal (join condition is met).

```sql
select p.prod_name,p.price,c.cat_name from products p right join category c on p.cid=c.cat_id;
     prod_name     |  price   | cat_name 
-------------------+----------+----------
 ACER ASPIRE       | 33000.00 | COMPUTER
 DELL INSPIRON     | 43000.00 | COMPUTER
 LENOVE IDEAPAD    | 18000.00 | COMPUTER
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE
 IPHONE 7          | 60000.00 | MOBILE
 HONOR 6X          |  9000.00 | MOBILE
                   |          | FASHION
```


**FULL OUTER JOIN or FULL JOIN**  
Returns all rows from the LEFT-hand table and RIGHT-hand table with nulls in place where the join condition is not met.     
i.e Combines the result set of both RIGHT JOIN and LEFT JOIN

```sql
select p.prod_name,p.price,c.cat_name from products p full  join category c on p.cid=c.cat_id;

     prod_name     |  price   | cat_name 
-------------------+----------+----------
 ACER ASPIRE       | 33000.00 | COMPUTER
 DELL INSPIRON     | 43000.00 | COMPUTER
 LENOVE IDEAPAD    | 18000.00 | COMPUTER
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE
 IPHONE 7          | 60000.00 | MOBILE
 HONOR 6X          |  9000.00 | MOBILE
 NOKIA 1100        |  4000.00 | 
                   |          | FASHION
(8 rows)
```
**CROSS JOIN**

The CROSS JOIN creates a cartesian product between two sets of data.

```sql
select p.prod_name,p.price,c.cat_name from products p,category c;    

	prod_name     |  price   | cat_name 
-------------------+----------+----------
 ACER ASPIRE       | 33000.00 | COMPUTER
 DELL INSPIRON     | 43000.00 | COMPUTER
 LENOVE IDEAPAD    | 18000.00 | COMPUTER
 SAMSUNG GALAXY S9 | 35000.00 | COMPUTER
 IPHONE 7          | 60000.00 | COMPUTER
 HONOR 6X          |  9000.00 | COMPUTER
 NOKIA 1100        |  4000.00 | COMPUTER
 ACER ASPIRE       | 33000.00 | MOBILE
 DELL INSPIRON     | 43000.00 | MOBILE
 LENOVE IDEAPAD    | 18000.00 | MOBILE
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE
 IPHONE 7          | 60000.00 | MOBILE
 HONOR 6X          |  9000.00 | MOBILE
 NOKIA 1100        |  4000.00 | MOBILE
 ACER ASPIRE       | 33000.00 | FASHION
 DELL INSPIRON     | 43000.00 | FASHION
 LENOVE IDEAPAD    | 18000.00 | FASHION
 SAMSUNG GALAXY S9 | 35000.00 | FASHION
 IPHONE 7          | 60000.00 | FASHION
 HONOR 6X          |  9000.00 | FASHION
 NOKIA 1100        |  4000.00 | FASHION
(21 rows)
```

Display the product details and category name of all products
```sql
select p.prod_id,p.prod_name,p.price,c.cat_name from products p INNER JOIN category c on p.cid=c.cat_id;
 prod_id |     prod_name     |  price   | cat_name 
---------+-------------------+----------+----------
    1001 | ACER ASPIRE       | 33000.00 | COMPUTER
    1002 | DELL INSPIRON     | 43000.00 | COMPUTER
    1003 | LENOVE IDEAPAD    | 18000.00 | COMPUTER
    1004 | SAMSUNG GALAXY S9 | 35000.00 | MOBILE
    1005 | IPHONE 7          | 60000.00 | MOBILE
    1006 | HONOR 6X          |  9000.00 | MOBILE
(6 rows)

```

Display the products for the given category name
```sql
select p.*,c.cat_name from products p right join category c on p.cid=c.cat_id where c.cat_name='FASHION';
 prod_id | prod_name | price | cid | cat_name 
---------+-----------+-------+-----+----------
         |           |       |     | FASHION
(1 row)
```


Display the product  with given category name for the given price range
```sql
select p.prod_id,p.prod_name,p.price,c.cat_name from products p right join category c on p.cid=c.cat_id where c.cat_name='COMPUTER' and p.price between 20000 and 40000;
 prod_id |  prod_name  |  price   | cat_name 
---------+-------------+----------+----------
    1001 | ACER ASPIRE | 33000.00 | COMPUTER
(1 row)
```
Display the category that donot have any products
```sql
select c.cat_name from category c LEFT JOIN products p on  c.cat_id=p.cid where p.prod_id is NULL;
 cat_name 
----------
 FASHION
(1 row)

```
## Aggregate Functions - Group By - Having
***
An aggregate function is a special kind of function that operates on several rows of a query at once,
returning a single result.

All aggregate functions by default exclude nulls values before working on the data.

Aggregate Functions:
* COUNT
* SUM
* AVG
* MIN
* MAX

```sql
select * from category;
 cat_id | cat_name 
--------+----------
      1 | COMPUTER
      2 | MOBILE
      3 | FASHION
(3 rows)

select * from products;
 prod_id |     prod_name     |  price   | cid 
---------+-------------------+----------+-----
    1001 | ACER ASPIRE       | 33000.00 |   1
    1002 | DELL INSPIRON     | 43000.00 |   1
    1003 | LENOVE IDEAPAD    | 18000.00 |   1
    1004 | SAMSUNG GALAXY S9 | 35000.00 |   2
    1005 | IPHONE 7          | 60000.00 |   2
    1006 | HONOR 6X          |  9000.00 |   2
    1007 | NOKIA 1100        |  4000.00 |    
(7 rows)
```

**COUNT**

The COUNT function returns the total number of values in the specified field   
When an asterisk (*) is used with count function, returns the total number of rows.

Number of products in product table

```sql
select count(*) "Total Products" from products;
 Total Products 
----------------
              7
(1 row)
```

Count num of products that have price>=25000 in mobile category

```sql
select count(prod_id) from products p inner join category c on p.cid=c.cat_id where c.cat_name='MOBILE' and p.price>=25000;
 count 
-------
     2
(1 row)
```
**SUM**   
The SUM function returns the sum of all the values in the specified column.   

Total price of mobiles
```sql
select sum(p.price) "Total price" from products p inner join category c on p.cid=c.cat_id where c.cat_name='MOBILE';
 Low End Mobile 
----------------
      104000.00
(1 row)
```
**AVG**   
AVG function returns the average of the values in a specified column

Average price for mobile category
```sql
select round(avg(p.price),2) as avg_price from products p inner join category c on p.cid=c.cat_id where c.cat_name='MOBILE';

avg_price   
----------
 34666.67
(1 row)
```
**MIN**  
The MIN function returns the smallest value in the specified table field.

Cheapest mobile price   
```sql
select min(p.price) "Low End Mobile" from products p inner join category c on p.cid=c.cat_id where c.cat_name='MOBILE';

 Low End Mobile 
----------------
        9000.00
(1 row)
```
**MAX**    
The MAX function returns the largest value from the specified table field.

Highest priced mobile price   
```sql
select max(p.price) "Top End Mobile" from products p inner join category c on p.cid=c.cat_id where c.cat_name='MOBILE';
 Top End Mobile 
----------------
       60000.00
(1 row)
```

## GROUP BY - HAVING
***

**GROUP BY**   
The group by clause is used to divide the rows in a table into smaller groups that have the same values in the specified columns.

**HAVING**  
HAVING clause is used for conditional retrieval of rows from a grouped result where  ***WHERE*** clause can only use to restrict individual rows.  
- HAVING cannot be used without grouping.


Display num of products, max price, min price in each category name

```sql
select c.cat_name,count(p.prod_id),max(p.price),min(p.price) from category
c LEFT JOIN products p on c.cat_id=p.cid group by c.cat_name;
 cat_name | count |   max    |   min    
----------+-------+----------+----------
 MOBILE   |     3 | 60000.00 |  9000.00
 COMPUTER |     3 | 43000.00 | 18000.00
 FASHION  |     0 |          |         
(3 rows)
```
Display the avg price for all categories and exclude the group that contains one product or less

```sql
select c.cat_name,round(avg(p.price),2) "Avg Price" from category c LEFT JOIN products p
on c.cat_id=p.cid group by c.cat_name having count(p.prod_id)>1;
 cat_name | Avg Price 
----------+-----------
 MOBILE   |  34666.67
 COMPUTER |  31333.33
(2 rows)
```
Display num of products for all categories whose price is greater than or equal to 10000 and exclude the group that have less than two products
 
```sql
select c.cat_name,count(p.prod_id) "No. of products" from category c LEFT JOIN products p 
on c.cat_id=p.cid where p.price>=10000 group by c.cat_name having count(p.prod_id)>=2;

 cat_name | No. of products 
----------+-----------------
 MOBILE   |               2
 COMPUTER |               3
(2 rows)
```

## Subqueries
***

A subquery is a query within a query.   
The main query that contains the subquery is called the OUTER QUERY.   
A subquery is also called an INNER QUERY.   
The inner query executes first before its parent query so that the results of an inner query can be passed to the outer query.

 + A subquery must be enclosed in parentheses. 
 + A subquery must be placed on the right side of the       comparison operator. 
+ Subqueries cannot manipulate their results internally, therefore ORDER BY clause cannot be added into a subquery.
+ Use single-row operators(<,>,=,!=,>=,<=) with single-row subqueries. 
+ If a subquery returns a null value to the outer query, the outer query will not return any rows when using certain comparison operators in a WHERE clause.
+ Subqueries can also be nested

A subquery may occur in :
  + A SELECT clause
  + A FROM clause
   + A WHERE clause


###	Types of Subquery:

**Single row subquery** :    
Returns zero or one row. (Single row comparison operators are used in where clause i.e =,>,<,>=,<=,!=)

Display the employees whose salary is higher than the average salary throughout the company.
```sql
select * from cmr_employee where sal>=(select coalesce(avg(sal),0) from cmr_employee);
	 eid  | ename |    sal    | dept | mgr_id 
	------+-------+-----------+------+--------
	 1002 | tom   |  65000.00 | pr   |       
	 1005 | peter |  85000.00 | pr   |   1002
	 1008 | ravi  | 125000.00 | hr   |   1004
	(3 rows)
```
**Multiple row subquery** :       
Returns one or more rows. (multiple row comparison operators like IN, ANY, ALL are used)

When **ALL** is used it should match with everything in inner query.    
When **ANY** is used atleast one of them should be matched.

Display the employee who are managers
```sql	
select ename from cmr_employee where eid in(select distinct mgr_id from cmr_employee );
	   ename    
	------------
	 ram
	 tom
	 sai kumar
	 rana singh
	(4 rows)
```

Display the employee details who earn max sal in each dept
```sql	
select * from cmr_employee where sal in(select max(sal) from cmr_employee group by dept);

	 eid  | ename |    sal    | dept | mgr_id 
	------+-------+-----------+------+--------
	 1005 | peter |  85000.00 | pr   |   1002
	 1008 | ravi  | 125000.00 | hr   |   1004
	(2 rows)

select empno,empname from emp where dep_id=ANY (select dep_id from department where loc_id=1700)

select dep_id,avg(sal) from emp group by dep_id having avg(sal)>=all(select avg(sal) from emp group by dep_id);

```         
**Multiple column subqueries** :          
Returns one or more columns. 

Display the employee details who earns min sal in each dept. 
```sql	
select * from cmr_employee where (dept,sal) in (select dept,min(sal) from cmr_employee group by dept);
	 eid  |   ename   |   sal    | dept | mgr_id 
	------+-----------+----------+------+--------
	 1003 | sai kumar | 35000.00 | hr   |   1001
	 1006 | balu      | 43000.00 | pr   |   1002
	 1007 | anand     | 35000.00 | hr   |   1003
	(3 rows)

```       
**Correlated subqueries** :          
Reference one or more columns in the outer SQL statement.The subquery is known as a correlated subquery because the subquery is related to the outer SQL statement.

Display the employees whose salary is more than the average salary in each department.

```sql
select * from cmr_employee c where sal>=(select avg(coalesce(sal,0)) from cmr_employee e where e.dept=c.dept);
	
    eid  | ename |    sal    | dept | mgr_id 
	------+-------+-----------+------+--------
	 1002 | tom   |  65000.00 | pr   |       
	 1005 | peter |  85000.00 | pr   |   1002
	 1008 | ravi  | 125000.00 | hr   |   1004
	(3 rows)
```
Display employees with their manager names
```sql
select e.ename "employee",(select m.ename from cmr_employee m  where e. mgr_id=m.eid) as manager from cmr_employee e;

	  employee  |  manager   
	------------+------------
	 ram        | 
	 tom        | 
	 sai kumar  | ram
	 rana singh | ram
	 peter      | tom
	 balu       | tom
	 anand      | sai kumar
	 ravi       | rana singh
	(8 rows)
   ```

The subquery can be nested inside a SELECT, INSERT, UPDATE or DELETE statement or inside another subquery

```sql
insert into employee_temp select *  from employee where emp_id in(select emp_id from employee );
	INSERT 0 11

update employee_temp set emp_sal=emp_sal*1.10 where emp_id in (select emp_id from employee_temp where emp_dept='pr');
	UPDATE 4
	
   select * from cmr_employee ;
	
    eid  |   ename    |    sal    | dept | mgr_id 
	------+------------+-----------+------+--------
	 1001 | ram        |  55000.00 | hr   |       
	 1002 | tom        |  65000.00 | pr   |       
	 1003 | sai kumar  |  35000.00 | hr   |   1001
	 1004 | rana singh |  45000.00 | hr   |   1001
	 1005 | peter      |  85000.00 | pr   |   1002
	 1006 | balu       |  43000.00 | pr   |   1002
	 1007 | anand      |  35000.00 | hr   |   1003
	 1008 | ravi       | 125000.00 | hr   |   1004
	(8 rows)
```
## CASE - COALESCE
***

**CASE**  
The SQL CASE expression is a generic conditional expression, similar to if/else statements in other languages:

Syntax:  
```
CASE
  WHEN <search-condition> THEN <output>
  WHEN <search-condition> THEN <output>
  ...
  ELSE <output>
END
```


Shorthand notation
```
CASE <search-parameter>
  WHEN <search-value> THEN <output>
  WHEN <search-value> THEN <output>
  ...
  ELSE <output>
END
```

Tables for querying
```sql
select * from category;
 cat_id | cat_name 
--------+----------
      1 | COMPUTER
      2 | MOBILE
      3 | FASHION
(3 rows)

select * from products;
 prod_id |     prod_name     |  price   | cid 
---------+-------------------+----------+-----
    1001 | ACER ASPIRE       | 33000.00 |   1
    1002 | DELL INSPIRON     | 43000.00 |   1
    1003 | LENOVE IDEAPAD    | 18000.00 |   1
    1004 | SAMSUNG GALAXY S9 | 35000.00 |   2
    1005 | IPHONE 7          | 60000.00 |   2
    1006 | HONOR 6X          |  9000.00 |   2
    1007 | NOKIA 1100        |  4000.00 |    
(7 rows)
```

Increase the prices of mobile by 50%

```sql
select p.prod_name,p.price,c.cat_name,case
when c.cat_name='MOBILE' then 1.50*p.price
else p.price                                
end         
as "Revised Prices"
from products p inner join category c on p.cid=c.cat_id;

     prod_name     |  price   | cat_name | Revised Prices 
-------------------+----------+----------+----------------
 ACER ASPIRE       | 33000.00 | COMPUTER |       33000.00
 DELL INSPIRON     | 43000.00 | COMPUTER |       43000.00
 LENOVE IDEAPAD    | 18000.00 | COMPUTER |       18000.00
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE   |     52500.0000
 IPHONE 7          | 60000.00 | MOBILE   |     90000.0000
 HONOR 6X          |  9000.00 | MOBILE   |     13500.0000
(6 rows)
```

Shorthand notation


Increase the prices of mobile by 20% and computers by 10%
```sql
select p.prod_name,p.price,c.cat_name,case c.cat_name
when 'MOBILE' then 1.50*p.price
when 'COMPUTER' then 1.10*p.price
else p.price
end
as "Revised Prices"
from products p inner join category c on p.cid=c.cat_id;

     prod_name     |  price   | cat_name | Revised Prices 
-------------------+----------+----------+----------------
 ACER ASPIRE       | 33000.00 | COMPUTER |     36300.0000
 DELL INSPIRON     | 43000.00 | COMPUTER |     47300.0000
 LENOVE IDEAPAD    | 18000.00 | COMPUTER |     19800.0000
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE   |     52500.0000
 IPHONE 7          | 60000.00 | MOBILE   |     90000.0000
 HONOR 6X          |  9000.00 | MOBILE   |     13500.0000
(6 rows)
```
**COALESCE**     
This is often useful to substitute a default value for NULL values when data is retrieved for display.   
* The COALESCE function returns the first of its arguments that is NOT NULL.    
* NULL is returned only if all arguments are NULL. 
* The arguments in the coalesce must be of same data type.
```sql
select coalesce(null,null,null,100,20);
 coalesce 
----------
      100
(1 row)

select coalesce(null,100,'jones');
ERROR:  invalid input syntax for integer: "jones"
LINE 1: select coalesce(null,100,'jones');
```
Find the average sal of employees;

Without using coalesce
```sql
select round(avg(emp_sal),2) from employee;
  round   
----------
 34500.00
(1 row)
```

With coalesce
```sql
select round(avg(coalesce(emp_sal,0)),2) from employee;
  round   
----------
 31363.64
```
Table for querying
```sql
select * from person;

 first_name | middle_name | last_name  
------------+-------------+------------
 Sandeep    |             | Alimi
 Sachin     |             | Vadlakonda
 Bharath    | chary       | B
 Gayatri    | Kapoor      | G
 Srinivas   | kumar       | K
 Vivek      | yadav       | Gorla
(6 rows)
```
Without COALESCE
```sql
select first_name ||' '||middle_name||' '||last_name as full_name from person;
     full_name     
-------------------
 
 
 Bharath chary B
 Gayatri Kapoor G
 Srinivas kumar K
 Vivek yadav Gorla
(6 rows)
```
With COALESCE
```sql
select first_name ||' '||coalesce(middle_name,' ')||' '||last_name as full_name from person;
      full_name      
---------------------
 Sandeep   Alimi
 Sachin   Vadlakonda
 Bharath chary B
 Gayatri Kapoor G
 Srinivas kumar K
 Vivek yadav Gorla
(6 rows)
```
## DISTINCT and DISTINCT ON
***

DISTINCT removes all the duplicates rows.       
Must follow select command.

```sql
select * from employee limit 10;
 id |         name         |      department      |  salary  
----+----------------------+----------------------+----------
  1 | Gabi Croom           | Engineering          | 20680.17
  2 | Loreen Skayman       | Engineering          | 14392.06
  3 | Cornelia Cochet      | Business Development | 35137.12
  4 | Chicky Spraggs       | Sales                | 14034.18
  5 | Giulia Andersch      | Legal                | 21737.08
  6 | Marie Lillecrap      | Business Development | 33189.32
  7 | Ashleigh Hurrell     | Product Management   | 29250.78
  8 | Isabel Ogilvie       | Legal                |  8465.57
  9 | Siward Jolland       | Human Resources      |  7902.90
 10 | Jacynth Killingworth | Training             | 10018.15
(10 rows)

select distinct department from employee limit 10;
        department        
--------------------------
 Engineering
 Training
 Business Development
 Sales
 Marketing
 Accounting
 Services
 Support
 Research and Development
 Legal
(10 rows)


 ```
 If you specify multiple columns, the DISTINCT clause will evaluate the duplicate based on the combination of values of these columns.  

**Distinct On**:

PostgreSQL also provides the DISTINCT ON (expression) to keep the “first” row of each group of duplicates using the following 
```sql
select * from employee where salary in (select max(salary) from employee group by department) order by department;


  id  |         name         |        department        |  salary  
------+----------------------+--------------------------+----------
  227 | Avrom Formilli       | Accounting               | 49980.75
  964 | Linell Probert       | Business Development     | 49612.71
  311 | Nehemiah Shee        | Engineering              | 49916.01
  966 | Collie Defty         | Human Resources          | 49879.48
  639 | Nikaniki Prin        | Legal                    | 49830.76
  727 | Issy Pakenham        | Marketing                | 49884.90
  491 | Tami Capell          | Product Management       | 49225.95
  662 | Renato Saiz          | Research and Development | 49909.68
  127 | Roslyn Ellicombe     | Sales                    | 49444.99
  922 | Gael Basset          | Services                 | 49314.11
  538 | Alexandros Jozefczak | Support                  | 49874.84
 1000 | Karee Featherstone   | Training                 | 49675.26
(12 rows)



SELECT DISTINCT ON (department)
    *
FROM
    employee
ORDER BY
    department,
    salary DESC;
  id  |         name         |        department        |  salary  
------+----------------------+--------------------------+----------
  227 | Avrom Formilli       | Accounting               | 49980.75
  964 | Linell Probert       | Business Development     | 49612.71
  311 | Nehemiah Shee        | Engineering              | 49916.01
  966 | Collie Defty         | Human Resources          | 49879.48
  639 | Nikaniki Prin        | Legal                    | 49830.76
  727 | Issy Pakenham        | Marketing                | 49884.90
  491 | Tami Capell          | Product Management       | 49225.95
  662 | Renato Saiz          | Research and Development | 49909.68
  127 | Roslyn Ellicombe     | Sales                    | 49444.99
  922 | Gael Basset          | Services                 | 49314.11
  538 | Alexandros Jozefczak | Support                  | 49874.84
 1000 | Karee Featherstone   | Training                 | 49675.26
(12 rows)

   ``` 


The main difference between the two is DISTINCT applies to the entire tuple (columns) in the query mentioned whereas the DISTINCT ON is used to perform on the single column (can also be used for multiple columns).

## COPY
***

COPY - a mechanism to bulk load data in or out of Postgres.



To import entire data along with headers from csv
```sql
copy emp(empno,empname,hourpay,sal,commission,numsales,dno,dep_id) from 'path-to-file/test.csv' delimiter ',' csv header;
COPY 16
```
If the csv contains all columns then column names are not reqiured 
```sql
copy emp from 'path-to-file/test.csv' delimiter ',' csv header;
COPY 16
```
To import only few rows from file
```sql
copy temp(empno,empname,hourpay,sal) from 'path-to-file/newtest.csv' delimiter ',' csv header;
COPY 16
```
To export data from postgres to files

```sql
copy customers to 'path-to-file/customers.csv' delimiter ',' csv header;
COPY 4
```
To export only specific columns from Postgres to files
```sql
copy customers(name) to 'path-to-file/customers.csv' delimiter ',' csv header;
COPY 4
```
> Note : You need file permissions to write to a file

## Temporary Tables
***

A temporary table is a short-lived table that exists for the duration of a database session or current transaction.    
Dropped at the end of a session or transaction.       

A temporary table is visible only to the session that creates it. Any indexes created on a temporary table are also automatically temporary.

Can have same name as a permanent table but we cannot access the permanent table until temp table is dropped

```sql
CREATE TABLE customers(id SERIAL PRIMARY KEY, name VARCHAR NOT NULL);
CREATE TEMP TABLE customers(customer_id INT);
 select * from customers;
 customer_id 
-------------
(0 rows)
```
The temprorary table is created in a special schema (pg_temp_nn)

```sql
\dt
            List of relations
  Schema   |   Name    | Type  |  Owner   
-----------+-----------+-------+----------
 pg_temp_2 | customers | table | postgres
 public    | test      | table | postgres

```


To access permanent table schema name is used
```sql
\d
                 List of relations
  Schema   |       Name       |   Type   |  Owner   
-----------+------------------+----------+----------
 pg_temp_2 | customers        | table    | postgres
 public    | customers_id_seq | sequence | postgres
 public    | test             | table    | postgres
(3 rows)

select * from public.customers;
 id | name 
----+------
(0 rows)

```

To drop a temporary table from the database.
```sql
drop table customers;
DROP TABLE

\dt
           List of relations
 Schema |   Name    | Type  |  Owner   
--------+-----------+-------+----------
 public | customers | table | postgres
 public | test      | table | postgres
(2 rows)
```
## VIEWS
***
* A view is named query that provides another way to present data in the database tables.   
* Views are based on one or more tables, which are known as base tables.   
* Views does not contain any data.   
* Views are definitions built on top of other tables (or views).   
* If data is changed in the underlying table, the same change is reflected in the view.   
* A view can be accessed as a virtual table in PostgreSQL.   
* Temporary views can also be created.
```sql
create view part_view as select * from part limit 5 offset 100;
CREATE VIEW


select * from part_view;
   id   |   partno   |  partname   | partdescr | machine_id | parttype 
--------+------------+-------------+-----------+------------+----------
 495474 | PNo:495474 | Part 495474 |           |     495474 | Engine
 495504 | PNo:495504 | Part 495504 |           |     495504 | Brakes
 495567 | PNo:495567 | Part 495567 |           |     495567 | Steering
 495579 | PNo:495579 | Part 495579 |           |     495579 | Brakes
 495677 | PNo:495677 | Part 495677 |           |     495677 | Steering
(5 rows)
```
To change the view definition
```sql
create or replace view part_view as select * from part where machine_id between 400000 and 400010;
CREATE VIEW
```
* You can't drop the column's from the view

To rename a view
```sql
alter view part_view rename to view_part;
ALTER VIEW
```



## Materialized Views

Materialized view stores data physically and refreshes the data periodically from the base tables.   

The materialized views have many advantages in many scenarios such as faster access to data from a remote server, data caching etc.

Disadvantage of simple view   
* Each time a view is used in a query, the query that created the view is executed.   
* This makes simple views inefficient to access time

Materialized views allow you to persist a view in the database physically

Creating a materialized view.
```sql
create materialized view part_view as select * from part where machine_id between 400000 and 400010;
SELECT 11
```

Load no data into the materialized view while you are creating it.
```sql
create materialized view part_view as select * from part where machine_id between 400000 and 400010 with NO DATA;
CREATE MATERIALIZED VIEW
Time: 77.148 ms
```

In case you use WITH NO DATA, the view is flagged as unreadable. It means that you cannot query data from the view until you load data into it.

```sql
select * from part_view;
ERROR:  materialized view "part_view" has not been populated
HINT:  Use the REFRESH MATERIALIZED VIEW command.
Time: 0.969 ms
```

To load data into a materialized view.
```sql
testdb: refresh materialized view  part_view;
REFRESH MATERIALIZED VIEW
Time: 269.896 ms

 select * from part_view;
   id   |   partno   |  partname   | partdescr | machine_id |  parttype  
--------+------------+-------------+-----------+------------+------------
 400004 | PNo:400004 | Part 400004 |           |     400004 | Suspension
 400002 | PNo:400002 | Part 400002 |           |     400002 | Brakes
 400009 | PNo:400009 | Part 400009 |           |     400009 | Steering
 400000 | PNo:400000 | Part 400000 |           |     400000 | Brakes
 400001 | PNo:400001 | Part 400001 |           |     400001 | General
 400007 | PNo:400007 | Part 400007 |           |     400007 | Suspension
 400010 | PNo:400010 | Part 400010 |           |     400010 | Suspension
 400008 | PNo:400008 | Part 400008 |           |     400008 | Engine
 400006 | PNo:400006 | Part 400006 |           |     400006 | Steering
 400005 | PNo:400005 | Part 400005 |           |     400005 | Driveline
 400003 | PNo:400003 | Part 400003 |           |     400003 | Brakes
(11 rows)

Time: 2.937 ms
```
When you refresh data for a materialized view, 
PostgreSQL locks the entire table therefore you cannot query data against it.

> REFRESH MATERIALIZED VIEW CONCURRENTLY view_name;

With CONCURRENTLY option, PostgreSQL creates a temporary updated version of the materialized view, 
compares two versions and performs INSERT and UPDATE only the differences.    

You can query against the materialized view while it is being updated.    

One requirement for using CONCURRENTLY option is that the materialized view must have a UNIQUE index. 

## Common Table Expressions (CTE's)
***

Common Table Expressions or CTE's can be thought of as defining temporary tables that exist just for one query.   
CTEs are similar to a view that is materialized only while that query is running and does not exist outside of that query. 

+ Allows large queries to be more readable.
+ Simplifies complex queries and joins.
+ Postgres will evaluate the query inside the CTE and store the results in a temporary table.

```sql
create table foo (id int, padding text);
CREATE TABLE
insert into foo (id, padding) select id, md5(random()::text) from generate_series(1, 1000000) as id order by random();

INSERT 0 1000000

with cte as (select * from foo) select * from cte where id = 500000;
   id   |             padding              
--------+----------------------------------
 500000 | a61e648432b2197021d534e80cd8ebd2
(1 row)

select * from cmr_employee;
 eid  |   ename    |    sal    | dept | mgr_id 
------+------------+-----------+------+--------
 1001 | ram        |  55000.00 | hr   |       
 1002 | tom        |  65000.00 | pr   |       
 1003 | sai kumar  |  35000.00 | hr   |   1001
 1004 | rana singh |  45000.00 | hr   |   1001
 1005 | peter      |  85000.00 | pr   |   1002
 1006 | balu       |  43000.00 | pr   |   1002
 1007 | anand      |  35000.00 | hr   |   1003
 1008 | ravi       | 125000.00 | hr   |   1004
(8 rows)

```
Display employee details with manager name of all employee

```sql
with emp_join_cte as (
select * from cmr_employee)
select e.eid,e.ename,e.sal,e.dept,cte.ename from cmr_employee e LEFT JOIN emp_join_cte cte on e.mgr_id=cte.eid;

 eid  |   ename    |    sal    | dept |   ename    
------+------------+-----------+------+------------
 1001 | ram        |  55000.00 | hr   | 
 1002 | tom        |  65000.00 | pr   | 
 1003 | sai kumar  |  35000.00 | hr   | ram
 1004 | rana singh |  45000.00 | hr   | ram
 1005 | peter      |  85000.00 | pr   | tom
 1006 | balu       |  43000.00 | pr   | tom
 1007 | anand      |  35000.00 | hr   | sai kumar
 1008 | ravi       | 125000.00 | hr   | rana singh
```
**Recursive CTE's:**

CTEs which allow themselves to be called until some condition is met.     
Recursive queries are typically used to deal with hierarchical or tree-structured data.

+ Possible to express recursion in sql.
+ Works with the concept of working table (which is a temporary table to store results)

Syntax:

```sql
with recursive_name (columns) as(
 <initial query>
union all
<recursive query>
)
<query>
```

Display the employee who are working under ram

```sql
WITH RECURSIVE manager_tree AS (
select eid,ename,mgr_id from cmr_employee where eid=1001 
UNION ALL
select e.eid,e.ename,e.mgr_id from cmr_employee e INNER JOIN manager_tree mtree on e.mgr_id=mtree.eid)
select * from manager_tree;

 eid  |   ename    | mgr_id 
------+------------+--------
 1001 | ram        |       
 1003 | sai kumar  |   1001
 1004 | rana singh |   1001
 1007 | anand      |   1003
 1008 | ravi       |   1004
(5 rows)
```

## Window Function's
***
Window functions are used to perform an operation across a set of table rows that are related to current row.  
Comparable as of aggregate function but the rows retain their identities.   
The window function is able to access more than just the current row. It does not reduce the number of rows in a window.

**PARTITION BY**

The PARTITION BY clause divides rows into multiple groups or partitions to which the window function is applied. 
Like the example below, we used the product group to divide the products into groups (or partitions).

The PARTITION BY clause is optional. If you skip the PARTITION BY clause, the window function will treat the whole result set as a single partition.

**ORDER BY**         

The ORDER BY clause specifies the order of rows in each partition to which the window function is applied.

The ORDER BY clause uses the NULLS FIRST or NULLS LAST option to specify whether nullable values should be first or last in the result set. The default is NULLS LAST option.
```sql
select * from category;
 cat_id |  cat_name  
--------+------------
      1 | COMPUTER
      2 | MOBILE
      3 | FASHION
      4 | EAR PHONES
(4 rows)

select * from products;
 prod_id |     prod_name     |  price   | cid 
---------+-------------------+----------+-----
    1001 | ACER ASPIRE       | 33000.00 |   1
    1002 | DELL INSPIRON     | 43000.00 |   1
    1003 | LENOVE IDEAPAD    | 18000.00 |   1
    1004 | SAMSUNG GALAXY S9 | 35000.00 |   2
    1005 | IPHONE 7          | 60000.00 |   2
    1006 | HONOR 6X          |  9000.00 |   2
    1007 | NOKIA 1100        |  4000.00 |    
    1008 | REAL ME           | 15000.00 |   2
    1009 | OPPO              | 25000.00 |   2
    1015 | VIVO V9           | 15000.00 |   2
    1016 | NOISE SHOTS       |  1500.00 |   4
    1017 | AIRBUDS AIR       |  3500.00 |   4
    1018 | AIRPODS           | 15000.00 |   4
    1019 | BOAT AIR DOPES    |  5000.00 |   4
(14 rows)
``` 
```sql
select cat_name,AVG(price) from products p inner join category c on p.cid=c.cat_id group by cat_name;
  cat_name  |          avg          
------------+-----------------------
 EAR PHONES | 6250.0000000000000000
 MOBILE     |    26500.000000000000
 COMPUTER   |    31333.333333333333
(3 rows)


--WITH window function

select prod_id,prod_name,price,cat_name,AVG(price) OVER
( PARTITION BY cat_name ) from products p inner join category c on p.cid=c.cat_id

 prod_id |     prod_name     |  price   |  cat_name  |          avg          
---------+-------------------+----------+------------+-----------------------
    1001 | ACER ASPIRE       | 33000.00 | COMPUTER   |    31333.333333333333
    1002 | DELL INSPIRON     | 43000.00 | COMPUTER   |    31333.333333333333
    1003 | LENOVE IDEAPAD    | 18000.00 | COMPUTER   |    31333.333333333333
    1019 | BOAT AIR DOPES    |  5000.00 | EAR PHONES | 6250.0000000000000000
    1016 | NOISE SHOTS       |  1500.00 | EAR PHONES | 6250.0000000000000000
    1017 | AIRBUDS AIR       |  3500.00 | EAR PHONES | 6250.0000000000000000
    1018 | AIRPODS           | 15000.00 | EAR PHONES | 6250.0000000000000000
    1009 | OPPO              | 25000.00 | MOBILE     |    26500.000000000000
    1015 | VIVO V9           | 15000.00 | MOBILE     |    26500.000000000000
    1004 | SAMSUNG GALAXY S9 | 35000.00 | MOBILE     |    26500.000000000000
    1005 | IPHONE 7          | 60000.00 | MOBILE     |    26500.000000000000
    1006 | HONOR 6X          |  9000.00 | MOBILE     |    26500.000000000000
    1008 | REAL ME           | 15000.00 | MOBILE     |    26500.000000000000
(13 rows)
```
All aggregate functions AVG(), MIN(), MAX(), SUM() and COUNT() can be used as window functions.

Window functions provided by Postgres: 

**ROW_NUMBER**

Number the current row within its partition starting from 1.

```sql
select prod_id,prod_name,cat_name,price,ROW_NUMBER () over ( PARTITION by cat_name order by price)from products p INNER JOIN category c on p.cid=c.cat_id;

 prod_id |     prod_name     |  cat_name  |  price   | row_number 
---------+-------------------+------------+----------+------------
    1003 | LENOVE IDEAPAD    | COMPUTER   | 18000.00 |          1
    1001 | ACER ASPIRE       | COMPUTER   | 33000.00 |          2
    1002 | DELL INSPIRON     | COMPUTER   | 43000.00 |          3
    1016 | NOISE SHOTS       | EAR PHONES |  1500.00 |          1
    1017 | AIRBUDS AIR       | EAR PHONES |  3500.00 |          2
    1019 | BOAT AIR DOPES    | EAR PHONES |  5000.00 |          3
    1018 | AIRPODS           | EAR PHONES | 15000.00 |          4
    1006 | HONOR 6X          | MOBILE     |  9000.00 |          1
    1015 | VIVO V9           | MOBILE     | 15000.00 |          2
    1008 | REAL ME           | MOBILE     | 15000.00 |          3
    1009 | OPPO              | MOBILE     | 25000.00 |          4
    1004 | SAMSUNG GALAXY S9 | MOBILE     | 35000.00 |          5
    1005 | IPHONE 7          | MOBILE     | 60000.00 |          6
(13 rows)
```

**RANK**                                                     

Rank the current row within its partition with gaps.

```sql
select prod_id,prod_name,cat_name,price,
RANK () over ( PARTITION by cat_name order by price)
from products p INNER JOIN category c on p.cid=c.cat_id;

 prod_id |     prod_name     |  cat_name  |  price   | rank 
---------+-------------------+------------+----------+------
    1003 | LENOVE IDEAPAD    | COMPUTER   | 18000.00 |    1
    1001 | ACER ASPIRE       | COMPUTER   | 33000.00 |    2
    1002 | DELL INSPIRON     | COMPUTER   | 43000.00 |    3
    1016 | NOISE SHOTS       | EAR PHONES |  1500.00 |    1
    1017 | AIRBUDS AIR       | EAR PHONES |  3500.00 |    2
    1019 | BOAT AIR DOPES    | EAR PHONES |  5000.00 |    3
    1018 | AIRPODS           | EAR PHONES | 15000.00 |    4
    1006 | HONOR 6X          | MOBILE     |  9000.00 |    1
    1015 | VIVO V9           | MOBILE     | 15000.00 |    2
    1008 | REAL ME           | MOBILE     | 15000.00 |    2
    1009 | OPPO              | MOBILE     | 25000.00 |    4
    1004 | SAMSUNG GALAXY S9 | MOBILE     | 35000.00 |    5
    1005 | IPHONE 7          | MOBILE     | 60000.00 |    6
(13 rows)


--Rank based on the price from the table. It gives the same rank for two rows with the same price and then skips the immediate next number.
```
**DENSE_RANK**

Rank the current row within its partition without gaps.

```sql
select prod_id,prod_name,cat_name,price,
DENSE_RANK () over ( PARTITION by cat_name order by price)
from products p INNER JOIN category c on p.cid=c.cat_id;

 prod_id |     prod_name     |  cat_name  |  price   | dense_rank 
---------+-------------------+------------+----------+------------
    1003 | LENOVE IDEAPAD    | COMPUTER   | 18000.00 |          1
    1001 | ACER ASPIRE       | COMPUTER   | 33000.00 |          2
    1002 | DELL INSPIRON     | COMPUTER   | 43000.00 |          3
    1016 | NOISE SHOTS       | EAR PHONES |  1500.00 |          1
    1017 | AIRBUDS AIR       | EAR PHONES |  3500.00 |          2
    1019 | BOAT AIR DOPES    | EAR PHONES |  5000.00 |          3
    1018 | AIRPODS           | EAR PHONES | 15000.00 |          4
    1006 | HONOR 6X          | MOBILE     |  9000.00 |          1
    1015 | VIVO V9           | MOBILE     | 15000.00 |          2
    1008 | REAL ME           | MOBILE     | 15000.00 |          2
    1009 | OPPO              | MOBILE     | 25000.00 |          3
    1004 | SAMSUNG GALAXY S9 | MOBILE     | 35000.00 |          4
    1005 | IPHONE 7          | MOBILE     | 60000.00 |          5
(13 rows)
```

**FIRST_VALUE**

Returns a value evaluated against the first row within its partition

```sql
select prod_id,prod_name,price,cat_name,FIRST_VALUE(price) OVER
( PARTITION BY cat_name order by price) as LOWEST_PRICE_PER_GROUP from products p inner join category c on p.cid=c.cat_id;

 prod_id |     prod_name     |  price   |  cat_name  | lowest_price_per_group 
---------+-------------------+----------+------------+------------------------
    1003 | LENOVE IDEAPAD    | 18000.00 | COMPUTER   |               18000.00
    1001 | ACER ASPIRE       | 33000.00 | COMPUTER   |               18000.00
    1002 | DELL INSPIRON     | 43000.00 | COMPUTER   |               18000.00
    1016 | NOISE SHOTS       |  1500.00 | EAR PHONES |                1500.00
    1017 | AIRBUDS AIR       |  3500.00 | EAR PHONES |                1500.00
    1019 | BOAT AIR DOPES    |  5000.00 | EAR PHONES |                1500.00
    1018 | AIRPODS           | 15000.00 | EAR PHONES |                1500.00
    1006 | HONOR 6X          |  9000.00 | MOBILE     |                9000.00
    1015 | VIVO V9           | 15000.00 | MOBILE     |                9000.00
    1008 | REAL ME           | 15000.00 | MOBILE     |                9000.00
    1009 | OPPO              | 25000.00 | MOBILE     |                9000.00
    1004 | SAMSUNG GALAXY S9 | 35000.00 | MOBILE     |                9000.00
    1005 | IPHONE 7          | 60000.00 | MOBILE     |                9000.00
(13 rows)
```

**LAST_VALUE**

Returns a value evaluated against the last row in its partition or group.

```sql
select prod_id,prod_name,price,cat_name,LAST_VALUE(price) OVER
( PARTITION BY cat_name order by price) as HIGHEST_PRICE_PER_GROUP from products p inner join category c on p.cid=c.cat_id;

 prod_id |     prod_name     |  price   |  cat_name  | highest_price_per_group 
---------+-------------------+----------+------------+-------------------------
    1003 | LENOVE IDEAPAD    | 18000.00 | COMPUTER   |                18000.00
    1001 | ACER ASPIRE       | 33000.00 | COMPUTER   |                33000.00
    1002 | DELL INSPIRON     | 43000.00 | COMPUTER   |                43000.00
    1016 | NOISE SHOTS       |  1500.00 | EAR PHONES |                 1500.00
    1017 | AIRBUDS AIR       |  3500.00 | EAR PHONES |                 3500.00
    1019 | BOAT AIR DOPES    |  5000.00 | EAR PHONES |                 5000.00
    1018 | AIRPODS           | 15000.00 | EAR PHONES |                15000.00
    1006 | HONOR 6X          |  9000.00 | MOBILE     |                 9000.00
    1015 | VIVO V9           | 15000.00 | MOBILE     |                15000.00
    1008 | REAL ME           | 15000.00 | MOBILE     |                15000.00
    1009 | OPPO              | 25000.00 | MOBILE     |                25000.00
    1004 | SAMSUNG GALAXY S9 | 35000.00 | MOBILE     |                35000.00
    1005 | IPHONE 7          | 60000.00 | MOBILE     |                60000.00
(13 rows)

--With adding Frame clause

select prod_id,prod_name,price,cat_name,LAST_VALUE(price) OVER
 ( PARTITION BY cat_name order by price RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as HIGHEST_PRICE_PER_GROUP from products p inner join category c on p.cid=c.cat_id;
 prod_id |     prod_name     |  price   |  cat_name  | highest_price_per_group 
---------+-------------------+----------+------------+-------------------------
    1003 | LENOVE IDEAPAD    | 18000.00 | COMPUTER   |                43000.00
    1001 | ACER ASPIRE       | 33000.00 | COMPUTER   |                43000.00
    1002 | DELL INSPIRON     | 43000.00 | COMPUTER   |                43000.00
    1016 | NOISE SHOTS       |  1500.00 | EAR PHONES |                15000.00
    1017 | AIRBUDS AIR       |  3500.00 | EAR PHONES |                15000.00
    1019 | BOAT AIR DOPES    |  5000.00 | EAR PHONES |                15000.00
    1018 | AIRPODS           | 15000.00 | EAR PHONES |                15000.00
    1006 | HONOR 6X          |  9000.00 | MOBILE     |                60000.00
    1015 | VIVO V9           | 15000.00 | MOBILE     |                60000.00
    1008 | REAL ME           | 15000.00 | MOBILE     |                60000.00
    1009 | OPPO              | 25000.00 | MOBILE     |                60000.00
    1004 | SAMSUNG GALAXY S9 | 35000.00 | MOBILE     |                60000.00
    1005 | IPHONE 7          | 60000.00 | MOBILE     |                60000.00
(13 rows)

Notice that we added the frame clause RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 
because by default the frame clause is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW.
```

**LAG AND LEAD**

The LAG() function has the ability to access data from the previous row, while the LEAD() function can access data from the next row.

Syntax:
```
LAG/LEAD (expression [,offset] [,default])
```

Expression – a column or expression to compute the returned value.         
Offset – the number of rows preceding ( LAG)/ following ( LEAD) the current row. It defaults to 1.    
Default – the default returned value if the offset goes beyond the scope of the window. The default is NULL if you skip it.

```sql
select prod_name,cat_name,price,LAG(price,1) over ( PARTITION BY cat_name order by price) as prev_price,
price-LAG(price,1) over ( PARTITION BY cat_name order by price) as cur_prev_dif 
from products p inner join category c on p.cid=c.cat_id;

     prod_name     |  cat_name  |  price   | prev_price | cur_prev_dif 
-------------------+------------+----------+------------+--------------
 LENOVE IDEAPAD    | COMPUTER   | 18000.00 |            |             
 ACER ASPIRE       | COMPUTER   | 33000.00 |   18000.00 |     15000.00
 DELL INSPIRON     | COMPUTER   | 43000.00 |   33000.00 |     10000.00
 NOISE SHOTS       | EAR PHONES |  1500.00 |            |             
 AIRBUDS AIR       | EAR PHONES |  3500.00 |    1500.00 |      2000.00

---we can see prev price column which is fetched using the function LAG.

select prod_name,cat_name,price,
LEAD(price,1) over ( PARTITION BY cat_name order by price) as next_price,
price-LEAD(price,1) over ( PARTITION BY cat_name order by price) as cur_next_dif 
from products p inner join category c on p.cid=c.cat_id;

     prod_name     |  cat_name  |  price   | next_price | cur_next_dif 
-------------------+------------+----------+------------+--------------
 LENOVE IDEAPAD    | COMPUTER   | 18000.00 |   33000.00 |    -15000.00
 ACER ASPIRE       | COMPUTER   | 33000.00 |   43000.00 |    -10000.00
 DELL INSPIRON     | COMPUTER   | 43000.00 |            |             
 NOISE SHOTS       | EAR PHONES |  1500.00 |    3500.00 |     -2000.00
 ```

**NTH_VALUE**

The NTH_VALUE() function returns a value from the nth row in an ordered partition of a result set.

NTH_VALUE() function to return all product names together with the second most expensive product.

```SQL
with temp as (select prod_id,prod_name,price,cat_name,
NTH_VALUE(prod_name,2) over (partition by cat_name order by price desc range between UNBOUNDED PRECEDING and UNBOUNDED FOLLOWING)  "second" from products p inner join category c on p.cid=c.cat_id)
select t.*,pr.price from temp t inner join products pr on t.second = pr.prod_name ;

 prod_id |     prod_name     |  price   |  cat_name  |      second       |  price   
---------+-------------------+----------+------------+-------------------+----------
    1003 | LENOVE IDEAPAD    | 18000.00 | COMPUTER   | ACER ASPIRE       | 33000.00
    1001 | ACER ASPIRE       | 33000.00 | COMPUTER   | ACER ASPIRE       | 33000.00
    1002 | DELL INSPIRON     | 43000.00 | COMPUTER   | ACER ASPIRE       | 33000.00
    1006 | HONOR 6X          |  9000.00 | MOBILE     | SAMSUNG GALAXY S9 | 35000.00
    1008 | REAL ME           | 15000.00 | MOBILE     | SAMSUNG GALAXY S9 | 35000.00
    1015 | VIVO V9           | 15000.00 | MOBILE     | SAMSUNG GALAXY S9 | 35000.00
    1009 | OPPO              | 25000.00 | MOBILE     | SAMSUNG GALAXY S9 | 35000.00
    1004 | SAMSUNG GALAXY S9 | 35000.00 | MOBILE     | SAMSUNG GALAXY S9 | 35000.00
    1005 | IPHONE 7          | 60000.00 | MOBILE     | SAMSUNG GALAXY S9 | 35000.00
    1016 | NOISE SHOTS       |  1500.00 | EAR PHONES | BOAT AIR DOPES    |  5000.00
    1017 | AIRBUDS AIR       |  3500.00 | EAR PHONES | BOAT AIR DOPES    |  5000.00
    1019 | BOAT AIR DOPES    |  5000.00 | EAR PHONES | BOAT AIR DOPES    |  5000.00
    1018 | AIRPODS           | 15000.00 | EAR PHONES | BOAT AIR DOPES    |  5000.00
(13 rows)

```

**NTILE**

NTILE() function allows you to divide ordered rows in the partition into a specified number of ranked groups as equal size as possible. 

These ranked groups are called buckets.

The NTILE() function assigns each group a bucket number starting from 1.

Following query divides the products price range in a particular category to 3 categories( assume low, med, high) 

```sql
select prod_name,price,cat_name,NTILE(3) over (partition by cat_name  order by price) "Price range" from products p,category c where p.cid=c.cat_id;

      prod_name     |  price   |  cat_name  | Price range 
-------------------+----------+------------+-------------
 LENOVE IDEAPAD    | 18000.00 | COMPUTER   |           1
 ACER ASPIRE       | 33000.00 | COMPUTER   |           2
 DELL INSPIRON     | 43000.00 | COMPUTER   |           3
 NOISE SHOTS       |  1500.00 | EAR PHONES |           1
 AIRBUDS AIR       |  3500.00 | EAR PHONES |           1
 BOAT AIR DOPES    |  5000.00 | EAR PHONES |           2
 AIRPODS           | 15000.00 | EAR PHONES |           3
 HONOR 6X          |  9000.00 | MOBILE     |           1
 VIVO V9           | 15000.00 | MOBILE     |           1
 REAL ME           | 15000.00 | MOBILE     |           2
 OPPO              | 25000.00 | MOBILE     |           2
 SAMSUNG GALAXY S9 | 35000.00 | MOBILE     |           3
 IPHONE 7          | 60000.00 | MOBILE     |           3
(13 rows)
```
## Full Text Search
***
Full Text Searching (or just text search) provides the capability to identify natural-language documents that satisfy a query

 ~, ~*, LIKE, and ILIKE operators can be used but have disadvantages:  
 * There is no linguistic support
 * They provide no ordering (ranking) of search results
 * They tend to be slow because there is no index support
 * The regular expression can support LIKE operator searches up to a certain extent and 
 still lack the mechanism to handle plural words and derived words. 
 * A single word can have many derived words that are tedious to find out using regular expressions.

```sql
select content  from news where content like '%implement%';
                                    content                                    
-------------------------------------------------------------------------------
 Bare bones implementations of some of the foundational models and algorithms.
(1 row)

select content  from news where content like '%Implement%';
 content 
---------
(0 rows)
```

**to_tsvector**
* Parses the document to get a list of tokens, reduces tokens to lexemes and produces a list of lexemes with their position as a tsvector. 
* 'ts' in the tsvector represents ‘text search’. 

**to_tsquery**  
For querying the tsvector for certain words, phrases or complex queries separated by boolean operators & (AND), | (OR) or ! (NOT).
```sql
select * from to_tsvector('Bare bones implementations of some of the foundational models and algorithms.');
                             to_tsvector                              
----------------------------------------------------------------------
 'algorithm':11 'bare':1 'bone':2 'foundat':8 'implement':3 'model':9
(1 row)
```
Num indicates its presence in the text or document
```sql
select * from to_tsquery('english','delevering|products|to|customers'); 
             to_tsquery             
------------------------------------
 ( 'delev' | 'product' ) | 'custom'
(1 row)


select * from part where to_tsvector('english',partdescr) @@ to_tsquery('Foil');
   id   |   partno   |  partname   |                 partdescr                  | machine_id 
--------+------------+-------------+--------------------------------------------+------------
 300000 | Pno:300000 | Part:300000 | aluminium foils are used to make this part |          0

```
We can use AND (&),OR (|), NEGATION operator(!) in the to_tsquery to match with multiple words..
```sql
SELECT document_id, document_text FROM documents  
WHERE document_tokens @@ to_tsquery('jump & quick');  
 document_id |             document_text
-------------+---------------------------------------
           3 | The five boxing wizards jump quickly.
           4 | How vexingly quick daft zebras jump!
(2 rows)
```
The **ts_rank** function is used to get the relevancy score of the match (based on the frequency of matched lexemes).
```sql
select ts_rank(
to_tsvector('english', 'This department delivers the actual product to customers') , 
to_tsquery('english', 'product')
);
ts_rank
-----------
0.0607927106

select ts_rank(
  to_tsvector('english', 'This department delivers the actual product to customers') , 
  to_tsquery('english', 'product & customers')
);
ts_rank
-----------
0.0985008553


select ts_rank(
  to_tsvector('english', 'This department delivers the actual product to customers') , 
  to_tsquery('english', 'project')
);
ts_rank
-----------
0
```
> Performance Improvement on Full-Text Search   
>* Create a separate column for normalized vectors in same table or use materialized views and use index(GIN) on them for searching


## Transaction isolation levels
***

The SQL standard defines four levels of transaction isolation.
1. Serializable
2. Repeatable read
3. Read committed
4. Read uncommitted

* Every transaction has its isolation level set to one of these when it is created. The default level is “read committed”.
* Note that the SQL standard also defines “read uncommitted”, which is not supported in Postgres. 
   You have to use the nearest, higher level of “read committed”.

**Read Committed**   
dirty read:-  
A transaction trying to read uncommitted data from another transaction (concurrent transaction)

The read committed isolation level guarantees that dirty reads will never happen.
```sql
select * from t;
 a | b 
---+---
(0 rows)

T1: BEGIN;
T1: insert into t values(1,100);  
T1: select * from t;                                    
  a |  b              
  ---+-----           
  1 | 100             
  (1 row) 
                                            T2: select * from t;    
                                                    a |  b  
                                                    ---+-----
                                                        | 
                                                    (0 row) 

T1: commit;           

                                            T2: select * from t;        
                                                    a |  b  
                                                    ---+-----
                                                    1 | 100
                                                    (1 row)           

```
 The second transaction could not read the first transaction’s as-yet-uncommitted data. 
   
 In PostgreSQL, it is not possible to lower the isolation level to below this level so that dirty reads are allowed.

 **Repeatable read**
 
 nonrepeatable read:-   
 These happen when a transaction reads a row, and then reads it again a bit later but gets a different result – 
 because the row was updated in between by another transaction. 
 
 **repeatable read** will ensure that the second (or any) read will also return the same result as the first read. 
 
nonrepeatable read example
```sql
T1: begin;
  BEGIN
T1: select * from t;
        a |  b  
        ---+-----
        1 | 100
        (1 row)

                                            T2: begin;
                                                BEGIN
                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 100
                                                    (1 row)

T1: update t set b=200 where a=1;
  UPDATE 1
T1: select * from t;
        a |  b  
        ---+-----
        1 | 200
        (1 row)

                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 100
                                                    (1 row)

T1: commit;
  COMMIT

                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 200
                                                    (1 row)                                
```
With  Repeatable read isolation
```sql
T1: begin transaction isolation level repeatable read;
  BEGIN
T1: select * from t;
        a |  b  
        ---+-----
        1 | 100
        (1 row)

                                            T2: begin;
                                                BEGIN
                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 100
                                                    (1 row)

                                            T2: update t set b=200 where a=1;
                                                UPDATE 1
                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 200
                                                    (1 row)
T1: select * from t;
    a |  b  
    ---+-----
    1 | 100
    (1 row)


                                            T2: commit;
                                                COMMIT
                                                                                    
T1: select * from t;
    a |  b  
    ---+-----
    1 | 100
    (1 row)                                                
```
**Serializable**   
lost updates:-  
 Updates performed in one transaction can be “lost”, or overwritten by another transaction that happens to run concurrently

 Serialization provides the highest level of safety.

Lost update example
```sql
T1: begin;
  BEGIN
T1: select * from t;
        a |  b  
        ---+-----
        1 | 100
        (1 row)

                                            T2: begin;
                                                BEGIN
                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 100
                                                    (1 row)
T1: update t set b=200 where a=1;
  UPDATE 1
                            T2: update t set b=300 where a=1;
T1: commit;
  COMMIT                        
                            T2: UPDATE 1  

                                            T2: commit;
                          COMMIT

T1: select * from t;
        a |  b  
        ---+-----
        1 | 300
        (1 row)

                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 300
                                                    (1 row) 
```   

Here the second transaction’s UPDATE blocks, because PostgreSQL places a lock to prevent another update until the first transaction is finished.    

However, the first transaction’s change is lost, because the second one “overwrote” the row.

To avoid that we use serializable
```sql
T1: begin;
  BEGIN
T1: select * from t;
        a |  b  
        ---+-----
        1 | 100
        (1 row)

                                            T2: begin transaction isolation level serializable;
                                                BEGIN
                                            T2: select * from t;
                                                    a |  b  
                                                    ---+-----
                                                    1 | 100
                                                    (1 row)
T1: update t set b=200 where a=1;
  UPDATE 1
                            T2: update t set b=300 where a=1;

T1: commit;
  COMMIT                        
                            T2: ERROR:  could not serialize access due to concurrent update                                                    
```
At this level, the commit of the first transaction fails.   
The first transaction’s actions were based on facts that were rendered invalid by the time it was about to commit.
## Locking
***
To prevent situations where multiple users want to update the same data at the same time, locks are used to control this situation.

Postgres we have 3 mechanisms of locking:    
 * table-level, row-level, and advisory locks.    
 * Table and row-level locks can be explicit or implicit. Advisory locks are mainly explicit.    
 * Explicit locks are acquired on explicit user requests (with special queries) and implicit are acquired by standard SQL commands.         
  
**Table Level locks**   
Most of the table-level locks are acquired by built-in SQL commands, but they can also be acquired explicitly with LOCK command. 
 
 Syntax:
 LOCK [ TABLE ] name IN [ lock_mode_name ] mode;
 
 **ACCESS SHARE** – 
 all queries that only read the table acquire this lock.
 
 **ROW SHARE** – 
 The SELECT FOR UPDATE and SELECT FOR SHARE commands acquire this lock on the target table.
 
 **ROW EXCLUSIVE** – 
 The UPDATE, INSERT and DELETE commands acquire this lock on the target table, all queries that modify table acquire this lock.
 
 **SHARE UPDATE EXCLUSIVE** – 
 The VACUUM (without FULL), ANALYZE, CREATE INDEX CONCURRENTLY, and some forms of ALTER TABLE commands acquire this lock.
 
 **EXCLUSIVE** – 
 This lock mode allows only reads to process in parallel with a transaction that acquired this lock.
 
 **ACCESS EXCLUSIVE** – 
 The ALTER TABLE, DROP TABLE, TRUNCATE, REINDEX, CLUSTER, and VACUUM FULL commands acquire lock on a table referenced in query. This mode is the default mode of the LOCK command.
 
 |              Runs concurrently with              | SELECT | INSERT/UPDATE/DELETE | CREATE INDEX / CONC/ VACUUM/ ANALYZE | CREATE INDEX | CREATE TRIGGER | ALTER TABLE/ DROP TABLE/ TRUNCATE/ VACUUM / FULL |
|:------------------------------------------------:|:------:|:--------------------:|:------------------------------------:|:------------:|:--------------:|:------------------------------------------------:|
|                      SELECT                      |    ✅   |           ✅          |                   ✅                  |       ✅      |        ✅       |                         ❌                        |
|               INSERT/UPDATE/DELETE               |    ✅   |           ✅          |                   ✅                  |       ❌      |        ❌       |                         ❌                        |
|       CREATE INDEX / CONC/ VACUUM/ ANALYZE       |    ✅   |           ✅          |                   ❌                  |       ❌      |        ❌       |                         ❌                        |
|                   CREATE INDEX                   |    ✅   |           ❌          |                   ❌                  |       ✅      |        ❌       |                         ❌                        |
|                  CREATE TRIGGER                  |    ✅   |           ❌          |                   ❌                  |       ❌      |        ❌       |                         ❌                        |
| ALTER TABLE/ DROP TABLE/ TRUNCATE/ VACUUM / FULL |    ❌   |           ❌          |                   ❌                  |       ❌      |        ❌       |                         ❌                        |
 
 Two transactions can’t hold locks on conflicting modes on the same table at the same time.
 Transaction is never in conflict with itself. Non-conflicting locks can be held concurrently by many transactions.

|   Requested Lock Mode  | Current Lock Mode |           |               |                        |       |                     |           |                  |
|:----------------------:|:-----------------:|:---------:|:-------------:|:----------------------:|:-----:|:-------------------:|:---------:|:----------------:|
|                        |    ACCESS SHARE   | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE |
|      ACCESS SHARE      |                   |           |               |                        |       |                     |           |         X        |
|        ROW SHARE       |                   |           |               |                        |       |                     |     X     |         X        |
|      ROW EXCLUSIVE     |                   |           |               |                        |   X   |          X          |     X     |         X        |
| SHARE UPDATE EXCLUSIVE |                   |           |               |            X           |   X   |          X          |     X     |         X        |
|          SHARE         |                   |           |       X       |            X           |       |          X          |     X     |         X        |
|   SHARE ROW EXCLUSIVE  |                   |           |       X       |            X           |   X   |          X          |     X     |         X        |
|        EXCLUSIVE       |                   |     X     |       X       |            X           |   X   |          X          |     X     |         X        |
|    ACCESS EXCLUSIVE    |         X         |     X     |       X       |            X           |   X   |          X          |     X     |         X        |


**ROW Level Locks**

Row-level locks: exclusive or share lock. 

***EXCLUSIVE LOCK***   
* An exclusive row-level lock is automatically acquired when row is updated or deleted.
* Row-level locks don’t block data querying, they block just writes to the same row.
 * Exclusive row-level lock can be acquired explicitly without the actual changing of the row with SELECT FOR UPDATE command. 
```sql
select * from t;
    a |  b   
    ---+------
    1 |  100
    2 | 1000
    (2 rows)


T1: begin;
  BEGIN
  select * from t where a=2 for update;
        a |  b   
        ---+------
        2 | 1000
        (1 row)

                                                T2: update t set b=2000 where a=2;

Row with a=2 is under lock
table t cannot update that row(a=2) until T1 is committed or rollback.

T1: commit;
  COMMIT
                                    T2: UPDATE 1
              
```
***SHARE LOCK***
* Share row-level lock can be acquired with SELECT FOR SHARE command. 
* A shared lock does not prevent other transactions from acquiring the same shared lock. 
* However, no transaction is allowed to update, delete, or exclusively lock a row on which any other transaction holds a shared lock.

***DEAD LOCK***
* Deadlocks can occur when two transactions are waiting for each other to finish their operations.
```sql
select * from t;
    a |  b   
    ---+------
    2 | 2000
    1 |  200
    (2 rows)

T1: begin;
  BEGIN

T1: update t set b=100 where a=1;
  UPDATE 1
                        
                                                        T2: begin;
                                                            BEGIN

                                                            update t set b=1000 where a=2;
                                                            UPDATE 1
                                                                    
                                                        T2: update t set b=500 where a=1;
                                                        
T1: update t set b=5000 where a=2;

  ERROR:  deadlock detected
  DETAIL:  Process 8540 waits for ShareLock on transaction 1330; blocked by process 8547.
  Process 8547 waits for ShareLock on transaction 1329; blocked by process 8540.
  HINT:  See server log for query details.
  CONTEXT:  while updating tuple (0,14) in relation "t"

                                        T2: UPDATE 1

```  
***ADVISORY LOCKS***
* PostgreSQL provides means for creating locks that have application-defined meanings.
These are called advisory locks. 
* As the system does not enforce their use, it is up to the application to use them correctly.
* Advisory locks can be useful for locking strategies that are an awkward fit for the MVCC model.

 
## Indexes:
Indexes are special lookup tables that the database search engine can use to speed up data retrieval.
Simply put, an index is a pointer to data in a table.   
An index in a database is very similar to an index in the back of a book.
For example, if you want to reference all pages in a book that discusses a certain topic, you have to first refer to the index, which lists all topics alphabetically and then refer to one or more specific page numbers.

An index helps to speed up SELECT queries and WHERE clauses; however,  it slows down data input
 

PostgreSQL provides several index types:        
+ B-tree
+ Hash
+ GiST 
+ GIN
           
Each index type uses a different algorithm that is best suited to different types of queries. By default, the CREATE INDEX command creates B-tree indexes, which fit the most common situations.

**B-Trees**       
B Trees can handle equality and range queries on data.Using Postgres whenever an index column is involved in a comparison (<,>,<=,>=,=), creates a tree which is balanced and even.

+ If the column is unique then by default b-tree will be assigned to the column.       
+ They can operate against all datatypes, and can also be used to retrieve NULL values. 
+ B-tree indexes can also be used to retrieve data in sorted order.

```sql
---By default btree is assigned to the dep_id which is unique.

Table "public.department"

  Column  |          Type          | Modifiers 
----------+------------------------+-----------
 dep_id   | integer                | 
 dep_name | character varying(255) | 
 mang_id  | integer                | 
 loc_id   | integer                | 
Indexes:
    "depid_unique" UNIQUE CONSTRAINT, btree (dep_id)
```
Describe the table part which gives the information about the table columns,type of the column and modifiers.

```sql
\d part
Table "public.part"
Column          |      Type              |   Modifiers                     
 id             | integer                 | not null default nextval('part_id_seq'::regclass)
 partno         | character varying(20)| not null
 partname       | character varying(80)| not null
 partdescr      | text                    |
 machine_id     | integer                 | not null
Indexes:
    "part_pkey" PRIMARY KEY, btree (id)
    "part_partno_key" UNIQUE CONSTRAINT, btree (partno)
```
```sql
---insert million rows into the table using generate series..
with populate_qry as (select gs from generate_series(1,1000000) as gs )
insert into part (partno, partname,machine_id) SELECT 'PNo:'||gs, 'Part '||gs,0 from populate_qry;
INSERT 0 1000000
```

```sql
---fetching values without any index on the column name..
select * from part where partname='Part 100000';

id   |   partno   |  partname   | partdescr | machine_id 
--------+------------+-------------+-----------+-----------
 100000 | PNo:100000 | Part 100000 |           |          0
(1 row)

Time: 124.284 ms
```

```sql
explain select * from part where partname='Part 100000';
                QUERY PLAN                        
---------------------------------------------------------
 Seq Scan on part  (cost=0.00..19853.00 rows=1 width=61)
   Filter: ((partname)::text = 'Part 100000'::text)
(2 rows) 

Time: 1.297 ms
```

Fetching the result using the index on the column...

```sql
---create index on the column 
create index part_partname_idx ON part(partname);
CREATE INDEX
Time: 5183.763 ms

select * from part where partname='Part 100000';

   id   |   partno   |  partname   | partdescr | machine_id 
--------+------------+-------------+-----------+------------
 100000 | PNo:100000 | Part 100000 |           |          0
(1 row)

Time: 1.153 ms

explain  select * from part where partname='Part 100000';
               QUERY PLAN                           
-------------------------------------------------------------------------------
 Index Scan using part_partname_idx on part  (cost=0.42..8.44 rows=1 width=61)
   Index Cond: ((partname)::text = 'Part 100000'::text)
(2 rows)

Time: 1.031 ms
```
We can see the select command with index on the column is fetching results fast when compared without index..

**Hash Index**

Hash Index is only useful for equality comparison.
They are not transaction safe, need to be manually rebuilt after crashes

```sql
---Add new column into the table..

alter table part add parttype varchar(100) CHECK (parttype in ('Engine','Suspension','Driveline','Brakes','Steering','General')) NOT NULL DEFAULT 'General';
ALTER TABLE
Time: 42690,557 ms (00:42,691)

---filling the parttype with category value in the column.
with catqry as  (select id,(random()*6)::int % 6 as cat from part)
update part SET parttype = CASE WHEN cat=1 THEN 'Engine' WHEN cat=2 THEN 'Suspension' WHEN cat=3 THEN 'Driveline' WHEN cat=4 THEN 'Brakes' WHEN cat=5 THEN 'Steering' ELSE 'General' END FROM catqry WHERE part.id=catqry.id;
UPDATE 1000000

---using default scanning on the partype.

select count(*) from part where id % 500 = 0 AND parttype = 'Steering';
 count 
-------
   325
(1 row)

Time: 157.231 ms
```
Create hash index and then fetch the values from the table..

```sql
create index part_parttype_idx on part using hash(parttype);
WARNING:  hash indexes are not WAL-logged and their use is discouraged
CREATE INDEX
Time: 67455.255 ms

select count(*) from part where id % 500 = 0 AND parttype = 'Steering';
 count 
-------
   325
(1 row)

Time: 125.282 ms
```
**Generalized Inverted Indexes(GIN)**

Generalized Inverted Indexes, commonly referred to as GIN, are most useful when you have data types that contain multiple values in a single column.

“GIN is designed for handling cases where the items to be indexed are composite values, and the queries to be handled by the index need to search for element values that appear within the composite items.

For example, the items could be documents, and the queries could be searches for documents containing specific words.”

GINs are good for indexing array values,  as well as for implementing full-text search.

```sql
CREATE TABLE users (
    first_name text,
    last_name text
);
```

Insert and check the 
```sql
insert into users SELECT md5(random()::text), md5(random()::text) FROM (SELECT * FROM generate_series(1,1000000) AS id) AS x;

SELECT count(*) FROM users where first_name ilike '%aeb%';
 count 
-------
  7309
(1 row)

Time: 741.521 ms
SELECT count(*) FROM users where first_name ilike '%aeb%' or last_name ilike'%aeb%';
 count 
-------
 14653
(1 row)

Time: 1432.538 ms
```

```sql
CREATE INDEX users_search_idx ON users USING gin (first_name gin_trgm_ops, last_name gin_trgm_ops);
CREATE INDEX
Time: 28524.548 ms

SELECT count(*) FROM users where first_name ilike '%aeb%';
 count 
-------
  7309
(1 row)

Time: 38.672 ms
SELECT count(*) FROM users where first_name ilike '%aeb%' or last_name ilike'%aeb%';
 count 
-------
 14653
(1 row)

Time: 54.504 ms
```


What is gin_trgm_ops?         
This option tells Postgres to index using trigrams over our selected columns.    
A trigram is a data structure that holds 3 letters of a word.      
Essentially, Postgres will break down each text column down into trigrams and use that in the index when we search against it.



**GIST**

GiST (Generalized Search Tree) indexes are most useful when you have data that can in some way overlap with the value of that same column but from another row.

Used to index the geometric data types, as well as full-text search.

The best thing about GiST indexes: 
+ If you have say a geometry data type and you want to see if two polygons contained some point. 
+ In one case a specific point may be contained within box, while another point only exists within one polygon. 

```sql
create table task_scheduling(task_id serial primary key,task_name varchar(50) not null,
resource_name varchar(20) not null,start_date date not null,end_date date,
constraint task_scheduling_uk UNIQUE (task_name,resource_name));

CREATE TABLE
Time: 133.405 ms
```

Each task should hold only one resource at time and no other resource should be assigned to the same task in the given period for allocated resource

To make non-overlapping rows with a constraint...

```sql
create extension btree_gist ;
CREATE EXTENSION
Time: 199.615 ms

alter table task_scheduling add constraint task_scheduling_per EXCLUDE USING GIST(task_name WITH=,daterange(start_date,end_date) WITH &&);
ALTER TABLE
Time: 63.500 ms

insert into task_scheduling(task_name,resource_name,start_date,end_date) values('Print photo copies','Printer','2020-02-01','2021-02-01');
INSERT 0 1

select * from task_scheduling;
 task_id |     task_name      | resource_name | start_date |  end_date  
---------+--------------------+---------------+------------+------------
       1 | Print photo copies | Printer       | 2020-02-01 | 2021-02-01
(1 row)

Time: 0.694 ms

insert into task_scheduling(task_name,resource_name,start_date,end_date) values('Print photo copies','Xerox','2020-02-01','2020-10-01');

ERROR:  conflicting key value violates exclusion constraint "task_scheduling_per"
DETAIL:  Key (task_name, daterange(start_date, end_date))=(Print photo copies, [2020-02-01,2020-10-01)) 
conflicts with existing key (task_name, daterange(start_date, end_date))=(Print photo copies, [2020-02-01,2021-02-01)).
Time: 1.250 ms
```

As Printer resource is assigned to task from  2020-02-01 to 2021-02-01.            
No resource can be assigned to that task in the given period

```sql
insert into task_scheduling(task_name,resource_name,start_date,end_date) values('Print photo copies','Xerox','2018-02-01','2020-02-01');
INSERT 0 1
Time: 30.165 ms

insert into task_scheduling(task_name,resource_name,start_date,end_date) values('Print photo copies','Scanner','2021-02-01',null);
INSERT 0 1
Time: 45.414 ms

insert into task_scheduling(task_name,resource_name,start_date,end_date) values('Print photo copies','Wifi Printer','2023-02-01',null);
ERROR:  conflicting key value violates exclusion constraint "task_scheduling_per"

DETAIL:  Key (task_name, daterange(start_date, end_date))=(Print photo copies, [2023-02-01,)) 
conflicts with existing key (task_name, daterange(start_date, end_date))=(Print photo copies, [2021-02-01,)).
Time: 1.127 ms
```

**SP-GiST**

SP-GiST is an abbreviation for space-partitioned GiST.  
SP-GiST supports partitioned search trees, which facilitate the development of a wide range of different non-balanced data structures, such as quad-trees, k-d trees, and suffix trees (tries).

+ The common feature of these structures is that they repeatedly divide the search space into partitions that need not be of equal size.

+ Developed for in-memory usage.
+ SP-GiST can also be used in exclusion constraints.


**BRIN**    
BRIN stands for “Block Range Index”.    
A block range is a group of pages adjacent to each other, where summary information about all those pages is stored in Index.  

For example, Datatypes like integers – dates where sort order is linear – can be stored as min and max value in the range. 

Block range indexes are best when there is some natural ordering to the data, and the data tends to be very large.

If you’re querying against a large set of data that is naturally grouped together such as data for several zip codes.

BRIN helps to ensure that similar zip codes are located near each other on disk.

BRIN indexes are usually very small compared to B-Tree indexes. 

```sql
*****Without BRIN index
select count(*) from part where machine_id between 5000 and 10000; (Here it is using hash index right now)
count 
-------
  5001
(1 row)

Time: 182.702 ms

*****With BRIN Index

create index part_machine_id_idx_brin on part using brin(machine_id);
CREATE INDEX
Time: 323.403 ms
select count(*) from part where machine_id between 5000 and 10000;
 count 
-------
  5001
(1 row)

Time: 94.291 ms
```
## Partial -Functional Indexes
***

PostgreSQL partial index even allows you to specify the rows of a table that should be indexed.  
This partial index helps speed up the query while reducing the size of the index.

A partial index is useful for commonly used WHERE conditions that use constant values—like
```sql
SELECT
    customer_id,
    first_name,
    email
FROM
    customer
WHERE
    active = 0;

explain select customer_id,first_name,email from customer where active=0;
                        QUERY PLAN                         
-----------------------------------------------------------
 Seq Scan on customer  (cost=0.00..16.49 rows=15 width=42)
   Filter: (active = 0)
(2 rows)
```
You can optimize this query by creating an index for the column  active
```sql
create index customer_active_idx on customer(active);
CREATE INDEX

explain select customer_id,first_name,email from customer where active=0;
                                      QUERY PLAN                                       
---------------------------------------------------------------------------------------
 Index Scan using customer_active_idx on customer  (cost=0.28..12.30 rows=15 width=42)
   Index Cond: (active = 0)
(2 rows)
```
Here index fulfills its purpose, however, it includes many rows that are never searched, namely all the active customers (i.e active=1). 

To define an index that includes only inactive customers
```sql
create index customer_inactive_idx on customer(active) where active=0;
CREATE INDEX
```
Here WHERE clause specifies which rows should be added to the index.
```sql
explain select customer_id,first_name,email from customer where active=0;
                                      QUERY PLAN                                       
---------------------------------------------------------------------------------------
 Index Scan using customer_inactive_idx on customer  (cost=0.28..12.30 rows=15 width=42)
   Index Cond: (active = 0)
```
**FUNCTIONAL INDEXES**  
For a functional index, an index is defined on the result of a function applied to one or more columns of a single table
 
The customer table has a B-Tree index defined for the last_name column.
```sql
explain select customer_id,first_name,last_name from customer where last_name='Purdy';
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Index Scan using idx_last_name on customer  (cost=0.28..8.29 rows=1 width=17)
   Index Cond: ((last_name)::text = 'Purdy'::text)
(2 rows)

---When executing the above query, PostgreSQL uses the idx_last_name index 

explain select customer_id,first_name,last_name from customer where LOWER(last_name)='purdy';
                        QUERY PLAN                        
----------------------------------------------------------
 Seq Scan on customer  (cost=0.00..17.98 rows=3 width=17)
   Filter: (lower((last_name)::text) = 'purdy'::text)
(2 rows)
```
When executing the above query PostgreSQL could not utilize the index for lookup

To improve this query, you can define a functional index as follows
```sql
create index customer_functionalindex_lastname on customer(LOWER(last_name));
CREATE INDEX

explain select customer_id,first_name,last_name from customer where LOWER(last_name)='purdy';
                                           QUERY PLAN                                           
------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on customer  (cost=4.30..11.15 rows=3 width=17)
   Recheck Cond: (lower((last_name)::text) = 'purdy'::text)
   ->  Bitmap Index Scan on customer_functionalindex_lastname  (cost=0.00..4.30 rows=3 width=0)
         Index Cond: (lower((last_name)::text) = 'purdy'::text)
(4 rows)
```
Besides functions like UPPER/LOWER you can also index expressions like A + B and even use user-defined functions in the index definition.

Functions that cannot be “indexed” are random number generators and functions that depend on environment variables and
functions that are non-deterministic (i.e the result of the function call is not fully determined by its parameters.    
Only functions that always return the same result for the same parameters are considered to be deterministic )


## Triggers
***
A trigger is a named database object that is associated with a table, 
and it activates when a particular event (e.g. an insert, update or delete) occurs for the table/views.   

* Useful if the database is accessed by various applications and for cross-functionality            
* Used for maintaining the integrity of the information on the database. 

Trigger Types
* Row Level Trigger   
If the trigger is marked FOR EACH ROW then the trigger function will be called for each row that is getting modified by the event.

* Statement Level trigger    
 The FOR EACH STATEMENT option will call the trigger function only once for each statement, regardless of the number of the rows getting modified.  

Table for querying
```sql
select * from products;
 prod_id |     prod_name     |  price   | cid 
---------+-------------------+----------+-----
    1001 | ACER ASPIRE       | 33000.00 |   1
    1002 | DELL INSPIRON     | 43000.00 |   1
    1003 | LENOVE IDEAPAD    | 18000.00 |   1
    1004 | SAMSUNG GALAXY S9 | 35000.00 |   2
    1005 | IPHONE 7          | 60000.00 |   2
    1006 | HONOR 6X          |  9000.00 |   2
    1007 | NOKIA 1100        |  4000.00 |    
(7 rows)
```
**AFTER INSERT**
```sql
create or replace function product_audit_func() returns trigger as $$
BEGIN
insert into product_audit values(NEW.prod_id,current_timestamp);
return new;
END;
$$ language plpgsql;
CREATE FUNCTION


create TRIGGER product_trigger AFTER INSERT on products for each row execute procedure product_audit_func();
CREATE TRIGGER


insert into products(prod_name,price,cid) values('REAL ME',15000,2);
INSERT 0 1

select * from products;
 prod_id |     prod_name     |  price   | cid 
---------+-------------------+----------+-----
    1001 | ACER ASPIRE       | 33000.00 |   1
    1002 | DELL INSPIRON     | 43000.00 |   1
    1003 | LENOVE IDEAPAD    | 18000.00 |   1
    1004 | SAMSUNG GALAXY S9 | 35000.00 |   2
    1005 | IPHONE 7          | 60000.00 |   2
    1006 | HONOR 6X          |  9000.00 |   2
    1007 | NOKIA 1100        |  4000.00 |    
    1008 | REAL ME           | 15000.00 |   2
(8 rows)


select * from product_audit;
  id  |         entry_date         
------+----------------------------
 1008 | 2020-02-12 15:16:39.594311
(1 row)

alter table product_audit add trigger_name text;
ALTER TABLE

alter table product_audit add operation text;
ALTER TABLE


create or replace function product_audit_func() returns trigger as $$
BEGIN
insert into product_audit values(NEW.prod_id,current_timestamp,TG_NAME,TG_OP);
return new;
END;
$$ language plpgsql;
CREATE FUNCTION

insert into products(prod_name,price,cid) values('OPPO',25000,2);
INSERT 0 1

select * from product_audit;
  id  |        entry_date         |  trigger_name   | operation 
------+---------------------------+-----------------+-----------
 1009 | 2020-02-12 15:21:12.32357 | product_trigger | INSERT
(1 row)
```
**BEFORE INSERT OR UPDATE**
```sql
create or replace function price_validator() returns trigger as $$
BEGIN
if NEW.cid=1 and NEW.price<15000 then
raise EXCEPTION 'Price of computer should be greater than 15000';
elseif NEW.cid=2 and NEW.price<500 then
raise EXCEPTION 'Price of mobile should be > 500';
end IF;
return new;
END;
$$ language plpgsql;
CREATE FUNCTION


create trigger price_validator_trigger before insert or update on products for each row execute procedure price_validator();
CREATE TRIGGER


insert into products(prod_name,price,cid) values('MACBOOK AIR',14000,1);
ERROR:  Price of computer should be greater than 15000

insert into products(prod_name,price,cid) values('ONE PLUS 7',200,2);
ERROR:  Price of mobile should be > 500
```

**AFTER DELETE**
For DELETE operation we should use OLD 
```sql
create or replace function product_audit_del_func() returns trigger as $$
BEGIN
insert into product_audit values(OLD.prod_id,current_timestamp,TG_NAME,TG_OP);
return new;
END;
$$ language plpgsql;
CREATE FUNCTION


create trigger product_del_trigger AFTER DELETE on products for each row execute procedure product_audit_del_func();
CREATE TRIGGER


delete from products where prod_id=1014;
DELETE 1


select * from product_audit;
  id  |         entry_date         |    trigger_name     | operation 
------+----------------------------+---------------------+-----------
 1009 | 2020-02-12 15:21:12.32357  | product_trigger     | INSERT
 1014 | 2020-02-12 15:38:04.442649 | product_trigger     | INSERT
 1014 | 2020-02-12 15:50:37.742463 | product_del_trigger | DELETE
(3 rows)

```
Dropping a Trigger:     
> drop trigger trigger_name on "Table Name" ;


Some of the uses Of Triggers
1.    Auditing: You can use triggers to track the table transactions by logging the event details.

2.    Forcing Check Constraint: You can create a trigger by which you can check the constraints before applying the transaction to the table.

3.   Automatic Population: By using triggers you can also auto-populate tables fields by new transactions records manipulation.














  





 
