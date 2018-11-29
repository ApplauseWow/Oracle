# 实验4：对象管理

## 实验目的：
了解Oracle表和视图的概念，学习使用SQL语句Create Table创建表，学习Select语句插入，修改，删除以及查询数据，学习使用SQL语句创建视图，学习部分存储过程和触发器的使用。
## - 实验场景：
假设有一个生产某个产品的单位，单位接受网上订单进行产品的销售。通过实验模拟这个单位的部分信息：员工表，部门表，订单表，订单详单表。

## 实验内容：
## 用户名：HOLY_KWOK
### 录入数据：
要求至少有1万个订单，每个订单至少有4个详单。至少有两个部门，每个部门至少有1个员工，其中只有一个人没有领导，一个领导至少有一个下属，并且它的下属是另一个人的领导（比如A领导B，B领导C）。（注：ORDERS表和ORDER_DETAILS表是在实验三基础上依照本次实验四的建表要求做了些许修改后使用的）

- ORDERS表数据概览：
![运行结果](./万条订单数据.jpg )

- ORDER_DETAILS表数据概览：
![运行结果](./万条订单详情数据.jpg )

- 部门表DEPARTMENTS数据全览：
![运行结果](./部门数据.jpg )

- 产品表PRODUCTS数据全览：
![运行结果](./产品数据.jpg)

- 员工表EMPLOYEES数据全览：
![运行结果](./员工数据.jpg)

###  序列的应用
插入ORDERS和ORDER_DETAILS 两个表的数据时，主键ORDERS.ORDER_ID, ORDER_DETAILS.ID的值必须通过序列SEQ_ORDER_ID取得，不能手工输入一个数字。
在实验三中已经采用了序列加触发器的方式来插入ORDERS表和ORDER_DETAILS表的数据（自动插入ORDER_ID值）。
```sql
//创建序列的语句
create sequence SEQ_ID
minvalue 1
maxvalue 99999
start with 1
increment by 1
cache 20
order;
```

###  触发器的应用：
维护ORDER_DETAILS的数据时（insert,delete,update）要同步更新ORDERS表订单应收货款ORDERS.Trade_Receivable的值。
我在实验三中已经创建成功了自动插入order_id值的触发器，本次实验四给出同步更新ORDERS表订单应收货款ORDERS.Trade_Receivable的值的触发器即可。
- 第一步————先创建好触发器存储临时ORDER_ID的临时表ORDER_ID_TEMP
根据实验四.sql已创建临时表

- 第二步————创建插入order_id数据到临时表ORDER_ID_TEMP的触发器
```sql
//给order_id_temp临时表插入order_id数据的触发器：
CREATE OR REPLACE EDITIONABLE TRIGGER "ORDER_DETAILS_ROW_TRIG"
AFTER DELETE OR INSERT OR UPDATE  ON ORDER_DETAILS
FOR EACH ROW
BEGIN
  --DBMS_OUTPUT.PUT_LINE(:NEW.ORDER_ID);
  IF :NEW.ORDER_ID IS NOT NULL THEN
    MERGE INTO ORDER_ID_TEMP A
    USING (SELECT 1 FROM DUAL) B
    ON (A.ORDER_ID=:NEW.ORDER_ID)
    WHEN NOT MATCHED THEN
      INSERT (ORDER_ID) VALUES(:NEW.ORDER_ID);
  END IF;
  IF :OLD.ORDER_ID IS NOT NULL THEN
    MERGE INTO ORDER_ID_TEMP A
    USING (SELECT 1 FROM DUAL) B
    ON (A.ORDER_ID=:OLD.ORDER_ID)
    WHEN NOT MATCHED THEN
      INSERT (ORDER_ID) VALUES(:OLD.ORDER_ID);
  END IF;
END;
```

- 第三步————创建同步修改 TRADE_RECEIVABLE 值的触发器ORDER_DETAILS_SNTNS_TRIG
```sql
CREATE OR REPLACE EDITIONABLE TRIGGER "ORDER_DETAILS_SNTNS_TRIG"
AFTER DELETE OR INSERT OR UPDATE ON ORDER_DETAILS
declare
  m number(8,2);
BEGIN
  FOR R IN (SELECT ORDER_ID FROM ORDER_ID_TEMP)
  LOOP
    select sum(PRODUCT_NUM*PRODUCT_PRICE) into m from ORDER_DETAILS
      where ORDER_ID=R.ORDER_ID;
    if m is null then
      m:=0;
    end if;
    UPDATE ORDERS SET TRADE_RECEIVABLE = m - discount
      WHERE ORDER_ID=R.ORDER_ID;
  END LOOP;
  delete from ORDER_ID_TEMP;
END;
```

- 各个表创建情况：
![运行结果](./各种表.jpg)

- 触发器生效图：
![运行结果](./同步更新1.jpg )

![运行结果](./同步更新2.jpg)


###  查询数据：
1.查询某个员工的信息
- 实现查询的sql语句：
```sql
select * from employees where name='KWOK';
```

- 查询结果截图：
![运行结果](./sql1.jpg)

2.递归查询某个员工及其所有下属，子下属员工。
- 实现查询的sql语句：
```sql
select * from employees e start with e.EMPLOYEE_ID=1 connect by e.MANAGER_ID=prior e.EMPLOYEE_ID;
```

- 查询结果截图：
![运行结果](./sql2.jpg)

3.查询订单表，并且包括订单的订单应收货款:Trade_Receivable值在触发器已经实现了计算
- 实现查询的sql语句：
```sql
select * from orders;
```

- 查询结果截图：
![运行结果](./sql3.jpg )

4.查询订单详表，要求显示订单的客户名称和客户电话，产品类型用汉字描述。
- 实现查询的sql语句：
```sql
SELECT CUSTOMER_NAME, CUSTOMER_TEL, PRODUCT_TYPE FROM ORDERS o, ORDER_DETAILS d, PRODUCTS p 
WHERE o.ORDER_ID=566 and o.ORDER_ID=d.ORDER_ID and d.PRODUCT_NAME=p.PRODUCT_NAME;
```

- 查询结果截图：
![运行结果](./sql4.jpg )

5.查询出所有空订单，即没有订单详单的订单。
- 因为不存在空订单，，所以没有查询。。
- 实现查询的sql语句：无

- 查询结果截图：无

6.查询部门表，同时显示部门的负责人姓名。
- 实现查询的sql语句：
```sql
SELECT DEPARTMENT_NAME AS "部门名", NAME AS "负责人" FROM DEPARTMENTS d, EMPLOYEES e
WHERE (d.department_id= e.department_id and e.employee_id=1) or (d.department_id= e.department_id and e.employee_id=3);
```

- 查询结果截图：
![运行结果](./sql6.jpg )

7.查询部门表，统计每个部门的销售总金额。
- 实现查询的sql语句：
```sql
SELECT SUM(TRADE_RECEIVABLE), DEPARTMENT_NAME AS "部门名称" FROM ORDERS partition(PARTITION_BEFORE_2019), DEPARTMENTS d
where d.department_id=2
group by d.department_name;

	SELECT SUM(TRADE_RECEIVABLE), DEPARTMENT_NAME AS "部门名称" FROM ORDERS partition(PARTITION_BEFORE_2018), DEPARTMENTS d
where d.department_id=1
group by d.department_name;

SELECT SUM(TRADE_RECEIVABLE), DEPARTMENT_NAME AS "部门名称" FROM ORDERS partition(PARTITION_BEFORE_2017), DEPARTMENTS d
where d.department_id=1
group by d.department_name;
```

- 查询结果截图：
![运行结果](./sql7.jpg )
