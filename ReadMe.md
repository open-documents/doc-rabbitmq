

Server
又称作Broker，用于接受客户端的连接，实现AMQP实体服务；
Connection
 连接，应用程序与Broker的网络连接；
Channel
 网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道。客户端可建立多个Channel，每个Channel代表一个会话任务；
Message
  消息，服务器和应用程序之间传送的数据，有Properties和Body组成。Properties可以对消息进行修饰，比如消息的优先级、延迟等高级特性；Body则是消息体内容，即我们要传输的数据；
       仅仅创建了客户端到Broker之间的连接后，客户端还是不能发送消息的。需要为每一个Connection创建Channel，AMQP协议规定只有通过Channel才能执行AMQP的命令。一个Connection可以包含多个Channel。之所以需要Channel，是因为TCP连接的建立和释放都是十分昂贵的，如果一个客户端每一个线程都需要与Broker交互，如果每一个线程都建立一个TCP连接，暂且不考虑TCP连接是否浪费，就算操作系统也无法承受每秒建立如此多的TCP连接。RabbitMQ建议客户端线程之间不要共用Channel，至少要保证共用Channel的线程发送消息必须是串行的，但是建议尽量共用Connection。
Virtual Host
      虚拟地址，是一个逻辑概念，用于进行逻辑隔离，是最上层的消息路由。一个Virtual Host里面可以有若干个Exchange和Queue，同一个Virtual Host里面不能有相同名称的Exchange或者Queue；
       Virtual Host是权限控制的最小粒度；
Exchange
交换机，用于接收消息，可根据路由键将消息转发到绑定的队列；
Binding:
  Exchange和Queue之间的虚拟连接，Exchange在与多个Message Queue发生Binding后会生成一张路由表，路由表中存储着Message Queue所需消息的限制条件即Binding Key。当Exchange收到Message时会解析其Header得到Routing Key，Exchange根据Routing Key与Exchange Type将Message路由到Message Queue。Binding Key由Consumer在Binding Exchange与Message Queue时指定，而Routing Key由Producer发送Message时指定，两者的匹配方式由Exchange Type决定
Routing Key:
 一个路由规则，虚拟机可用它来确定如何路由一个特定的消息；
Queue:
也称作Message Queue，即消息队列，用于保存消息并将他们转发给消费者；





-- -- 


以安装RabbitMQ3.11.x为例。

参考文档：
- RabbitMQ和Erlang/OTP版本对应关系：https://www.rabbitmq.com/which-erlang.html
- Erlang/OTP源码：


# 1 安装Erlang/OTP

因为RabbitMQ是用Erlang语言编写，因此需要准备Erlang的运行时（根据版本对应关系，此处选择 Erlang/OTP 25.0.3）。

Erlang有3种不同类型的包：
- erlang-base：里面只包含了必要OTP组件,只有大约13M
- erlang：包括所有OTP组件及OTP Suite，没有erlang-doc、erlang-manpages、erlang-mode
- esl-erlang：里面包含所有的OTP组件，比erlang package体积大一些

有2种安装方式：rpm包安装和源码编译安装。

## rpm包安装

优点：
- 不用配置环境变量
缺点：
- 一般官网提供rpm包会滞后于源码包，因此可能官方没有对应版本的二进制rpm包。
- 提供的一般都是esl-erlang rpm包。

rpm包网址：`https://www.erlang-solutions.com/downloads/`

