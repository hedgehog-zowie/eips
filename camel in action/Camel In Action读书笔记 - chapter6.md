## 第六章 camel单元测试

本章包括：

介绍和使用Camel测试套件

多种环境下的camel测试

使用mock

模拟真实组件

模拟错误

不使用mock进行测试



备注：mock测试就是在测试过程中，对于某些不容易构造或者 不容易获取的对象，用一个虚拟的对象来创建以便测试的测试方法。



对于确保应用集成是成功的，单元测试时非常重要的。JUnit已经成为单元测试的标准API，Camel测试套件就是建立在JUnit之上，利用了JUnit已有的功能。

测试Camel应用，一个好的方式是：启动camel应用，向应用发送消息，验证消息是否被正确的路由。见图6.1.向应用发送一个消息，应用把消息转换为另一种格式，并返回输出结果。你可以验证输出结果是否正确。

这就是Camel测试套件的用法，您首先需要设置期望返回的结果作为单元测试的条件，然后发送消息，接着验证结果是否正确。Mock组件就是基于这个原则运行的。



Camel测试套件简介



Camel为项目的单元测试提供了丰富的工具，一个测试套件，使你可以快速写出单元测试，就像使用JUnit API一样。使用此测试套件，可以进行camel项目的单元测试。



图6.2 把测试套件归结为三部分：JUnit扩展类：基于JUnitAPI，使Camel单元测试更容易；Mock组件和Producer template(生产者模板类)



6.1.1 Camel的JUnit扩展类

什么是Camel的JUnit扩展类？它指的是包含在camel-testjar包中的6个java类,详情见表6.1。在这六个类中，你只会经常使用其中的某几个。



6.1.2 使用Camel测试套件

我们使用下面的这个路由开启测试之旅：

from("file:inbox").to("file:outbox");



一个比较简单的解决方式是使用Camel测试套件，在下面的几节内容中，你将会使用CamelTestSupport类和CamelSpringTestSupport类进行单元测试。 

6.1.3 使用CamelTestSupport类进行单元测试 


在本章中，进行Camel单元测试，你需要引入以下依赖： 

<dependency> 

<groupId>org.apache.camel</groupId> 

<artifactId>camel-test</artifactId> 

<version>2.5.0</version> 

<scope>test</scope> 

</dependency> 

<dependency> 

<groupId>junit</groupId> 

<artifactId>junit</artifactId> 

<version>4.8.1</version> 

<scope>test</scope> 

</dependency> 


注意：Spring2.5版本只兼容低于JUnit4.4的JUnit版本；Spring3.0兼容JUnit4.4以上的版本。 


目前，你只需知道，Camel单元测试套件位于包camel-test-2.5.jar中。 


创建测试类如下： 

package camelinaction; 

import java.io.File; 

import org.apache.camel.Exchange; 

import org.apache.camel.builder.RouteBuilder; 

import org.apache.camel.test.junit4.CamelTestSupport; 

import org.junit.Test; 

public class FirstTest extends CamelTestSupport { 

@Override 

protected RouteBuilder createRouteBuilder() throws Exception { 

return new RouteBuilder() { 

@Override 

public void configure() throws Exception { 

from("file://target/inbox").to("file://target/outbox"); 

} 

}; 

} 

@Test 

public void testMoveFile() throws Exception { 

template.sendBodyAndHeader("file://target/inbox", "Hello World", 

Exchange.FILE_NAME, "hello.txt"); 

Thread.sleep(1000); 

File target = new File("target/outbox/hello.txt"); 

assertTrue("File not moved", target.exists()); 

} 

} 



FirstTest类必须继承org.apache.camel.junit4.CamelTestSupport类，才可以使用Camel测试套件。通过覆盖createRouteBuilder方法，你可以可以在单元测试类中直接创建Builder，只需覆盖configure方法。 

测试方法是标准的JUnit测试方法，使用了@Test注解。可以看到，测试方法中的代码很少，因为他没有想传统的测试一样使用java File API，而是利用camel的ProducerTemplate类创建了一个客户端，向File端点(EndPoint)发送消息， 



6.1.4 对一个已存在的RouteBuilder类进行测试 


