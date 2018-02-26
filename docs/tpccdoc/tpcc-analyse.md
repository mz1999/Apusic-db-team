TPCC-MYSQL
====
TPC-C测试用到的模型是一个大型的商品批发销售公司，它拥有若干个分布在不同区域的商品仓库。<br>
当业务扩展的时候，公司将添加新的仓库。每个仓库负责为10个销售点供货，其中每个销售点为3000个客户提供服务，每个客户提交的订单中，平均每个订单有10项产品，所有订单中约1%的产品在其直接所属的仓库中没有存货，必须由其他区域的仓库来供货。<br>
同时，每个仓库都要维护公司销售的100000种商品的库存记录。<br>

所以可以通过调整仓库数量（W）来，来调整体的数据量等级。<br>

TPCC-MYSQL是TPC-C规范的一个实现。<br>

* 标准由来<br>
TPC(Transaction Processing Performance Council，事务处理性能委员会)是由数十家会员公司创建的非盈利组织，总部设在美国。TPC的成员主要是计算机软硬件厂家，而非计算机用户，其功能是制定商务应用基准程序的标准规范、性能和价格度量，并管理测试结果的发布。


表结构与关系
===


![image](https://github.com/mz1999/Apusic-db-team/blob/master/docs/tpccdoc/img/TPCC-MYSQL-model.png)


初始化以后查看各表的数据量
====

* 表数据量与初始化仓库数量的关系

表名| 一个仓库 | 2个仓库 | 6个仓库 | 变化
------------ | --------------- | ------------ | ------------ | ------------
customer | 30000 | 60000 | 180000 | 增加
district | 10 | 20 | 60 | 增加 
history | 30000 | 60000 | 180000 | 增加
item | 100000 | 100000 | 100000
new_orders | 9000 | 18000 | 54000 | 增加
order_line | 299976 | 598908 | 1799838 | 增加
orders | 30000 | 60000 | 180000 | 增加
stock | 100000 | 200000 | 600000 | 增加
warehouse | 1 | 2 | 6 | 增加
合计 | 598987 | 1096930 | 3093904 | 增加

仓库数量与数量是线性对应的，总数据量可以近似的认为是50万*仓库数量。



* 查看表数据量的SQL<br>
select count(1),'customer' from customer<br>
UNION ALL<br>
select count(1),'district' from district<br>
UNION ALL<br>
select count(1),'history' from history<br>
UNION ALL<br>
select count(1),'item' from item<br>
UNION ALL<br>
select count(1),'new_orders' from new_orders<br>
UNION ALL<br>
select count(1),'order_line' from order_line<br>
UNION ALL<br>
select count(1),'orders' from orders<br>
UNION ALL<br>
select count(1),'stock' from stock<br>
UNION ALL<br>
select count(1),'warehouse' from warehouse<br>


执行分析
====

-c参数决定了并发连接数量，每个并发就会建立一个数据库连接，持续进行业务操作<br>

在mysql中可以使用 show processlist查看连接数量以及正在执行的SQL语句<br>
![image](https://github.com/mz1999/Apusic-db-team/blob/master/docs/tpccdoc/img/showprocesslist.png)

执行后数据量的变化情况

表名| 一个仓库 | 第一次执行后 | 第二次执行后 | 变化
------------ | --------------- | ------------ | ------------ | ------------ 
customer | 30000 | 30000 | 30000 | 
district | 10 | 10 | 10 | 
history | 30000 | 35276 | 39797 | 增加
item | 100000 | 100000 | 100000 | 
new_orders | 9000 | 8941 | 8896 | 先增加，后减少
order_line | 299976 | 352177 | 397143 | 增加
orders | 30000 | 35221 | 39696 | 增加
stock | 100000 | 100000 | 100000 |
warehouse | 1 | 1 | 1 |
合计 | 598987 | 661626 | 715543 | 增加