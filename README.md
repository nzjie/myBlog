#随手写的博客 当做是记录吧

1.ubuntu免密码登录远程服务器
日常工作中，经常需要登录远程服务器进行工作，而每次敲那一长串命令和密码，非常耗时，虽然密码可以写在脚本里直接执行，但是密码还是每次得敲，而且公司的密码，一般都设置得很复杂，一两次还好，每次都敲，也烦也耗时；使用密码登录还有一个最大的弊端就是不安全，你每次登录，密码就会在网络中游走一次，这是非常不安全的，很多公司规定不能使用这种方式登录，所以这里跟大家分享一下使用密钥登录
机器1：个人电脑 Ubuntu14.04 LTS系统 
机器2： 阿里云服务器 ubuntu16.04 LTS系统 
步骤1：在我的机器上生成ssh密钥对 命令：ssh-keygen -t rsa 在这过程中会提示输入密码， 密码可以不输入 直接enter 接着会提示密钥对的保存路径 也可以不输入 使用默认的路径 ～/.ssh/目录（“～”是当前用户的home路径） 进入～/.ssh/， ls -l 可以看到刚才生成的密钥对，如下所示 id_rsa id_rsa.pub known_hosts 其中 id_rsa是密钥，一定要保存好，切勿泄露，id_rsa.pub是公钥，需要放到远程机器上的 
步骤2：上传公钥到远程服务器上 使用命令 scp ～/.ssh/id_rsa.pub 用户名@服务器ip:～/.ssh/authorized_keys scp是linux下跨机器复制命令，详细用法请自行google， 这个命令需要注意的是，ssh需要使用默认的端口 如果不是默认的端口，则不能使用该命令进行远程拷贝 ，上面的命令是把 本机的id_rsa.pub里面的内容赋值到远程服务器的~/.ssh/authorized_keys文件里，注意 如果authorized_keys文件本来有内容 ， 则会被覆盖 所以，如果怕被覆盖，可以使用其他名字上传 如：scp ～/.ssh/id_rsa.pub 用户名@服务器ip:～/.ssh/abcde 这样，我们的公钥就会保存在acbde的文件里，然后 使用密码登录上远程服务器，进入~/.ssh/文件夹，使用命令
cat abcde>>authorized_keys 注意，这里一定要使用>> 不能使用单个> 单个也会覆盖，两个表示追加。 到这里基本完成了，尝试使用命令登录ssh xxx@xxx.xxx.xxx.xxx -p xx 如果没有提示输入密码，那么就算是操作成功了

2.远程登录tomcat manager app
最近在阿里云服务器上搭建了tomcat，由于是第一次搭建远程tomcat，中途出现了一小插曲，再此记录一下。其实大部分步骤和本机搭建没什么两样，毕竟tomcat是一个非常成熟的夸平台web容器。首先upload tomcat压缩包到服务器，解压到指定文件夹，为bin下面的几个脚本设置运行权限，再到阿里云官网控制台开放服务器的8080端口，启动tomcat,在本机输入 http://ip:8080， 熟悉的界面如期而至；我以为就这么轻松搞定时，问题来了，当我点击manager app进入管理应用界面时，出现了403，马上意识到没有配用户名密码，vi打开{tomcatpath}/conf/tomcat-user.xml 添加<role rolename="manager-gui"/> <user username="xxx" password="xxx" roles="manager-gui" />,重启tomcat；再次访问，what the hell， 还是403禁止访问，一开始以为打开方式不对，之后强刷换 浏览器 隐身模式 从本机拷贝一份能正常操作的tomcat-user.xml覆盖服务器的 重启tomcat 重启服务器 所有的所以都试过，还是不行，当时非常纳闷，因为所有的办法都试过了，还不行，由于一开始以为问题的根源都是在tomcat-user.xml文件的配置没有搞好，所有baidu google一直往这方面搜索，都没找到能解决的答案。当我绝望的看着那403页面时，咦，好像发现了 什么，在这个页面有一行 By default the Manager is only accessible from a browser running on the same machine as Tomcat. If you wish to modify this restriction, you'll need to edit the Manager's context.xml file.大概意思是说manager默认只允许本机器访问，如果需要改变这个限制，则需要编辑context.xml文件，那么问题又来了，路径呢？？？没办法 ，搜索吧{tomcatpath} 下 find . -name context.xml 得到的结果是在{tomcatpath}/webapps/host-manager/META-INF/context.xml   {tomcatpath}/webapps/manager/META-INF/context.xml 但是META-INF一般都是放外部jar的信息用的呀，确定是这个context.xml吗？不管了，试试再说吧，试？怎么试？没有模板，怎么个配法。不纠结了，既然问题的源头都找到了，何不google一下呢，有了定向的问题，还快就搜到了答案，在{tomcatpath}/conf/Catalina/localhost下面的manager.xml(如果没有，则新建)添加
<Context privileged="true" antiResourceLocking="false"   
         docBase="${catalina.home}/webapps/manager">
             <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="^.*$" />