一般情况下，camel的路由Builder会单独定义在一个类中： 

package camelinaction; 

import org.apache.camel.builder.RouteBuilder; 

public class FileMoveRoute extends RouteBuilder { 

@Override 

public void configure() throws Exception { 

from("file://target/inbox").to("file://target/outbox"); 

} 

} 


在测试类中使用方式： 

protected RouteBuilder createRouteBuilder() throws Exception { 

return new FileMoveRoute(); 

} 


6.1.5 使用SpringCamelTestSupport类进行单元测试 


SpringCamelTestSupport类使用于测试在Spring XML中创建的路由的。 

<beans xmlns="http://www.springframework.org/schema/beans" 

xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 

xsi:schemaLocation=" 
http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
http://camel.apache.org/schema/spring
http://camel.apache.org/schema/spring/camel-spring.xsd"> 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<route> 

<from uri="file://target/inbox"/> 

<to uri="file://target/outbox"/> 

</route> 

</camelContext> 

</beans> 


测试类如下； 

package camelinaction; 

import java.io.File; 

import org.apache.camel.Exchange; 

import org.apache.camel.test.junit4.CamelSpringTestSupport; 

import org.junit.Test; 

import org.springframework.context.support.AbstractXmlApplicationContext; 

import org.springframework.context.support.ClassPathXmlApplicationContext; 

public class SpringFirstTest extends CamelSpringTestSupport { 

protected AbstractXmlApplicationContext createApplicationContext() { 

return new ClassPathXmlApplicationContext( 

"camelinaction/firststep.xml"); 

} 

@Test 

public void testMoveFile() throws Exception { 

template.sendBodyAndHeader("file://target/inbox", 

"Hello World", Exchange.FILE_NAME, "hello.txt"); 

Thread.sleep(1000); 

File target = new File("target/outbox/hello.txt"); 

assertTrue("File not moved", target.exists()); 

String content = context.getTypeConverter() 

.convertTo(String.class, target); 

assertEquals("Hello World", content); 

} 

} 


6.1.6 多环境中的单元测试 


一个camel路由常常在不同的环境中进行单元测试，如在个人电脑上测试，在一个专用的测试平台上测试等等。每当到一个新的环境，我们不想重新编写测试类，这就需要我们处理容易变化的部分。 

我们将讨论两个解决方案：1、基于Camel的Properties组件；2、利用Spring的属性占位符。 

1、基于Camel的Properties组件 

Camel有一个Properties组件，支持在路由中使用外部定义的properties文件。Properties组件与Spring的属性占位符非常类似，而且改进了某些地方： 

----此组件存在于camel-core.jar包中，这意味着不需要引入第三方包就可以直接使用； 

----既可用于Spring xml中，也可用于java DSL中； 

----支持在Properties文件中使用占位符； 


更多关于Properties组件的信息，参见文档；http://camel.apache.org/properties.html


假设你想在两种环境中对上面的文件移动路由进行测试：产品环境和测试环境。使用Camel的Properties组件在Spring XML中，方式如下： 

<bean id="properties" class="org.apache.camel.component.properties.PropertiesComponent"> 

<property name="location" value="classpath:rider-prod.properties"/> 

</bean> 

在rider-prod.properties文件中，以key/value的形式定义外部属性： 

file.inbox=rider/files/inbox 

file.outbox=rider/files/outbox 


在Spring xml中定义的camelContext元素，可以在内部的endpoint URI中直接使用上面定义的外部属性： 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<route> 

<from uri="{{file.inbox}}"/> 

<to uri="{{file.outbox}}"/> 

</route> 

</camelContext> 


需要注意的是：Camel的属性占位符语法与Spring的属性占位符有一些不同。Camel的Properties组件使用{{key}}语法，而Spring使用${key}语法。 

除了在Spring XML中使用bean的形式定义Camel的Properties组件；还可以采用在camelContext中使用<propertyPlaceholder>元素的方式来定义： 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<propertyPlaceholder id="properties" 

location="classpath:rider-prod.properties"/> 

<route> 

<from uri="{{file.inbox}}"/> 

