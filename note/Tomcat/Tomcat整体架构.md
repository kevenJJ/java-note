# Tomcat

## 架构

tomcat 由 Server、Service、Connector、Engine、Host、Context组件组成，其中带有s的代表一个tomcat实例上可以存在多个组件，比如Context(s)，tomcat允许部署多个应用，每一个应用对应一个Context。这些组件在tomcat的 conf/server.xml 文件中可以找到，对tomcat的调优需要改动该文件。

```xml
server.xml
<Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />

    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />

    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t "%r" %s %b" />
      </Host>
    </Engine>
</Service>
```



## Server

![Server](C:\Users\Administrator\Desktop\note\img\Tomcat-Server-01.png)

Server 组件对应org.apache.catalina.Server接口，类图如上所示。

* Server 继承至 LifeCycle接口，LifeCycle是一个非常重要的接口，各大组件都继承了这个接口，用于管理tomcat的生命周期，比如init、start、stop、destory；LifeCycle 使用了观察者模式，LifeCycle 是一个监听者，它会向注册的 LifeCycleListener 观察者发出各种事件
* Server 提供了 findService、getCatalina、getCatalinaHome、getCatalinaBase 等接口，支持查找、遍历 Service 组件，这里似乎看到了和 Serivce 组件的些许联系

```java
public interface Server extends Lifecycle {

    public NamingResourcesImpl getGlobalNamingResources();

    public void setGlobalNamingResources(NamingResourcesImpl globalNamingResources);

    public javax.naming.Context getGlobalNamingContext();

    public int getPort();

    public void setPort(int port);

    public String getAddress();

    public void setAddress(String address);

    public String getShutdown();

    public void setShutdown(String shutdown);

    public ClassLoader getParentClassLoader();

    public void setParentClassLoader(ClassLoader parent);

    public Catalina getCatalina();

    public void setCatalina(Catalina catalina);

    public File getCatalinaBase();

    public void setCatalinaBase(File catalinaBase);

    public File getCatalinaHome();

    public void setCatalinaHome(File catalinaHome);

    public void await();

    public Service findService(String name);

    public Service[] findServices();

    public void removeService(Service service);

    public Object getNamingToken();
}
```

## Service

Service的默认实现类是StardardService，类结构和StardardServer很相似，也是继承至LifecycleMBeanBase，实现了Service接口，由Service接口不难发现Service组件的内部结构

* 拥有 Engine 实例
* 拥有 Server 实例
* 可以管理多个 Connector 实例
* 拥有 Excutor 引用

```java
public class StandardService extends LifecycleMBeanBase implements Service {
    // 省略若干代码
}

public interface Service extends Lifecycle {

    public Engine getContainer();

    public void setContainer(Engine engine);

    public String getName();

    public void setName(String name);

    public Server getServer();

    public void setServer(Server server);

    public ClassLoader getParentClassLoader();

    public void setParentClassLoader(ClassLoader parent);

    public String getDomain();

    public void addConnector(Connector connector);

    public Connector[] findConnectors();

    public void removeConnector(Connector connector);

    public void addExecutor(Executor ex);

    public Executor[] findExecutors();

    public Executor getExecutor(String name);

    public void removeExecutor(Executor ex);

    Mapper getMapper();
}
```

## Connector

Connector 是 tomcat 中监听 TCP 端口的组件，server.xml 默认实现了两个 Connector，分别用于监听 http、ajp 端口。对应的代码是 org.apache.catalina.connector.Connector，它是一个实现类，并且实现了 Lifecycle 接口

```xml
<code class="hljs xml has-numbering"><Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" /></code>
```

http对应的Connector配置如上所示，其中protocol用于指定http协议的版本，还可以支持2.0；connectionTimeout定义了连接超时时间，单位是毫秒；redirectPort是SSL的重定向端口，它会把请求重定向到8443这个端口 AJP

* HTTP：连接器监听8080端口，负责建立HTTP连接。在通过浏览器访问Tomcat服务器的Web应用时，使用的就是这个连接器。
* AJP：连接器监听8009端口，负责和其他的HTTP服务器建立连接。在把Tomcat与其他HTTP服务器集成时，就需要用到这个连接器。

```xml
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

Apache jserver协议(AJP)是一种二进制协议，它可以将来自web服务器的入站请求发送到位于web服务器后的应用服务器。如果我们希望把Tomcat集成到现有的(或新的)Apache http server中，并且希望Apache能够处理web应用程序中包含的静态内容，或者使用Apache的SSL处理，我们便可以使用该协议。但是，在实际的项目应用中，AJP协议并不常用，大多数应用场景会使用nginx+tomcat实现负载。

## Container

org.apache.catalina.Container接口定义了容器的api，它是一个处理用户servlet请求并返回对象给web用户的模块，它有四种不同的容器：

* Engine，表示整个 Catalina 的 servlet 引擎
* Host，表示一个拥有若干个 Context 的虚拟主机
* Context，表示一个 Web应用，一个context包含一个或多个wrapper
* Warpper，表示一个独立的servlet

![Container](C:\Users\Administrator\Desktop\note\img\Tomcat-Container.png)

Engine、Host、Context、Wrapper都有一个默认的实现类StandardXXX，均继承至ContainerBase。