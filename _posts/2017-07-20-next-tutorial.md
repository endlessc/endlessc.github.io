# Hadoop2.7.4之集群搭建（伪分布式）
### 1. 下载Hadoop

[下载地址](http://hadoop.apache.org/releases.html)

### 2. 安装Hadoop
	tar -zxvf hadoop-2.7.4.tar.gz -C /opt/soft
### 3. 查看Hadoop是32 or 64 位
```
	cd /opt/soft/hadoop-2.7.2/lib/native
	file libhadoop.so.1.0.0
```
	 vi /etc/hosts
### 5. 配置启动Hadoop
	需要配置的文件（ore-site.xml、hdfs-site.xml、mapred-site.xml、yarn-site.xml）

	1. 修改hadoop2.7.2/etc/hadoop/hadoop-env.sh指定JAVA_HOME
		export JAVA_HOME=/usr/java/jdk1.8.0_152
	2. 修改hdfs的配置文件
		1. 修改hadoop2.7.2/etc/hadoop/core-site.xml 如下：
```
      <configuration>
				<!-- 指定HDFS(namenode)的通信地址，本次搭建用的本机IP地址 -->
				<property>
					<name>fs.defaultFS</name>
					<value>hdfs://centos7-hadoop:9000</value>
				</property>
				<!-- 指定hadoop运行时产生文件的存储路径，要和下面的[dfs.namenode.name.dir]的配置路径一致 -->
				<property>
					<name>hadoop.tmp.dir</name>
					<value>file:/usr/local/hadoop/tmp/dfs/data</value>
				</property>
			</configuration>
```
			这里fs.defaultFS的value最好是写本机的静态IP，当然写本机主机名，再配置hosts是最好的，如果用localhost，然后在windows用java操作hdfs的时候，会连接不上主机。
		2. 修改hdfs-site.xml
```
			<configuration>
			   <property>
				   <name>dfs.replication</name>
				   <value>1</value>
			   </property>
			   <property>
				   <name>dfs.namenode.name.dir</name>
				   <value>file:/usr/local/hadoop/tmp/dfs/name</value>
			   </property>
			   <property>
				   <name>dfs.datanode.data.dir</name>
				   <value>file:/usr/local/hadoop/tmp/dfs/data</value>
			   </property>
			</configuration>
```
		3. 修改mapred-site.xml(没有则复制一份mapred-site.xml.template并命名为mapred-site.xml)
```
			<configuration>  
				<property>     
					   <name>mapreduce.framework.name</name>   
					   <value>yarn</value>      
				</property>  
			</configuration>
```
		4. 修改yarn-site.xml  
```
			<configuration>  
    			<property>  
				   <name>yarn.nodemanager.aux-services</name>  
				   <value>mapreduce_shuffle</value>  
				</property>  
				<property>  
				   <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>  
				   <value>org.apache.hadoop.mapred.ShuffleHandler</value>  
				</property>  
			</configuration>  
```
### 6. 配置SSH免密码登录
```
		ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
		cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
		chmod 0600 ~/.ssh/authorized_keys
```
### 7. HDFS文件系统格式化和系统启动
	1. 下面进行HDFS文件系统进行格式化：
			$bin/hdfs namenode -format
	
	2. 然后启用NameNode及DataNode进程：
			[root@e788119a834d hadoop-2.7.4]# ./sbin/start-dfs.sh
			[root@e788119a834d hadoop-2.7.4]# ./sbin/start-yarn.sh
	
		启动进程之后用jps命令查看进程情况，出现6个进程名字说明启动成功
```
		17041 NameNode
		17361 SecondaryNameNode
		10724 ResourceManager
		17178 DataNode
		17676 NodeManager
		17903 Jps
```

### 8. 集成HBase
	1. 安装过程


### 9. 安装过程中碰到的问题
	1. 关闭防火墙

```java
	firewall-cmd --state
	systemctl stop firewalld.service

2. 9000端口拒绝访问
	配置/etc/hosts文件，添加0.0.0.0 centos7-hadoop的域名映射

3. 域名映射问题
	- 修改主机名称
		vim /etc/hostname

	- 修改域名映射
		vim /etc/hosts
		127.0.0.1   localhost
		::1         localhost.localdomain
		0.0.0.0     centos7-hadoop

4. HBase安装问题
	- zookeeper服务器连接异常

```
```
java.net.ConnectException: 拒绝连接
at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:692)
at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:350)
at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1068)

```