<to uri="{{file.outbox}}"/> 

</route> 

</camelContext> 


接下来，就要创建可以用于产品环境和测试环境两种环境中的测试用例：代码6.4 


2、利用Spring的属性占位符。 

----加载Properties文件； 

----使用占位符定义endpoint 

----在路由中引用endpoint 


<context:property-placeholder properties-ref="properties"/> 

<util:properties id="properties" 

location="classpath:rider-prod.properties"/> 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<endpoint id="inbox" uri="file:${file.inbox}"/> 

<endpoint id="outbox" uri="file:${file.outbox}"/> 

<route> 

<from ref="inbox"/> 

<to ref="outbox"/> 

</route> 

</camelContext> 


rider-prod.properties文件的内容： 

file.inbox=rider/files/inbox 

file.outbox=rider/files/outbox 



6.2 使用mock组件 


Mock组件是Camel单元测试的基石，是单元测试变得更简单。作为一个汽车设计师，会使用假人来做碰撞使用，以此来模拟汽车对人类的影响，Mock组件使用了相同的方式，来模拟真实的组件。 


Mock组件在许多情况下非常有用： 

---当在开发或者测试阶段，真实的组件无法获取或者根本不存在的时候； 

---当真正的组件非常慢或需要多做许多额外的努力来建立和初始化的时候，比如数据库； 

---基于测试的目的，你需要将特殊的逻辑合并到真正的组件中，而特殊的逻辑在正常情况下很少发生的时候； 

---当你需要模拟组件遇到网络问题而引起的错误的时候 


如果没有Mock组件，你只能使用真实的组件来测试，大多数情况下，是比较困难的。Camel很重视测试,Camel的第一个版本就包含了Mock组件。Mock组件位于camel-core.jar包中，而不是camel-test.jar包中，这说明Mock组件的重要程度。 


6.2.1 Mock组件介绍 

单元测试的三个基本步骤如下图：图6.3 


在测试开始前，设置期望值：应该发生什么？ 

测试开始； 

测试之后：验证输出是否符合期望值 


Camel的Mock组件使你很容易实现这些步骤。 

Mock组件可以验证一个丰富多样的期望,如以下: 

---每一个endpoint应该收到的消息个数； 

---消息到达的顺序； 

---收到的负载 

---单元测试在预期的时间内运行； 


Mock组件允许配置粗粒度和细粒度的期望和模拟网络故障等错误。