</Context>

好 ,执行.startup.sh 再来，当我准备松口气时，发现还是too young too simple , 这次可恶的403页面没有出现了，但是密码正确硬是说我不正确，当时我就想拿起我这三块钱的拖鞋砸烂这三百块的电脑。绝望之际，我也不知为什么，就是很突然的想查看一下tomcat的进程，ps -ef | grep tomcat ,结果如下图

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/ps_tomcat.png)

 不看不知道，一看吓一跳，怎么会有两个进程的，难不成刚才直接运行了.startup.sh 而没有先执行shutdown.sh，就起来了两个了？但是为什么不会出现端口冲突呢，又是一个纳闷现象，管不了那么多了，先kill了两个tomcat进程在说，再次启动，噗，thanks godness!!!

3.将ssh的公钥复制到远程机器时，尽然不能成功，还是要输密码，原来是我在vi打开文件后，没有进入insert模式，直接粘贴了，也奇怪，它竟然能复制成功，但是就是不可行，后来删除内容，再进入编辑模式后粘贴，保存， 成功了

ubuntu安装nginx服务器：
1.登录阿里云服务器
2.下载nginx安装文件：
 wget http://nginx.org/download/nginx-1.15.1.tar.gz
 解压：
tar -zxvf nginx-1.15.1.tar.gz
安装依赖

sudo apt-get update
sudo apt-get install openssl
sudo apt-get install libssl-dev
sudo apt-get install libpcre3-de
进入解压后的文件夹：cd nginx-1.15.1
执行：./configure --with-http_ssl_module (如果不带上--with-http_ssl_module则不支持https)
编译：make
安装: make install (如果有错误 有可能是全选问题 试试使用sudo make install执行)
安装后的文件默认放在/usr/local/nginx/下面
3.测试：
sudo ./nginx -v 显示版本
sudo ./nginx -t 测试
sudo ./nginx -s reload 重新载入配置文件
sudo ./nginx -s stop 停止
sudo ./ngxin -s reopen 重启
上面的命令中 执行reload stop reopen可能会报错：nginx: [error] invalid PID number "" in "/usr/local/nginx/logs/nginx.pid"，这时可以向nginx指定配置文件：$ sudo ./nginx -c /usr/local/etc/nginx/nginx.conf
而执行上面的命令又有可能会出错（linxu经常是这样，一个错误未解决，另一个错误又出现了，心累）
nginx: [emerg] listen() to 0.0.0.0:80, backlog 511 failed (98: Address already in use)
端口被占用了（不出意外，应该是apache2占用了80端口，找到它并kill了它就ok了）
netstat -atunp 找到80端口的进程 执行
kill -9 pid杀死进程
再执行上面的 sudo ./nginx -c /usr/local/etc/nginx/nginx.conf命令
这时试试sudo ./nginx-s reload看看是不是正常启动了
打开浏览器，在地址栏输入ip地址 如果看到Welcome to nginx!表示已经成功搭建好nginx服务器了，如果不能访问，则有可能是服务器的端口没有打开，这时需要到服务器管理后台添加安全组策略了，至于如何添加，这里就不多加描述了，可以自行百度。


 
eclipse搭建maven和tomcat项目
我们知道，在myeclipse上搭建一个web项目非常简单，因为myeclipse已经帮我们做好了大部分工作了，但是，如果在eclipse上搭建web项目，过程还是有点繁琐的，既然繁琐，为什么不直接使用myeclipse呢，当然
是有原因的，myeclipse固然是好，但是它的缺点也很明显，首先，我们要知道，myeclipse是收费的，而eclipse是免费的，这个也是myeclipse最大的缺点，所以很多企业都不会使用myeclipse，而是直接使用免费的
eclipse，其次，myeclipse集成了非常多的其他插件，很多插件我们根本就不需要用到，这就使得myeclipse显得非常臃肿，启动也非常慢。
	上面大概的说了一下我们为什么需要使用eclipse的原因，接下来进入我们的主题，开始在eclipse搭建web项目，并整合spring
