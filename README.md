# CaptureChangeMySQL
## 介绍

nifi 组件 CaptureChangeMySQL 的使用及bug修复
CaptureChangeMySQL：
用于mysql binlog读取，可以增量抽取mysql数据

nifi版本：1.15.2

github地址：https://github.com/dahai1996/CaptureChangeMySQL
博客地址：https://www.cnblogs.com/sqhhh/p/15842304.html
---

## 配置
见 https://blog.csdn.net/baixf/article/details/94622813
详细介绍了该组件的配置及json文件格式处理
唯一不同的是，DistributedMapCacheServer 我们选用的是redis

## bug描述
暂停该组件后，再次启动，会报错：
IOException with BIGIN event due to lingering 'inTransaction' instance variable
报错消息描述见 https://issues.apache.org/jira/browse/NIFI-6428?jql=text%20~%20%22CaptureChangeMySQL%22

## bug分析
该组件会判断当前binlog是否处于事务中，用于事务取消后，回退消息
具体实现是一个boolean变量inTransaction
当我们提交一个事务的时候，总是有一条类型为xid的消息作为事务的结尾，于是inTransaction在此处变为false，然后组件提交此处的postion到状态中
此时，暂停组件，然后重启组件
组件从状态中标记的postion开始读取，于是又读到xid消息，可是此时inTransaction为false
于是出现逻辑上的错误：
当前不在事务中，却接收到事务结束的消息，于是报错！

## bug修改
我们只需要在取到xid消息的时候，不存储当前的postion到状态，存储下一个postion即可

## 单独修改打包上传
新建项目，修改相关代码，按照nifi的打包规则完成后，上传nar包到nifi下extensions目录即可，该目录会自动加载nar包，无需重启nifi
修改后的完整代码已上传到github：https://github.com/dahai1996/CaptureChangeMySQL