6.2.2 使用Mock组件进行单元测试 
 为了学习如何使用Mock组件，我们使用下面这个路由： 
 from("jms:topic:quote").to("mock:quote"); 
 这个路由消费来自JMS主题quote的消息，并把消息路由到一个名为quote的mock端点。 
 mock端点在Camel中的实现为：org.apache.camel.component.mock.MockEndpoint类，它提供了大量的方法来设置期望。表格6.2列出了最常用的一些方法。 

 使用MockEndpoint类进行单元测试： 
 package camelinaction; 
 import org.apache.camel.builder.RouteBuilder; 
 import org.apache.camel.component.mock.MockEndpoint; 
 import org.apache.camel.test.junit4.CamelTestSupport; 
 public class FirstMockTest extends CamelTestSupport { 
 @Override 
 protected RouteBuilder createRouteBuilder() throws Exception { 
 return new RouteBuilder() { 
 @Override 
 public void configure() throws Exception { 
 from("jms:topic:quote").to("mock:quote"); 
 } 
 }; 
 } 
 @Test 
 public void testQuote() throws Exception { 
 MockEndpoint quote = getMockEndpoint("mock:quote"); 
 quote.expectedMessageCount(1); 
 template.sendBody("jms:topic:quote", "Camel rocks"); 
 quote.assertIsSatisfied(); 
 } 
 } 

 ---使用CamelTestSupport类的getMockEndpoint方法获取MockEndpoint对象； 
 ---设置期望值 
 ---开始测试：向JMS主题发送一个消息； 
 ---使用assertIsSatisfied方法验证期望是否被满足，如果某一个期望没有被满足，Camel将会抛出异常：java.lang.AssertionError； 

 注意：assertIsSatisfied方法运行的超时时间为10秒，可以使用setResultWaitTime(long timeInMillis)方法设置超时时间。、 

 使用SEDA组件代替JMS组件 
 代码列表使用了JMS组件，但是，为了使事情变得更加简单，我们使用SEDA组件。 
 SEDA组件的文档地址：<http://camel.apache.org/seda.html.>

 可以通过注册SEDA组件来模拟JMS组件： 
 @Override 
 protected CamelContext createCamelContext() throws Exception { 
 CamelContext context = super.createCamelContext(); 
 context.addComponent("jms", context.getComponent("seda")); 
 return context; 
 } 
 覆盖createCamelContext方法，添加SEDA组件作为jms组件，这样做，你在路由中使用jms:组件的时候，实际上使用的是SEDA组件。 

 6.2.3 验证消息到达的正确性 
 expectedMessageCount方法只能验证消息到达的数量，并不能验证到达的消息的内容。要验证消息的内容，可以使用expectedBodiesReceived方法： 
 @Test 
 public void testQuote() throws Exception { 
 MockEndpoint mock = getMockEndpoint("mock:quote"); 
 mock.expectedBodiesReceived("Camel rocks"); 
 template.sendBody("jms:topic:quote", "Camel rocks"); 
 mock.assertIsSatisfied(); 
 } 

 验证多条消息按顺序到达： 
 @Test 
 public void testQuotes() throws Exception { 
 MockEndpoint mock = getMockEndpoint("mock:quote"); 
 mock.expectedBodiesReceived("Camel rocks", "Hello Camel"); 
 template.sendBody("jms:topic:quote", "Camel rocks"); 
 template.sendBody("jms:topic:quote", "Hello Camel"); 
 mock.assertIsSatisfied(); 
 } 
 验证多条消息到达(不关注顺序) 
 mock.expectedBodiesReceivedInAnyOrder("Camel rocks", "Hello Camel"); 

 如果有很多条消息到达，可以使用方法： 
 List bodies = ... //实例化期望到达的消息 
 mock.expectedBodiesReceived(bodies);//list作为参数 


 6.2.4 使用mock表达式 

 假设，你期望接收到一条消息，消息的内容中包含单词"Camel",下面是一种处理方式： 
 @Test 
 public void testIsCamelMessage() throws Exception { 
 MockEndpoint mock = getMockEndpoint("mock:quote"); 
 mock.expectedMessageCount(2); 
 template.sendBody("jms:topic:quote", "Hello Camel"); 
 template.sendBody("jms:topic:quote", "Camel rocks"); 
 assertMockEndpointsSatisfied(); 
 List<Exchange> list = mock.getReceivedExchanges(); 
 String body1 = list.get(0).getIn() 
 .getBody(String.class); 
 String body2 = list.get(1).getIn() 
 .getBody(String.class); 
 assertTrue(body1.contains("Camel")); 
 assertTrue(body2.contains("Camel")); 


 ---设置期望：收到两条消息； 
 ---发送消息； 
 ---确认满足了期望：此处使用的是assertMockEndpointsSatisfied方法(一站式方法)，而不是mock.assertIsSatisfied(); 方法； 
 ---使用getReceivedExchanges方法获取mock:quote端点收到所有exchange，进而获取消息体，验证其内容是否包含"Camel"。 
 上述代码在发送消息的前后都用到了期望值：发送消息前 mock.expectedMessageCount(2);发送消息后：assertTrue(body1.contains("Camel")); 还有一种方式，只需在一个地方声明期望。表格6.3列出了MockEndpoint类的额外的方法：在这些方法中你可以使用表达式来声明期望。可以使用messageme方法改进上面的单元测试： 
 @Test 
 public void testIsCamelMessage() throws Exception { 
 MockEndpoint mock = getMockEndpoint("mock:quote"); 
 mock.expectedMessageCount(2); 
 mock.message(0).body().contains("Camel"); 
 mock.message(1).body().contains("Camel"); 
 //mock.allMessages().body().contains("Camel"); 
 template.sendBody("jms:topic:quote", "Hello Camel"); 
 template.sendBody("jms:topic:quote", "Camel rocks"); 
 assertMockEndpointsSatisfied(); 
 } 

 如果你的期望是基于消息头的，怎么设置呢？ 
 mock.message(0).header("JMSPriority").isEqualTo(4); 

 上述代码中用到的 contains和 isEqualTo方式被称作建造方法，用于创建期望的谓词。表6.4列出了其他建造方法。 

 6.2.5 测试消息到达的顺序 
 假设你期望消息按一定的顺序到达，例如：期望消息到达的顺序是1,2,3，如果到达消息的顺序是1,3,2就是错误的，单元测试失败。 
 Mock组件提供了测试消息到达的升序、降序的功能。例如：expectsAscending方法的使用： 
 mock.expectsAscending(header("Counter")); 
 上面一句代码表达的期望是：期望收到消息的顺序是升序，用header中的Counter属性值排序。 
 当你要求收到的第一个消息的header("Counter")必须是1，你可以添加另一个期望： 
 mock.message(0).header("Counter").isEqualTo(1); 
 此时值为1, 2, 3, ...,或者1, 2, 4, 5, 6, 8, 10, ... 都会通过测试，因为expectsAscending方法并不要求这些值之间的步长值。 
 当你要求到达的消息中counter的值之间的步长值为固定值时，你需要自定义一个表达式。 

 使用自定义表达式 
 当已有的表达式和谓词无法满足你的需求时，你可以自定义表达式，此时你可以编写自己的代码来处理期望。代码列表：6.8 

 @Test 
 public void testGap() throws Exception { 
 final MockEndpoint mock = getMockEndpoint("mock:quote"); 
 mock.expectedMessageCount(3); 
 mock.expects(new Runnable() { 
 public void run() { 
 int last = 0; 
 for (Exchange exchange : mock.getExchanges()) { 
 int current = exchange.getIn() 
 .getHeader("Counter", Integer.class); 
 if (current <= last) { 
 fail("Counter is not greater than last counter"); 
 } else if (current - last != 1) { 
 fail("Gap detected: last: " + last 
 \+ " current: " + current); 
 } 
 last = current; 
 } 
 } 
 }); 
 template.sendBodyAndHeader("jms:topic:quote", "A", "Counter", 1); 
 template.sendBodyAndHeader("jms:topic:quote", "B", "Counter", 2); 
 template.sendBodyAndHeader("jms:topic:quote", "C", "Counter", 4); 
 mock.assertIsNotSatisfied(); 
 } 

 创建自定义表达式 
 ---使用expects方法，此方法接收实现Runnable接口的参数，你可以在实现Runnable接口的类中编写自定义逻辑； 
 ---在Runnable类中，你可以遍历收到的exchanges，进而获取header中的Counter属性值。 
 ---验证获取的属性值是否升序，是否步长值为1. 


 6.2.6 使用mock模拟真实的组件 
 假设有一个如下路由，在路由中你使用了HTTP服务：Jetty客户端，获取订单状态。 
 from("jetty:<http://web.rider.com/service/order>") 
 .process(new OrderQueryProcessor()) 
 .to("mina:tcp://miranda.rider.com:8123?textline=true") 
 .process(new OrderResponseProcessor()); 

 客户端发送了一个HTTP GET请求，并用订单的ID作为查询参数，到URL：<http://web.rider.com/service/order>。Camel使用OrderQueryProcessor把消息转换为骑士汽车零部件系统(名为米兰达)理解的格式。接着消息被使用TCP发送到miranda系统，接着Camel等待响应。响应消息在返回客户端之前被OrderResponseProcessor处理。 

 假设要你写一个单元测试，用于测试上述路由。挑战在于：此时你不能访问miranda系统获取订单状态，你必须模拟此系统的响应。 
 Camel提供了两种方法来模拟真实组件，见表6.5. 

 你可以使用Mock组件模拟endpoint，进而使用表6.5中的方法控制响应内容。那么，你需要Mock模拟的endpoint替换上述路由中真实的endpoint：mock：miranda。由于你准备在本地运行测试用例，你需要改成HTTP的hostname为localhost： 
 from("jetty:[http://localhost](http://localhost/):9080/service/order") 
 .process(new OrderQueryProcessor()) 
 .to("mock:miranda") 
 .process(new OrderResponseProcessor()); 

 单元测试用例如下代码列表6.9： 

 public class MirandaTest extends CamelTestSupport { 
 private String url = "[http://localhost](http://localhost/):9080/service/order?id=123"; 
 @Override 
 protected RouteBuilder createRouteBuilder() throws Exception { 
 return new RouteBuilder() { 
 @Override 
 public void configure() throws Exception { 
 from("jetty:[http://localhost](http://localhost/):9080/service/order") 
 .process(new OrderQueryProcessor()) 
 .to("mock:miranda") 
 .process(new OrderResponseProcessor()); 
 @Test 
 public void testMiranda() throws Exception { 
 MockEndpoint mock = getMockEndpoint("mock:miranda"); 
 mock.expectedBodiesReceived("ID=123"); 
 mock.whenAnyExchangeReceived(new Processor() { 
 public void process(Exchange exchange) throws Exception { 
 exchange.getIn().setBody("ID=123,STATUS=IN PROGRESS"); 
 } 
 }); 
 String out = template.requestBody(url, null, String.class); 
 assertEquals("IN PROGRESS", out); 
 assertMockEndpointsSatisfied(); 
 } 

 private class OrderQueryProcessor 
 implements Processor { 
 public void process(Exchange exchange) throws Exception { 
 String id = exchange.getIn().getHeader("id", String.class); 
 exchange.getIn().setBody("ID=" + id); 
 } 
 } 
 private class OrderResponseProcessor 
 implements Processor { 
 public void process(Exchange exchange) throws Exception { 
 String body = exchange.getIn().getBody(String.class); 
 String reply = ObjectHelper.after(body, "STATUS="); 
 exchange.getIn().setBody(reply); 
 } 
 } 
 } 

 testMiranda方法中， 
 ----获取mock:miranda endpoint， 
 ----设置期望：收到的消息体中内容为"ID=123" 
 ----使用whenAnyExchangeReceived方法返回响应，此方法允许你使用一个自定义的processor来设置响应："ID=123,STATUS=IN PROGRESS" 
 ----使用template实例的requestBody方法发送消息到 [http://localhost](http://localhost/):9080/service/order?id=123 端点(endpoint)； 
 ----设置期望：响应值等于"IN PROGRESS" 

 6.3 使用Mock组件模拟错误的发生 

 你可以通过拔掉网线来测试断网发生的错误,但这有点极端。我们将看看如何在单元测试中使用模拟错误，表6.6中列出的三种不同的技术。 


 6.3.1 使用processor模拟错误 

 对如下路由进行测试： 
 errorHandler(defaultErrorHandler() 
 .maximumRedeliveries(5).redeliveryDelay(10000)); 
 onException(IOException.class).maximumRedeliveries(3) 
 .handled(true) 
 .to("ftp://gear@ftp.rider.com?password=secret"); 
 from("file:/rider/files/upload?delay=1h") 
 .to("[http://rider.com?user=gear&password=secret](http://rider.com/?user=gear&password=secret)"); 

 当向HTTP服务发送一个文件时发生异常，你要模拟这种发生异常的情况，并且这种异常可以被onException处理并上传到ftp服务器。这样就可以测试出上述路由是否正确工作了。 

 由于此单元测试的关注点是异常，而不是具体组件的使用，所以你可以模拟HTTP和FTP endpoint。这样就不用安装一个真正的HTTP server和FTP server，上述路由变成如下形式： 

 errorHandler(defaultErrorHandler() 
 .maximumRedeliveries(5).redeliveryDelay(1000)); 
 onException(IOException.class).maximumRedeliveries(3) 
 .handled(true) 
 .to("mock:ftp"); 
 from("direct:file") 
 .to("mock:http"); 


 为了演示异常的抛出，可以向路由中添加一个processor，在processor中抛出异常： 
 from("direct:file") 
 .process(new Processor()) { 
 public void process(Exchange exchange) throws Exception { 
 throw new ConnectException("Simulated connection error"); 
 } 
 }) 
 .to("mock:http"); 

 接下来可以写一个测试方法来测试异常抛出： 
 @Test 
 public void testSimulateConnectionError() throws Exception { 
 getMockEndpoint("mock:http").expectedMessageCount(0); 
 MockEndpoint ftp = getMockEndpoint("mock:ftp"); 
 ftp.expectedBodiesReceived("Camel rocks"); 
 template.sendBody("direct:file", "Camel rocks"); 
 assertMockEndpointsIsSatisfied(); 
 } 

 ---期望收到的消息个数：0，因为路由中有异常抛出，此消息被路由到ftp服务器上。 

 6.3.2 使用mock模拟错误 

 在6.2.6节中，我们使用Mock组件模拟了一个真实的组件，除此之外，Mock组件也可以用来模拟错误。使用mock组件，就不需要更改路由(在路由中加processor)，只需直接在测试方法中模拟错误即可，与路由本身解耦： 
 @Test 
 public void testSimulateConnectionErrorUsingMock() throws Exception { 
 getMockEndpoint("mock:ftp").expectedMessageCount(1); 
 MockEndpoint http = getMockEndpoint("mock:http"); 
 http.whenAnyExchangeReceived(new Processor() { 
 public void process(Exchange exchange) throws Exception { 
 throw new ConnectException("Simulated connection error"); 
 } 
 }); 
 template.sendBody("direct:file", "Camel rocks"); 
 assertMockEndpointsSatisfied(); 
 } 

 6.3.3 使用拦截器interceptors模拟错误 

 首先我们需要了解拦截器。图6.4 channel扮演了控制器的角色，负责拦截路由中的消息。 
 拦截器允许你拦截消息，并处理消息本身。在这里，Channels扮演了一个重要的角色。Channel位于两个路由节点之间，负责处理消息、处理错误、拦截器、跟踪消息和收集消息信息。 

 Camel提供了三种开箱即用的拦截器，如图6.7。 
 在系统集成的单元测试中，你可以使用interceptSendToEndpoint类型的拦截器，拦截发送到远程的HTTP服务器的消息，重定向到一个processor，在processor中抛出异常： 
 interceptSendToEndpoint("<http://rider.com/rider>") 
 .skipSendToOriginalEndpoint(); 
 .process(new SimulateHttpErrorProcessor()); 

 ----当一个消息发送到HTTP endpoint，此消息被拦截，并被重定向到自定义的processor上，按照常理，processor之后，消息会被发送到原始的路由上，但是上述代码中使用了skipSendToOriginalEndpoint方法，所以此消息不会发送到原始的路由上。 
 可以看出，上述方法，污染了原有路由。在不污染原有路由的情况下，如何使用拦截器？答案是使用adviceWith方法 

 使用adviceWith为已有路由添加拦截器 

 adviceWith方法在单元测试期间可用,它允许您添加诸如拦截器和错误处理现有的路由。下面的代码演示了在测试方法中如何使用adviceWith方法： 
 @Test 
 public void testSimulateErrorUsingInterceptors() throws Exception { 
 RouteDefinition route = context.getRouteDefinitions().get(0); 
 route.adviceWith(context, new RouteBuilder() { 
 public void configure() throws Exception { 
 interceptSendToEndpoint("http://*") 
 .skipSendToOriginalEndpoint() 
 .process(new SimulateHttpErrorProcessor()); 
 } 
 }); 

 ---使用adviceWith方法的关键一点是：adviceWith方法要用到哪个路由上？在上面的例子中，你只有一个路由，使用getRouteDefinitions方法可以很容易得到这个路由。getRouteDefinitions方法返回的是注册到CamelContext中的所有路由列表。 
 ---得到路由后，就可以对此路由使用adviceWith方法了，使用方式：利用了一个RouterBuilder对象，在此对象的configure方法中使用java DSL定义了拦截器。注意：拦截器中使用了通配符，拦截了所有的HTTP endpoint。 
 ---如果有多个路由，你需要选择其中一个，如何选择？可以为路由设置ID，通过ID选择某个路由：context.getRouteDefinition("myCoolRouteId"). 