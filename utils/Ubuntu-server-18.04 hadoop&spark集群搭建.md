1. 安装ubuntu-18.04-server，安装过程中设置username为hadoop

   ```shell
   master 192.168.194.139
   slave1 192.168.194.142
   salve2 192.168.194.143
   ```

2. 启用root账户

   ```shell
   sudo passwd root
   ```

3. 设置hadoop用户sudo免密

   ```shell
   sudo visudo
   #添加
   hadoop ALL=(ALL) NOPASSWD: ALL
   ```

4. 更换阿里源

   `sudo vim /etc/apt/sources.list`

   ```shell
   deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
   deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
   deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
   deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
   deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
   deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
   deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
   deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
   deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
   deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
   ```

   记得 `sudo apt update`

5. 安装zsh和oh-my-zsh

   ```shell
   sudo apt install zsh git
   wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh
   bash ./install.sh
   ```

   zsh配置

   * 安装Powerlevel9k 

     ```shell
     git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
     ```

   * autojump插件 `sudo apt install autojump`

   * fasd插件 `sudo apt install fasd`

   * zsh-autosuggestions 

     ```shell
     git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
     ```

   * zsh-syntax-highlighting

     ```shell
     git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
     ```

   * 最后编辑`.zshrc`

     ```shell
     # 设置字体模式以及配置命令行的主题，语句顺序不能颠倒
     POWERLEVEL9K_MODE='nerdfont-complete'
     ZSH_THEME="powerlevel9k/powerlevel9k"
     
     # 以下内容去掉注释即可生效：
     # 启动错误命令自动更正
     ENABLE_CORRECTION="true"
     
     # 在命令执行的过程中，使用小红点进行提示
     COMPLETION_WAITING_DOTS="true"
     
     # 启用已安装的插件
     plugins=(
       git extract fasd zsh-autosuggestions zsh-syntax-highlighting
     )
     ```
     注：修改后需要重启才能生效

6. 设置ssh免密登陆

   ```shell
   ubuntu server 18.04已经安装好了 openssh-server
   ssh localhost //主要是为了让其自动生成.ssh文件夹
   cd ~/.ssh
   ssh-keygen -t rsa
   cat ./id_rsa.pub >> ./authorized_keys
   
   master和slave之间互相免密登陆
   ssh-copy-id hadoop@slave1
   ssh-copy-id hadoop@slave2
   ...
   ```

7. 修改hosts

   ```shell
   sudo vim /etc/hosts
   
   192.168.194.139 master
   192.168.194.142 salve1
   192.168.194.143 slave2
   ```