注意，以下教程是在eclipse luna版本进行，不同版本的eclipse可能会有点不一样
准备材料：apache-maven-3.5.4 下载地址：http://mirrors.hust.edu.cn/apache/maven/maven-3/3.5.4/source/apache-maven-3.5.4-src.zip
apache-tomcat-7.0.53 下载地址：https://tomcat.apache.org/download-70.cgi
1. 
1.1 解压下载的maven，进入解压后的目录，找到conf文件夹，点击进入，编辑settings.xml文件，找到
<localRepository>your repository path</localRepository>
配置你的仓库路径，如果不配，则默认使用 ${user.home}/.m2/repository，其他节点可根据自己的需求进行配置

1.2 解压tomcat到特定路径

2. 打开eclipse，点击 window --> preferences -->下图

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/setting.png)

3.配置tomcat：window --> preferences --> 下图：

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/tom1.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/tom2.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/tom3.png)


4.创建maven项目：右击 --> new -->other -> maven project --> 下图

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/maven1.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/maven2.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/maven3.png)

这时候创建好的项目并不能用于web开发。
5.将项目转换成Dynamic Web Project项目：
右击项目 --> properties --> 下图：

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/dy1.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/dy2.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/dy3.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/dy4.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/dy5.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/dy6.png)

添加项目到服务器上：

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/ser1.png)
![image](https://github.com/Mitnick5194/myBlog/blob/master/images/ser2.png)
![image](https://github.com/Mitnick5194/myBlog/blob/master/images/ser3.png)
![image](https://github.com/Mitnick5194/myBlog/blob/master/images/ser4.png)

当我们看到了hello world 证明我们的项目已经跑起来了。
6.整合spring：
6.1 打开pom.xml 文件，在dependencies节点（如果没有，则创建）下面添加
<!-- spring -->
    <dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-webmvc</artifactId>
		<version>2.5.6.SEC01</version>
	</dependency>
6.2 打开web.xml文件，tomcat启动时监听spring，添加如下配置：
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/spring.xml</param-value>
	</context-param>
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
	<!-- spring MVC -->
	<servlet>
		<servlet-name>springMVC</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring-mvc-conf.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>springMVC</servlet-name>
		<!-- 拦截所有以.do结束的请求 -->
		<url-pattern>*.do</url-pattern>
	</servlet-mapping>

6.3 在WEB-INF文件夹下面创建spring.xml和spring-mvc.xml文件
spring.xml暂时不注入任何东西，只加入必要的命名空间保证不报错就好了，

<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:p="http://www.springframework.org/schema/p"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">
</beans>

spring-mvc.xml按添加如下配置：

<?xml version="1.0" encoding="UTF-8" ?>
<!-- 基础业务模块配置文件 -->
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:p="http://www.springframework.org/schema/p"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
  http://www.springframework.org/schema/context
  http://www.springframework.org/schema/context/spring-context-2.5.xsd
	http://www.springframework.org/schema/tx
	http://www.springframework.org/schema/tx/spring-tx-2.5.xsd
	http://www.springframework.org/schema/aop
	http://www.springframework.org/schema/aop/spring-aop-2.5.xsd">
	
	
	<!-- 添加前后缀 -->
	<bean id="S-IRVR"
		class="org.springframework.web.servlet.view.InternalResourceViewResolver"
		p:prefix="/" p:suffix=".jsp" />
	<!-- 规约所有进行扫描的类，使用依赖控制器类名字的惯例优先原则， 
	将URI映射到控制器 如：“/xxx/index.do”对应“com.ajie.controller.XxxController.index()” -->
	<context:component-scan base-package="com.ajie.controller" />

	<bean id="S-CCHM"
		class="org.springframework.web.servlet.mvc.support.ControllerClassNameHandlerMapping">
		<property name="caseSensitive" value="false" />
	</bean>
	<!-- 除了惯例优先原则，以下是特例的URI及控制器映射 -->
	<bean id="S-SUHM"
		class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
		<property name="mappings">
			<value>
			<!-- 添加一个测试controller -->
			/test/hello/*.do=helloController
			</value>
		</property>
	</bean>
</beans>

创建controller：

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/con1.png)

![image](https://github.com/Mitnick5194/myBlog/blob/master/images/con2.png)