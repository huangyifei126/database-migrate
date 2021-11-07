# database-migrate
数据库之间数据迁移

# 方法1：通过表空间迁移
从MySQL的Innodb特性中我们知道，Inndob的表空间有共享和独享的特点，如果是共享的。则默认会把表空间存放在一个文件中（ibdata1），当开启独享表空间参数Innodb_file_per_table时，会为每个Innodb表创建一个.ibd的文件。文章讨论在独享表空间卸载、装载、迁移Innodb表的情况。
条件：

## 2台服务器：A和B，需要A服务器上的表迁移到B服务器。

Innodb表：sysUser，记录数：351781。

以下测试在MySQL 5.5.34中进行。

### 开始处理：

1：在B服务器上建立sysUser表，并且执行：

zjy@B : db_test 09:50:30>alter table sysUser discard tablespace;
2：把A服务器表的表空间（ibd）复制到B服务器的相应数据目录。

3：修改复制过来的ibd文件权限：

chown mysql:mysql sysUser.ibd

4：最后就开始加载：

zjy@B : db_test 10:00:03>alter table sysUser import tablespace;
ERROR 1030 (HY000): Got error -1 from storage engine

### 报错了，查看错误日志：

    131112 10:05:44  InnoDB: Error: tablespace id and flags in file './db_test/sysUser.ibd' are 2428 and 0, but in the InnoDB
    InnoDB: data dictionary they are 2430 and 0.
    InnoDB: Have you moved InnoDB .ibd files around without using the
    InnoDB: commands DISCARD TABLESPACE and IMPORT TABLESPACE?
    InnoDB: Please refer to
    InnoDB: http://dev.mysql.com/doc/refman/5.5/en/innodb-troubleshooting-datadict.html
    InnoDB: for how to resolve the issue.
    131112 10:05:44  InnoDB: cannot find or open in the database directory the .ibd file of
    InnoDB: table `db_test`.`sysUser`
    InnoDB: in ALTER TABLE ... IMPORT TABLESPACE


### 当遇到这个的情况：A服务器上的表空间ID 为2428，而B服务器上的表空间ID为2430。所以导致这个错误发生，解决办法是：让他们的表空间ID一致，即：B找出表空间ID为2428的表（CREATE TABLE innodb_monitor (a INT) ENGINE=INNODB;），修改成和sysUser表结构一样的的表，再import。要不就把A服务器的表空间ID增加到大于等于B的表空间ID。（需要新建删除表来增加ID）

### 要是A的表空间ID大于B的表空间ID，则会有：


    131112 11:01:45  InnoDB: Error: tablespace id and flags in file './db_test/sysUser.ibd' are 44132 and 0, but in the InnoDB
    InnoDB: data dictionary they are 2436 and 0.
    InnoDB: Have you moved InnoDB .ibd files around without using the
    InnoDB: commands DISCARD TABLESPACE and IMPORT TABLESPACE?
    InnoDB: Please refer to
    InnoDB: http://dev.mysql.com/doc/refman/5.5/en/innodb-troubleshooting-datadict.html
    InnoDB: for how to resolve the issue.
    131112 11:01:45  InnoDB: cannot find or open in the database directory the .ibd file of
    InnoDB: table `db_test`.`sysUser`
    InnoDB: in ALTER TABLE ... IMPORT TABLESPACE

这时的情况：A服务器上的表空间ID 为44132，而B服务器上的表空间ID为2436。（因为A是测试机子，经常做还原操作，所以表空间ID已经很大了，正常情况下。表空间ID不可能这么大。

既然表空间ID不对导致这个错误报出，那我们手动的让B的表空间ID追上A的表空间ID。

需要建立的表数量：44132-2436 = 41696个，才能追上。因为他本身就需要再建立一个目标表，所以需要建立的表数量为：41695。不过安全起见，最好也不要超过41695，以防B的表空间ID超过了A，则比如设置安全的值：41690，即使B没有到达A表空间ID的值，也应该差不多了，可以再手动的去增加。用一个脚本跑（需要建立的表比较多），少的话完全可以自己手动去处理：

    #!/bin/env python
    # -*- encoding: utf-8 -*-

    import MySQLdb
    import datetime

    def create_table(conn):
        query = '''
    create table tmp_1 (id int) engine =innodb
        '''
        cursor = conn.cursor()
        cursor.execute(query)
        conn.commit()
    def drop_table(conn):
        query = '''
    drop table tmp_1
        '''
        cursor = conn.cursor()
        cursor.execute(query)
        conn.commit()

    if __name__ == '__main__':
        conn = MySQLdb.connect(host='B',user='zjy',passwd='123',db='db_test',port=3306,charset='utf8')
        for i in range(41690):
            print i
            create_table(conn)
            drop_table(conn)
也可以开启多线程去处理，加快效率。

当执行完之后，再重新按照上面的1-3步骤进行一次，最后再装载：

    zjy@B : db_test 01:39:23>alter table sysUser import tablespace;
    Query OK, 0 rows affected (0.00 sec)
要是再提示A表空间ID大于B表的话，就再手动的按照脚本里面的方法来增加ID，这时候就只需要增加个位数就可以追上A的表空间ID了。
**上面只是一个方法，虽然可以迁移Innodb，但是出问题之后可能会引其Innodb的页损坏，所以最安全的还是直接用mysqldump、xtrabackup等进行迁移**
https://www.cnblogs.com/zhoujinyi/p/3419142.html

# 方法2：通过mysqldump
## 语法格式： mysqldump -u  root  -ppassword -T 目标目录  dbname  table  [ option ];         ----注意，T是大写

password：表示root用户的密码，和 -p 挨着，中间没有空格；

目标目录：指导出的文本文件的路径；

dbname：表示数据库的名称；

table：表示表的名称；

option：表示附加选项，如下：

--fields-terminated-by=字符串：设置字符串为字段的分隔符，默认值是“\t”；

--fields-enclosed-by=字符：设置字符来括上字段的值；

--fields-optionally-enclosed-by=字符：设置字符括上char、varchar、text等字符型字段；

--fields-escaped-by=字符：设置转义字符；

--lines-terminated-by=字符串：设置每行的结束符；

------------------------------------------------------------------------------------------------------------------------------

mysqldump会导出2个文件一个txt一个sql文件(只有表结构)：

## **mysqldump -u root -p123 -T C:\Users\del\Desktop see cr01 --fields-terminated-by=',' --lines-terminated-by='\r\n'**

字段间的分隔符号 每行的结束符

![alt 图片](https://img-blog.csdnimg.cn/img_convert/5a3732fc8c7bfc20fb70363cd1463288.png)