8. 搭建环境

   1. 安装jdk

      ```shell
      sudo apt install openjdk-8-jdk
      vim /etc/profile
      export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
      export CLASSPATH=.:%JAVA_HOME%/lib/dt.jar:%JAVA_HOME%/lib/tools.jar
      export PATH=$PATH:$JAVA_HOME/bin
      source /etc/profile
      ```

   2. 安装hadoop

      版本：hadoop-2.9.1

      安装路径：/usr/local/hadoop

      设置环境变量：`sudo vim /etc/profile`

      ```shell
      export HADOOP_HOME=/usr/local/hadoop
      export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
      export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
      export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
      ```

      编辑 etc/hadoop/hadoop-env.sh

      ```shell
      export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
      export HADOOP_COMMON_LIB_NATIVE_DIR=${HADOOP_HOME}/lib/native
      export HADOOP_OPTS="-Djava.library.path=${HADOOP_HOME}/lib/native/"
      ```

      编辑 etc/hadoop/hdfs-site.xml

      ```xml
      <configuration>
        <property>
          <name>dfs.replication</name>
          <value>2</value>
        </property>
        <property>
          <name>dfs.namenode.name.dir</name>
          <value>file:/usr/local/hadoop/dfs/name</value>
        </property>
        <property>
          <name>dfs.datanode.data.dir</name>
          <value>file:/usr/local/hadoop/dfs/data</value>
        </property>
      </configuration>
      ```

      编辑 etc/hadoop/core-site.xml

      ```xml
      <configuration>
        <property>
          <name>fs.defaultFS</name>
          <value>hdfs://master:9000</value>
        </property>
        <property>
          <name>hadoop.tmp.dir</name>
          <value>/usr/local/hadoop/tmp</value>
        </property>
      </configuration>
      ```

      编辑 etc/hadoop/mapred-site.xml

      ```xml
      <configuration>
        <property>
          <name>mapreduce.framework.name</name>
          <value>yarn</value>
        </property>
        <property>
          <name>mapreduce.jobhistory.address</name>
          <value>Master:10020</value>
        </property>
        <property>
          <name>mapreduce.jobhistory.webapp.address</name>
          <value>Master:19888</value>
        </property>
      </configuration>
      ```

      编辑 etc/hadoop/yarn-site.xml

      ```xml
      <configuration>
          <property>
              <name>yarn.resourcemanager.hostname</name>
              <value>master</value>
          </property>
          <property>
              <name>yarn.nodemanager.aux-services</name>
              <value>mapreduce_shuffle</value>
          </property>
      </configuration>
      ```

      编辑 etc/hadoop/slaves

      ```shell
      salve1
      slave2
      ```

      将配置好的hadoop分发到slave1和slave2

      ```shell
      # 注：首先将slave的/usr/local加上写权限：sudo chmod o+w /usr/local
      scp -r /usr/local/hadoop hadoop@slave1:/usr/local/hadoop
      ```

      格式化namdenode

      ```shell
      hdfs namenode -formate
      ```

      web ui 查看

      ```shell
      http://master:9000
      ```

   3. 安装zookeeper

      版本：3.4.13

      安装路径：`/usr/local/zookeeper`

      设置环境变量：`sudo vim /etc/profile`

      ```shell
      export ZOOKEEPER_HOME=/usr/local/zookeeper
      export PATH=$PATH:$ZOOKEEPER_HOME/bin
      ```

      编辑 conf/zoo.cfg

      ```shell
      dataDir=/usr/local/zookeeper/data    
      dataLogDir=/usr/local/zookeeper/logs    
      clientPort=2181  
      server.1=192.168.194.139:2888:3888  
      server.2=192.168.194.142:2888:3888    
      server.3=192.168.194.143:2888:3888
      ```

      在dataDir目录下建一个myid文件，值与其ip对应server.myid中的myid相同

      设置环境变量：sudo vim /etc/profile

      ```shell
      export ZOOKEEPER_HOME=/usr/local/zookeeper
      export PATH=$PATH:$ZOOKEEPER_HOME/bin
      ```

      ```shell
      #启动(注意，每个节点都要启动)
      zkServer.sh start
      #查看状态
      zkServer.sh status
      #停止
      zkServer.sh stop
      #jps查看会多一个 QuorumPeerMain 进程
      ```

   4. 安装hbase

      版本：2.0.2

      安装路径：`/usr/local/hbase`

      设置环境变量：`sudo vim /etc/profile`

      ```shell
      export HBASE_HOME=/usr/local/hbase
      export PATH=$PATH:$HBASE_HOME/bin
      ```

      编辑 conf/hbase-env.sh

      ```shell
      export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
      export HBASE_MANAGES_ZK=false //用来设置是使用hbase默认自带的 Zookeeper还是使用独立的ZooKeeper。HBASE_MANAGES_ZK=false 时使用独立的，为true时使用默认自带的.
      ```

      编辑 conf/hbase-site.xml

      ```xml
      <configuration>
        <property>
          <name>hbase.rootdir</name>
          <value>hdfs://master:9000/hbase</value>
        </property>
        <property>
          <name>hbase.cluster.distributed</name>
          <value>true</value>
        </property>   
        <property>
          <name>hbase.zookeeper.quorum</name>
          <value>master,slave1,slave2</value>
        </property>
        <property>
          <name>hbase.zookeeper.property.dataDir</name>
          <value>/usr/local/zookeeper/data</value>
        </property>
      </configuration>
      ```

      编辑regionservers

      ```shell
      slave1
      slave2
      ```

      将配置好的hbase分发给slave

      ```shell
      sudo scp -r /usr/local/hbase hadoop@slave1:/usr/local
      ...
      ```

      [可选操作]

      ```shell
      sudo vim /etc/security/limits.conf
      #添加
      hadoop           -       nofile          32768
      hadoop           -       nproc           32000
      sudo vim /etc/pam.d/common-session 
      #添加
      session required pam_limits.so
      #然后logout重登使配置生效
      #[以上每个节点都要操作]
      ```

      启动hbase：`start-hbase.sh`

   5. 安装scala

      版本：2.11.12

      安装路径：`/usr/local/scala`

      设置环境变量：`sudo vim /etc/profile`

      ```shell
      export SCALA_HOME=/usr/local/scala
      export PATH=$PATH:$SCALA_HOME/bin
      ```

      注：需要在/usr/lib/jvm/java-8-openjdk-amd64下touch realease

   6. 安装spark

      版本：spark-2.3.2-bin-without-hadoop.tgz

      安装路径：`/usr/local/spark`

      设置环境变量：`sudo vim /etc/profile`

      ```shell
      export SPARK_HOME=/usr/local/spark
      export PATH=$PATH:$SPARK_HOME/bin
      ```

      编辑 conf/spark-env.sh

      ```shell
      export SPARK_DIST_CLASSPATH=$(/usr/local/hadoop/bin/hadoop classpath)
      export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
      export SPARK_MASTER_IP=master
      ```

      编辑 conf/slaves

      ```shell
      slave1
      slave2
      ```

      启动spark之前确保hdfs已启动

      ```shell
      sbin/start-all.sh
      ```

      启动之后，master会有Master进程，slave会有Worker进程
      查看webui：http://master:8080