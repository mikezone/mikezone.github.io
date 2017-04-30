---
layout: cnblog_post
title:  "第一个Web Service的开发、发布及应用（Java）"
date:   2013-04-10 23:54:39
categories: 业余
---
一、服务的开发<br/>
(开发框架很多  以cxf为例创建项目后  导入cxf的支持jar包)<br/>
①在web项目的资源文件夹src中建立格式如org.mike.ws的包,创建服务的接口类HelloWorld.java代码如下:

```java
package org.mike.ws;
import javax.jws.WebService;

/**
 * Web服务接口，第一个Web Service规范的发布版，HelloWorld
 * 
 * @author Mike
 *
 */
@WebService
public interface HelloWorld {
    /**
     * @param name 名字
     * @return 欢迎参数指定的名字
     */
    public String sayHi(String name);   
}
```
②在src中建立如下格式如org.mike.ws.impl的包,创建服务的实现类HelloWorldImpl.java

```java
package org.mike.ws.impl;

import org.mike.ws.HelloWorld;
import javax.jws.WebService;

/**
 * Web服务接口，第一个web service规范的发布版，HelloWorld
 * 
 * @author Mike
 *
 */
@WebService(endpointInterface = "org.mike.ws.HelloWorld",
 serviceName="HelloWorld")
public class HelloWorldImpl implements HelloWorld {

    /**
     * @param name 名字
     * @return 欢迎参数指定的名字
     */
    @Override
    public String sayHi(String name) {
        return "欢迎"+name;
    }

}
```
③在org.mike.ws包中创建类WSServlet.java用于发布服务

```java
package org.mike.ws;
import javax.servlet.ServletConfig;
import javax.xml.ws.Endpoint;
import org.apache.cxf.transport.servlet.CXFNonSpringServlet;
import org.mike.ws.impl.HelloWorldImpl;
/**
 * @author Mike
 *
 */
public class WSServlet extends CXFNonSpringServlet {

    @Override
    public void loadBus(ServletConfig servletConfig){
        super.loadBus(servletConfig);
        //发布服务
        Endpoint.publish("/HelloWorld", new HelloWorldImpl());
        
    }
}
```

④修改网站的配置文件WebContent->WEB-INF->web.xml添加类说明和映射目录,添加代码如下注意添加的位置

```xml
<servlet>
    <servlet-name>WSServlet</servlet-name>
    <servlet-class>org.mike.ws.WSServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>WSServlet</servlet-name>
    <url-pattern>/ws/*</url-pattern>
    <!--上面很重要 使之应用最后映射为/ws/HelloWorld-->    
</servlet-mapping>
```
二、发布<br/>
将网站上传至服务器假设发布地址为http://127.0.0.1<br/>
此时可检验应用是否同时发布http://127.0.0.1/ws/HelloWorld?wsdl<br/>

三、客户端使用已发布的Web Service<br/>
说明：Web Service发布后是可以用任何语言访问的<br/>
本例使用Java演示<br/>
①创建Java工程wsClientTest<br/>
  导入Web Service支持类<br/>
  打开命令行界面转到本工程src目录下 输入命令wsimport -keep http://127.0.0.1/ws/<br/>HelloWorld?wsdl(该命令为jdk自带)<br/>
  src下边生成了Web Service支持类<br/>
②src下创建包test 并在包下创建类myTest.java代码如下

```java
package HelloWorldTest;
import org.mike.ws.impl.*;

public class test {
    public static void main(String[] args) {
        HelloWorld_Service factory = new HelloWorld_Service();
        HelloWorld hw=factory.getHelloWorldImplPort();
        System.out.println(hw.sayHi("Mike"));        
    }
}
```
运行后可以看到控制台输出了"欢迎Mike"