1）添加Erlang Solutions key。
```bash
rpm --import https://packages.erlang-solutions.com/rpm/erlang_solutions.asc
```
2）`/etc/yum.repos.d/` 下新建一个.repo文件，并手动添加下面的内容。
```bash
[erlang-solutions]
name=CentOS $releasever - $basearch - Erlang Solutions
baseurl=https://packages.erlang-solutions.com/rpm/centos/$releasever/$basearch
gpgcheck=1
gpgkey=https://packages.erlang-solutions.com/rpm/erlang_solutions.asc
enabled=1
```
3）erlang rpm包需要一些依赖包，而上述的源中没有提供，因此还需要提供EPEL（Extra Packages for Enterprise Linux，较CentOS源提供更新的包）仓库。
<font color="00E0FF">EPEL官网</font>：`https://docs.fedoraproject.org/en-US/epel/`
```bash
yum install epel-release
```
4.1）安装erlang（2选1）
```bash
sudo yum install erlang
```
4.2）安装esl-erlang（2选1）
```bash
sudo yum install esl-erlang
```
5）验证安装成功。
```bash
erl -noshell -eval '{ok, Version} = file:read_file(filename:join([code:root_dir(), "releases", erlang:system_info(otp_release), "OTP_VERSION"])), io:fwrite(Version), halt().'

# 24.3.4.1 可以看到这样安装比官网提供的rpm版本还低
```
6）卸载erlang。
```bash
yum -y remove erlang-*
rm -rf /usr/lib64/erlang
```



## 源码编译安装

优点：
- 可以安装任何erlang版本。
- 可以自定义安装。
缺点：
- 需要配置环境变量
- 不会自动生成systemctl脚本

1）下载Erlang/OTP源码。

Erlang/OTP源码下载网址：
- `https://www.erlang.org`
- `https://github.com/erlang/otp`

2）安装依赖
- 解压工具（如GNU `unzip`）和理解GNU TAR格式的 `tar` 程序。
- 构建工具。包括GNU `make`、编译器（如GNU C编译器 `gcc`）、Perl5（`perl`）、所需的头文件和库文件（如 `kernel-devel`、`ncurses-devel`、`glibc-devel` ）、`sed`（流式编辑器用于基本的文本转换）。
- 安装程序。
- 可选功能：OpenSSL、JDK（用于构建jinterface应用）、flex、wxWidgets、xsltproc、fop。
```bash
yum install make gcc gcc-c++ build-essential openssl openssl-devel unixODBC unixODBC-devel kernel-devel m4 ncurses-devel
```
3）进入Erlang/OTP源码目录。
4）生成makefile文件。
```bash
./configure --prefix=/usr/local/erlang # erlang将会被安装到的目录
```
5）编译和安装。
```bash
make && make install
```
6）配置环境变量
```bash
# file: /etc/profile
# set Erlang Env 
export ERLANG_HOME=/usr/local/erlang/bin # 前面指定的erlang被安装到的目录
export PATH=${ERLANG_HOME}/bin:$PATH
```
7）重启，并验证安装是否成功。
```bash
erl -noshell -eval '{ok, Version} = file:read_file(filename:join([code:root_dir(), "releases", erlang:system_info(otp_release), "OTP_VERSION"])), io:fwrite(Version), halt().'

# 25.0.4
```

# 2 安装RabbitMQ

RabbitMQ git网址：https://github.com/rabbitmq/rabbitmq-server/releases

1）下载二进制文件。（后缀为tar.xz）
2）解压和解包。
```bash
xz -d ...
tar -xvf ...
```
3）设置环境变量。
```bash
# Set RabbitMQ Env
RABBITMQ_VERSION=3.11.19
RABBITMQ_HOME=/home/xcxiao/opt/rabbitmq_server-$RABBITMQ_VERSION  # 安装目录
export PATH=RABBITMQ_HOME/sbin:$PATH
```

# 3 开启web管理组件

1）开启web管理组件。（需要先关闭RabbitMQ服务）
```bash
rabbitmq-plugins enable rabbitmq_management
```
2）创建用户。
```text
# 创建账号和密码
rabbitmqctl add_user xcxiao xcxwrka1314

# 设置用户角色：administrator
rabbitmqctl set_user_tags xcxiao administrator

# 为用户添加资源权限，添加配置、写、读权限
rabbitmqctl set_permissions -p "/" xcxiao ".*" ".*" ".*" ".*"

```
# 4 修改并开放端口


```text
# 该端口用于AMQP协议
listeners.tcp.default = 7216
# 该端口用于前端管理
management.tcp.port = 7217
```

开放接口
```bash
firewall-cmd --zone=public --add-port=7216/tcp --permanent && firewall-cmd --reload
firewall-cmd --zone=public --add-port=7217/tcp --permanent && firewall-cmd --reload
```