编译
=======
* 首先需要在本机安装mysql、git<br>

* 创建一个目录用来存放tpcc-mysql的测试代码<br>
mkdir tpcc<br>
cd tpcc<br>
git clone https://github.com/maobuji/tpcc-mysql.git<br>
cd tpcc-mysql/src<br>
make

* 如果make失败，则需要安装以下组件：<br>
yum install mysql-devel<br>
yum install gcc<br>

make成功后会在tpcc-mysql目录下生成tpcc_load和tpcc_start两个文件<br>

初始化
===

* 创建库<br>
mysqladmin -h127.0.0.1 -P3306 -uroot -proot create tpcc1000
* 创建表<br>
mysql -h127.0.0.1 -P3306 -uroot -proot -f tpcc1000 < create_table.sql
* 创建外键<br>
mysql -h127.0.0.1 -P3306 -uroot -proot -S /tmp/mysql.sock tpcc1000 < add_fkey_idx.sql

* 加载测试数据<br>
./tpcc_load -h127.0.0.1 -P3306 -uroot -proot -dtpcc1000 -w2<br>

  -h 主机IP     -P 端口号     -u 用户名      -p 密码     -d 数据库名     -w仓库数量


运行
====

* 执行数据导入<br>
./tpcc_start -h 127.0.0.1 -P 3306 -d tpcc1000 -u root -p root -w 10 -c 64 -r 30 -l 120 -f tpcclog_201409211538_64_THREADS.log >> tpcc_noaid_2_20140921_64.log 2>&1<br>

  -h 主机IP     -P 端口号     -u 用户名      -p 密码     -d 数据库名     -w仓库数量     -c 并发数     -r 预热时间     -l 持续测试时间      -i 报告生成间隔时长      -f 生成的报告名


