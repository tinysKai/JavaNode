#### 安装jdk
```
    chmod 777 jdk-7u79-linux-x64.gz
    tar zxvf jdk-7u79-linux-x64.gz
    mv jdk1.7.0_79/ /usr/local/jdk7
    
    配置环境变量
    vim /etc/profile
    export JAVA_HOME=/usr/local/jdk7
    export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    export PATH=$PATH:$JAVA_HOME/bin

    重新编译下：
    source /etc/profile

```

#### 安装tomcat
```
    chmod 755 apache-tomcat-7.0.50.tar.gz
    tar -xzf apache-tomcat-7.0.50.tar.gz
    mv apache-tomcat-7.0.50 /usr/local/tomcat7
    
    service iptables stop //关闭防火墙让win7能访问

```

#### 安装nginx
```
    安装nginx之前要安装的插件 : 
    yum -y install gcc gcc-c++ autoconf automake 
    yum -y install zlib zlib-devel openssl openssl-devel pcre pcre-devel
    
    tar -zxvf nginx-1.2.6.tar.gz 
    cd nginx-1.2.6
    ./configure
    make
    make install
    
    https://blog.csdn.net/dyllove98/article/details/8892509
```

##### 安装maven
```
tar -xvf apache-maven-3.3.3-bin.tar.gz
mv apache-maven-3.3.3 /usr/local/maven
vim /etc/profile
MAVEN_HOME=/usr/local/maven
export MAVEN_HOME
export PATH=${PATH}:${MAVEN_HOME}/bin
source /etc/profile

```
