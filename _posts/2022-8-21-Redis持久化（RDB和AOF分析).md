Redis持久化方案
Redis有两种数据持久化方案

 - RDB持久化
 - AOF持久化
 ## 1、RDB持久化
 RDB全称Redis Database Backup file（Redis数据备份文件），也被叫做Redis数据快照。简单来说就是把内存中的所有数据都记录到磁盘中。当Redis实例故障重启后，从磁盘读取快照文件，恢复数据。快照文件称为RDB文件，默认是保存在当前运行目录。

## 1.1、执行时机

 - 执行save命令
 - 执行bgsave命令
 - Redis停机时
 - 触发RDB条件时
 
**（1）save命令**

![image](https://user-images.githubusercontent.com/91970899/186367908-05822930-fdd5-4b63-9f21-5095b3ec6286.png)
执行save命令，redis会立即执行一次RDB。save命令会导致主进程执行，因为redis执行是单线程，所以这个过程其他所有命令都会被阻塞。只有在数据迁移时可能用到。
（**2）bgsave命令**
![image](https://user-images.githubusercontent.com/91970899/186368084-693585e5-b91c-4cea-b3c0-d20aea99d13e.png)
执行这个命令会异步执行RDB。这个命令会开启独立进程（子进程）完成RDB，主进程可以继续处理用户请求，不受影响。

**（3）停机时**

Redis停机时会执行一次save命令，实现RDB持久化。

**（4）触发RDB条件**

Redis内部有触发RDB的机制，可以在redis.conf文件中找到，格式如下：

```java
# 第一条命令意思为900秒内，如果至少有1个key被修改，则执行bgsave ，（ 如果是
save ""， 则表示禁用RDB）
save 900 1  
save 300 10  
save 60 10000 
```
关于RDB的其它配置也可以在redis.conf文件中设置：

```java
# 是否压缩 ,建议不开启，压缩也会消耗cpu，磁盘的话不值钱
rdbcompression yes

# RDB文件名称 （即记录数据的文件）
dbfilename dump.rdb  

# 文件保存的路径目录
dir ./ 
```

## 1.2、RDB原理
bgsave开始时会fork主进程得到子进程，子进程共享主进程的内存数据。完成fork后读取内存数据并写入 RDB 文件。

fork采用的是copy-on-write技术：

- 当主进程执行读操作时，访问共享内存；
- 当主进程执行写操作时，则会拷贝一份数据，执行写操作。
**注：子进程复制的是父进程的页表从而通过页表去映射实际的物理内存来操作数据，而当fork的过程中，物理内存的数据是只读状态（read-only），主进程想要写操作时，对应的则是copy-on-wirte技术，即复制一份要操作的数据，在操作，后续的读同样读当前操作后的数据。**
![image](https://user-images.githubusercontent.com/91970899/186368367-f5670bf5-bb4f-4aab-ad89-206857aa3a55.png)
## 1.3、RDB总结
RDB方式bgsave的基本流程？

- fork主进程得到一个子进程，共享内存空间
- 子进程读取内存数据并写入新的RDB文件
- 用新RDB文件替换旧的RDB文件

RDB会在什么时候执行？save 60 1000代表什么含义？

- 默认是服务停止时
- 代表60秒内至少执行1000次修改则触发RDB

RDB的缺点？

- RDB执行间隔时间长，两次RDB之间写入数据有丢失的风险
- fork子进程、压缩、写出RDB文件都比较耗时

## 2、AOF持久化

## 2.1、AOF原理
AOF全称为Append Only File（追加文件）。Redis处理的每一个写命令都会记录在AOF文件，可以看做是命令日志文件。**类似于MySQL的二进制文件**
![image](https://user-images.githubusercontent.com/91970899/186368503-9ae5652e-f22b-4ba0-8e08-c43489b4b20f.png)
如上图，所有执行过的写命令会被记录，当redis再次启动时，会执行AOF文件中所保存的命令，从而实现数据的恢复。
## 2.2、AOF配置
AOF默认是关闭的，需要修改redis.conf配置文件来开启AOF：

```java
# 是否开启AOF功能，默认是no，yes为开启
appendonly yes
# AOF文件的名称（默认），可设置为其他
appendfilename "appendonly.aof"
```

AOF的命令记录的频率也可以通过redis.conf文件来配：
(即AOF对应的三种处理策略)
```java
# 表示每执行一次写命令，立即记录到AOF文件
appendfsync always 
# 写命令执行完先放入AOF缓冲区，然后表示每隔1秒将缓冲区数据写到AOF文件，是默认方案
appendfsync everysec 
# 写命令执行完先放入AOF缓冲区，由操作系统决定何时将缓冲区内容写回磁盘
appendfsync no
```
![image](https://user-images.githubusercontent.com/91970899/186368605-484ccbe3-7521-4322-b063-1245d81eaeaf.png)
## 2.3、AOF重写
因为是AOF记录命令，RDB是记录数据，AOF文件会比RDB文件大的多。而且AOF可能会记录对同一个key的多次写操作，但只有最后一次写操作才有意义（**因为redis需要保存的是最后一次操作的数据**）。通过执行bgrewriteaof命令，可以让AOF文件执行重写功能，用最少的命令达到相同效果。
![在这里插入图片描述](https://img-blog.csdnimg.cn/a9f9b719d8fa45d692fd730f04a4ee04.png#pic_center)
如上图，左边的有三条命令，但其中两条都是对num的操作，第二次的操作会覆盖第一次的操作，所以记录第一次的操作没有意义。
故重写后的命令为：`mset name jack num 666 `


除了执行命令bgrewriteaof让AOF文件重写外，**Redis也会在触发阈值时自动去重写AOF文件。阈值也可以在redis.conf中配置：**

```java
# AOF文件比上次文件 增长超过多少百分比则触发重写 （当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时）
auto-aof-rewrite-percentage 100
# AOF文件体积最小多大以上才触发重写 
auto-aof-rewrite-min-size 64mb 
```

## RDB与AOF对比
RDB和AOF各有自己的优缺点，如果对数据安全性要求较高，在实际开发中往往会**结合**两者来使用。
![image](https://user-images.githubusercontent.com/91970899/186368700-bb73f27d-faf1-4474-b3f0-75401616c043.png)
**注：文章图片皆来源黑马Redis课件**



