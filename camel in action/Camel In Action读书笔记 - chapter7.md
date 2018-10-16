## 第七章 理解camel组件

本章包括：

camel组件预览

使用file、database组件

JMS组件

CXF web service

MINA组件

内存消息

自动任务：Quartz 和 Timer组件



7.1 camel组件预览



组件是Camel主要的扩展点。从Camel的第一个版本到Camel的当前版本，Camel的组件列表迅速增长，目前已有80多个组件，这些组件允许您集成不同的api、协议、数据格式等等，是你从繁重的编码工作中解放出来，Camel到达的最初的目标：是系统集成更简单。



那么Camel的组件是什么样呢？如果把Camel的路由比作高速公路，那么组件就类似公路上的坡道(上坡道、下坡道)，一个路由中的消息通过坡道旅行到另一个路由或者外部服务。图7.1 组件使用CamelContext来创建端点(endpoint)



如果只看看Camel组件的API，那么组件是非常简单的，只是一个实现Component接口的类：

public interface Component {

Endpoint createEndpoint(String uri) throws Exception;

CamelContext getCamelContext();

void setCamelContext(CamelContext context);

}

一个组件的主要任务是：端点(endpoint)的工厂。为了完成这个任务，组件接口中有CamelContext的引用。CamelContext提供了Camel常用的功能，比如注册组件、类加载、类型转换。关系图见图7.1.



把一个组件添加到运行时的camel中有两种方式：手工编写代码添加到CamelContext中；自动加载。



7.1.1 手工添加组件



CamelContext context = new DefaultCamelContext();

context.addComponent("jms",JmsComponent.jmsComponentAutoAcknowledge(connectionFactory));



上述代码中，你用JmsComponent.jmsComponentAutoAcknowledge方法创建了组件，注册到CamelContext中，注册的名字为jms。在路由中使用jms scheme可以引用到这个组件。



7.1.2 通过自动加载添加组件



自动加载的过程见图7.2



自动加载是Camle中已有组件的注册方式。为了加载新的组件，Camel会监控类路径下的META-INF/services/org/

apache/camel/component目录中的文件。此目录中的文件描述了组件的名称、组件类的全路径名。

举例：一个Bean组件，它对应META-INF/services/org/

apache/camel/component目录中的名为bean的文件，文件内容：

class=org.apache.camel.component.bean.BeanComponent

内容中的class属性告诉Camel加载类org.apache.camel.component.bean.BeanComponent作为新组件，文件的名称bean作为组件名。

Camel中的大部分组件的java代码模块都与Camel的核心模块做了分离，因为他们常常依赖第三方的包，如果不分离，将会是Camel的核心包过度膨胀。例如：Atom组件依赖Apache Abdera的Atom项目，我们可能不想在每个项目中都依赖Apache Abdera的Atom项目，所以Atom组件包含在一个独立于camel-core的模块中：camel-atom模块。



camel-core模块包含了13个非常有用的组件，见表7.2.





7.2 操作文件(File组件和FTP组件) 


在系统集中过程中，经常要做的一种集成就是与文件系统连接交互。你会发现这种情况比较奇怪,因为新系统通常提供不错的web服务和其他远程api作为集成点。集成的问题是,我们经常要处理旧的遗留系统,和基于文件的集成是常见的。例如,您可能需要读取由另一个应用程序创建的一个文件,它可能是发送一个命令,处理一个订单,数据日志记录,或其他东西。这种信息交换,如图7.3所示,在EIP称为文件传输file transfer模式。 

基于文件的集成非常普遍的另一个原因是,他们容易理解。即使是新手计算机用户也了解文件系统。尽管他们容易理解,基于文件的集成是很难做到完全正确。开发人员通常都必须处理复杂的IO api,特定于平台的文件系统问题,并发访问等。 

Camel对文件系统交互提供了广泛的支持。在本节中,我们将看看如何使用文件组件读取文件并把它们写到本地文件系统。我们还将讨论文件处理的一些高级选项，讨论如何使用FTP 组件访问远程文件。 


7.2.1 使用File组件读写文件 


使用URI的形式来配置File组件，常用的配置选项如表7.3所示，完整的配置选项参见文档：http://camel.apache.org/file2.html


Camel如何读取文件 


public void configure() { 

from("file:data/inbox?noop=true").to("stream:out"); 

} 

上述路由将从data/inbox目录中读取文件，并发文件内容打印到控制台上(使用Stream组件向System.out流发送消息)。路由中的noop=true选项告诉Camel：保留原始文件(不删除原文件)；如果去掉noop选项，将会使用Camel的默认行为：把成功消费的原始文件移到.camel目录中,这个目录名可以通过move配置选项来设置。如果想删除源文件，可以使用delete配置选项。 

默认情况，Camel将锁定正在处理的文件，知道路由处理完成。 


写文件 


public void configure() { 

from("stream:in?promptMessage=Enter something:").to("file:data/outbox"); 

} 


上述路由使用Stream组件接收控制台上的输入。 

URI stream:in将指示Camel读任何通过System.in输入到控制台上的内容,并创建一个消息。promptMessage配置选项显示一个输入提示。 

URI file:data/outbox 指示Camel把消息体写到data/outbox目录中的某个文件中，如果某文件不存在将会被创建。运行测试用例之后，你会发现生成了一个名称像f6a3a5ee-536b-43c3-8307-1b96e1ae7778的文件。这是因为你没有显示设置文件名，Camel选用了使用消息ID作为文件名。 

可以使用fileName配置选项来为URI file:data/outbox设置文件名： 

public void configure() { 

from("stream:in?promptMessage=Enter something:") 

.to("file:data/outbox?fileName=prompt.txt"); 

} 


此路由还有一个问题，默认情况下Camel会覆写prompt.txt文件的内容。在频繁写入的时候，你可以在每次创建新文件，这样就不会覆盖原内容。在Camel中实现这一点，可以在文件名中使用表达式(使用表达式语言在文件名中设置当前日期时间信息)。 

public void configure() { 

from("stream:in?promptMessage=Enter something:") 

.to("file:data/outbox?fileName=${date:now:yyyyMMdd-hh:mm:ss}.txt"); 

} 


表达式：date:now返回当前日期，你可以使用java.text.SimpleDataFormat接受的任一格式化选项。 



7.2.2 使用FTP组件读写远程文件 


存取远程服务器上的文件的常用方式是使用FTP，Camel支持三种形式的FTP： 


---Plain FTPmode transfer 

---SFTPfor secure transfer 

---FTPS (FTP Secure) for transfer with the Transport Layer Security (TLS) and 

Secure Sockets Layer (SSL) cryptographic protocols enabled 


FTP组件集成了File组件的所有特性和配置选项，并增加了许多新的选项，见表7.4，详情见官方文档：http://camel.apache.org/ftp2.html) 


因为FTP组件不是camel-core的一部分，需要添加第三方依赖包到项目中： 

<dependency> 

<groupId>org.apache.camel</groupId> 

<artifactId>camel-ftp</artifactId> 

<version>2.5.0</version> 

</dependency> 


我们使用Stream组件来演示存取远程FTP服务器上的文件： 

<route> 

<from uri="stream:in?promptMessage=Enter something:" /> 

<to uri="ftp://rider:secret@localhost:21000/data/outbox"/> 

</route> 

上述路由接收控制台上输入的内容，然后发送到ftp服务器上。 


7.3 异步消息(JMS组件) 


java消息服务(JMS)是一种非常有用的集成技术。它促进松散耦合的应用程序设计,内置支持可靠的消息传递,天生就是异步的。 

Camel没有为JMS提供provider，需要手工配置JMS provider(传入connectionFactory实例): 

<bean id="jms" class="org.apache.camel.component.jms.JmsComponent"> 

<property name="connectionFactory"> 

<bean class="org.apache.activemq.ActiveMQConnectionFactory"> 

<property name="brokerURL" value="tcp://localhost:61616" /> 

</bean> 

</property> 

</bean> 


---传入到ConnectionFactory中的tcp://localhost:61616 URI就是JMS provider。 

---本例中使用ActiveMQConnectionFactory，所以URI被ActiveMQ解析，URI告诉ActiveMQ使用TCP协议连接到broker：localhost:61616。 


默认的JMS ConnectionFactory没有使用连接池技术，所以对于每一个消息都会创建一个新的连接，连接到broker：localhost:61616。为了提高性能，ActiveMQ提供了ActiveMQ组件，此组件使用了连接池技术： 

<bean id="activemq" class="org.apache.activemq.camel.component.ActiveMQComponent"> 

<property name="brokerURL" value="tcp://localhost:61616" /> 

</bean> 


要使上述路由正常工作，需要引入activemq-camel模块： 

<dependency> 

<groupId>org.apache.activemq</groupId> 

<artifactId>activemq-camel</artifactId> 

<version>5.4.1</version> 

</dependency> 


Camel的JMS组件提供了大量的配置选项，大部分都只会用在特定的JMS只用场景。常用的配置选项见表7.5： 

为了在项目中使用Camel的JMS组件，你需要引入camel-jms模块和JMS provider的jar包到类路径下： 

<dependency> 

<groupId>org.apache.camel</groupId> 

<artifactId>camel-jms</artifactId> 

<version>2.5.0</version> 

</dependency> 


7.3.1 发送消息 接收消息 


图7.4演示了一个例子：使用发布/订阅模式，在这个模式中，accounting和production这些监听者可以订阅主题xmlOrders,新的订单会发布到主题xmlOrder上，此时，两个监听者都会收到订单消息的副本。 

在Camel中实现图7.4中的例子，你需要配置两个消费者，也就是说两个路由： 

from("jms:topic:xmlOrders").to("jms:accounting"); 

from("jms:topic:xmlOrders").to("jms:production"); 

当一个消息被发送(发布)到xmlOrders主题上，accounting 和production队列都会收到消息的副本。 


from("file:src/data?noop=true").to("jms:incomingOrders"); 

from("jms:incomingOrders") 

.choice() 

.when(header("CamelFileName").endsWith(".xml")) 

.to("jms:topic:xmlOrders") 

.when(header("CamelFileName").regex("^.*(csv|csl)$")) 

.to("jms:topic:csvOrders"); 

from("jms:topic:xmlOrders").to("jms:accounting"); 

from("jms:topic:xmlOrders").to("jms:production"); 



7.3.2 请求--响应类型的消息



我们知道，JMS消息默认是异步的。客户端发送消息到一个目的地之后，是不会等待消息的响应的。但是很多时候，客户端需要获取所发送消息的响应。JMS支持这种类型的消息，通过设置消息头部JMSReplyTo属性，消息接受者就会把响应发送到JMSReplyTo属性值上面，并且返回的消息还会包含一个JMSCorrelationID属性，在有多个消息响应时，客户端利用JMSCorrelationID来区分响应。消息流程图见图7.5。

Camel负责这种风格的消息,所以你不必创建特殊的应答队列,回复相关信息,等等。通过把消息的exchange模式改成InOut，Camel将启用JMS的请求--响应(request-reply)模式。

为了演示这种情况，让我们看看一个在骑士汽车零部件管理系统中的订单验证服务：检查订单,以确保订单列出的是实际的产品。此服务暴露为一个名为validate的队列：

from("jms:validate").bean(ValidatorBean.class);

---当调用这个服务的时候，你只需要设置消息的exchange模式为InOut，告诉Camel使用请求--响应类型的消息。可以使用exchangePattern配置选项来设置：

from("jms:incomingOrders").to("jms:validate?exchangePattern=InOut")

也可以在DSL中显示声明：

from("jms:incomingOrders").inOut().to("jms:validate")

在inOut方法中，你也可以传入端点URI参数：

from("jms:incomingOrders").inOut("jms:validate")



设置InOut模式之后，Camel发送消息到validate队列，然后用一个自动创建的临时应答队列来等待响应。当ValidatorBean返回一个消息传播回临时应答队列，之后路由继续向下进行。

除了使用临时创建的应答队列，还可以显示声明应答队列，使用JMSReplyTo头部属性或者在URI中使用replyTo配置选项。



单元测试一个请求--响应消息，可以使用ProducerTemplate类的requestBody方法：

Object result = template.requestBody("jms:incomingOrders",

"<order name=\"motor\" amount=\"1\" customer=\"honda\"/>");



7.3.3 消息映射



在进行JMS消息传递时，Camel隐藏了许多细节，这些细节一般你不需要关注。有一点需要知道：Camel将任意类型的消息体和消息头部映射为特定的JMS类型。



消息体映射



虽然Camel对消息体中包含的数据类型没有限制，但是JMS根据消息体中数据类型的不同区分了消息类型，见图7.6

当一个exchange到达类似如下路由节点to("jms:jmsDestinationName")时，exchange将会转换为5种消息类型中的一种。转换时，Camel会检查消息体的类型，进而决定转换为何种消息类型，接着创建相应的消息类型，发送到JMS目的地，表7.6显示了body类型和JMS消息类型之间的映射关系。这些换行由Camel完成。有时你可能需要保持JMS message完好无损。比如为了提高性能，不在每次都做消息映射，意味着处理消息会花费较少的时间；还有你存储的对象类型在Camel项目的类路径下不存在的时候，此时Camel进行发序列化时会报错。此时你可以实现自己的消息转换器(org.springframework.jms.support.converter.MessageConverter接口)；也可以设置URI配置选项mapJmsMessage为FALSE。





消息头映射



JMS的头部比消息体类型更加严格。在Camel中，一个header名称可以为任意java字符串；header类型可以为任意java对象。在向JMS端点发送或者接收消息时，就有一些问题了。



JMS对消息头的限制：



---消息头的名称都是以JMS开头的保留字，你不可以使用这些消息头名称；

---消息头名称必须是合法的java标示符；

---消息头的值可以是原始类型及其包装类型



为了处理这些限制，Camel做了很多事情：

1、用户所设置的以JMS开头的消息头名称在发送到JMS端点前被舍弃。Camel会试着将这些消息头名称转换为符合JMS规则的消息头名称。任意一个点号(.)字符都会被”_DOT_“替换，任意一个连接符号(-)都会被“_HYPHEN_”替换。例如，消息头名称为org.apache.camel.Test-Header将被转换为org_DOT_apache_DOT_camel_DOT_Test_HYPHEN_Header。当Camel接收到JMS发送的消息，会相反方向转换。

2、为了符合JMS规范，Camel会把非原始类型的消息头的舍弃，另外，Camel允许CharSequence, Date, BigDecimal, 和 BigInteger类型的消息头值，这些值将会转换为符合JMS规范的String类型。

对你的JMS消息传递应用程序Camel能做什么，你现在应该有一个好的理解。

JMS消息传递应用程序通常使用在一个组织内部。用户在企业防火墙外很少发送JMS消息到内部系统。在外部可以使WebService技术。



7.4 Web service(CXF组件) 


你可能很难找到不使用某种类型的web服务现代企业项目。web服务是非常有用的分布式应用程序的集成技术。Web服务常与面向服务的体系结构(SOA)关联,其中每个服务被定义为一个Web服务。 


可以把一个WebService看成网络上的一个API。这个API本身使用WSDL定义，WSDL文件中声明了可以调用的操作，输入参数、输出参数的类型等。WebService中的消息通常是格式符合简单对象访问协议(SOAP)规范的XML。此外，这些消息通过HTTP发送。见图7.7，web服务允许您编写Java代码，通过互联网使Java代码被调用。 

为了访问和发布web服务，Camel使用了Apache CXF。CXF是一个流行的webservice框架，支持大多数WebService标准。在这里，我们关注的是使用JAX-WS API开发WebService。JAX-WS定义了许多注解，使WebService工具(如CXF)把一个POJO发布为服务。 

WebService的开发有两种方式： 

---协议优先 

使用WSDL定义WebService提供的服务和参数类型--这些通常被称为WebService协议，与WebService交互，必须符合WebService协议。协议优先的意思是首先编写WSDL文件(手工编写或者使用工具)，然后根据wsdl生成java类(可使用CXF提供的工具实现这一步) 

---代码优先 

代码优先的意思是首先编写java类。然后根据java类生成WSDL文件。这是一种最简单的开发WebService的方式。 


7.4.1 配置CXF 


有两种方式来配置CXF组件URI：使用bean配置、直接使用URI配置。 


直接使用URI配置，模式如下： 

cxf://anAddress[?options] 


Address代表一个URL：http://rider.com:9999/order,options代表的是配置选项，配置选项见表7.8。 


使用bean配置 


在Spring中配置CXF endpoint bean，比URI配置方式具有更强大的功能：比如可以配置CXF拦截器、CXF总线、JAX-WS处理器等，配置形式如下： 

cxf:bean:beanName 


beanName是在Spring配置文件中定义的CXF端点的ID的名称，支持表7.8中的所有配置选项：代码列表7.2 

<beans xmlns="http://www.springframework.org/schema/beans" 

xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 

xmlns:cxf="http://camel.apache.org/schema/cxf" 

xsi:schemaLocation=" 
http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://camel.apache.org/schema/cxf
http://camel.apache.org/schema/cxf/camel-cxf.xsd"> 

<import resource="classpath:META-INF/cxf/cxf.xml"/> 

<import resource="classpath:META-INF/cxf/ 

? cxf-extension-soap.xml"/> 

<import resource="classpath:META-INF/cxf/ 

? cxf-extension-http-jetty.xml"/> 

<cxf:cxfEndpoint 

id="orderEndpoint" 

address="http://localhost:9000/order/" 

serviceClass="camelinaction.order.OrderEndpoint"/> 

</beans> 


在代码列表7.2中配置了一个CXF endpoint：orderEndpoint，你可以使用这个endpoint作为生产者或者消费者： 

cxf:bean:orderEndpoint 


在作为生产者和消费者使用的时候，两者之间有一个显著的差异。 

在一个WebService上下文中，camel中的一个生产者producer调用一个远程的WebService。这个WebService可以使用camel提供，也可以使用其他框架提供。 

在camel中调用一个WebService，使用类似java DSL中的to方法： 

...to("cxf:bean:orderEndpoint"); 


消费者consumer把整个路由公开为一个WebService。这是一个重要的概念。Camel路由可以是一个非常复杂的处理过程，拥有多个分支，多个处理节点，但是调用者只看到一个web服务(拥有输入参数和输出响应)。 


假设你有以下路由，包含多个步骤： 

from("jms:myQueue"). 

to("complex step 1"). 

... 

to("complex step N"); 


为了把上述路由公开到web上面，可以在路由开头添加一个CXF端点： 

from("cxf:bean:myCXFEndpointBean"). 

to("complex step 1"). 

... 

to("complex step N"); 

这样，当由myCXFEndpointBean配置的WebService被调用时，整个路由都会被调用。 


如果你以前使用过SOA或使用web服务,您可能会被Camel中的消费者迷惑。在web服务的世界中,消费者通常是一个客户端调用一个远程服务。在Camel中,消费者是一个服务器,所以定义发生了逆转! 


7.4.2 使用协议优先的开始方式 


这种开发方式，你首先创建一个WSDL文档，接着利用WebService工具生成java代码，过程如图7.8所示： 


为一个特定的web服务创建WSDL协议并非一项简单的任务。开始前最好确定好方法,类型和参数等内容。实例见代码列表7.3. 

如清单7.3中可以看到,一个WSDL协议比较复杂!从头编写这样的文档将会很难。通常,一个开始的好方法是使用一个向导或GUI工具。例如,在Eclipse中您可以使用File > New >其他> > WSDL Web服务向导来生成一个基于几个选项的WSDL文件框架。调整这个框架文件比从零开始容易得多。CXF框架本身也提供了类似的命令行工具。 

WSDL 1.1 与 2.0版本 


如果你使用WSDL文件之前,您可能已经注意到我们使用的版本的WSDL规范清单7.3所示。我们使用WSDL 1.1版本因为CXF的当前版本只支持1.1。这也是最常见的WSDL版本,您将看到使用。WSDL 2.0大幅改变,到目前为止很少有web服务工具(如CXF)支持它。 


选择所调用WebService的某个操作 


在消息发送到CXF 端点(endpoint)之前，可以通过设置消息的operationName头部来选择WebService中的某个操作，例如： 

<route> 

<from uri="direct:startOrder" /> 

<setHeader headerName="operationName"> 

<constant>order</constant> 

</setHeader> 

<to uri="cxf:bean:orderEndpoint"/> 

</route> 


7.4.3 代码优先的方式 

代码优先的开发方式通常被认为是一种比协议优先更容易的开发方式。代码优先开发方式中，首先编写java代码，使用JAX-WS注解，接着生成WSDL协议文件，过程如图7.9. 

使用这种开发方式，你需要弄明白你需要的方法。类型、参数，定义一个接口，代表这个web服务： 

@WebService 

public interface OrderEndpoint { 

String order(String partName, int amount, String customerName); 

} 

JAX-WS的javax.jws.WebService注解告诉CXF：这个接口是一个web服务。有许多注释允许您调整生成的WSDL,但在许多情况下默认就好。为了使用这个接口，需要配置CXF端点： 

<cxf:cxfEndpoint id="orderEndpoint" 

address="http://localhost:9000/order/" 

serviceClass="camelinaction.order.OrderEndpoint"/> 

在路由中使用： 

<from uri="cxf:bean:orderEndpoint" />



7.6 与数据库交互(JDBC和JPA组件) 


Camel提供了5个组件供访问数据库: 

1、JDBC组件---在Camel路由中访问JDBC API； 

2、SQL组件---支持在组件的URI中直接写sql语句，进行简单的查询； 

3、JPA组件---使用JPA框架映射java对象到关系数据库中； 

4、Hibernate组件---使用hibernate组件序列化java对象。由于license的不兼容，这个组件没有随camel项目一起发布。你可以在Camel扩展项目中找到：http://code.google.com/p/camel-extra

5、ibatis组件---允许您将Java对象映射到关系数据库。 


7.6.1 使用JDBC组件存取数据 

JDBC API定了java客户端如何与数据库交互。使用这个组件，需要添加camel-jdbc模块： 

<dependency> 

<groupId>org.apache.camel</groupId> 

<artifactId>camel-jdbc</artifactId> 

<version>2.5.0</version> 

</dependency> 


组件的URI配置选项见表7.10 

JDBC组件指向一个已加载的javax.sql.DataSource，URI语法： 

jdbc:dataSourceName[?options] 


URI指定后,组件就可以使用了。但您可能想知道实际的SQL语句如何指定。 

JDBC是一个动态组件，它几乎不会向目的地发送一个消息，而是把消息体的内容作为命令来执行。这样，这些命令就是SQL语句。在企业集成模式中，这一类消息被称为命令消息。因为JDBC端点接收一个命令，所以它作为消费者是没有意义的，因而这个组件不能在from语句中使用。当然，你可以使用select语句作为命令消息来提取数据，这种情况下，查询结果将被作为出栈消息添加到exchange中。 


为了演示的SQL命令消息的概念,让我们重温路由骑士汽车零部件系统中的那个订单路由。对于会计部门，当一个订单到达JMS消息队列，会计的业务应用程序不能使用这些数据，因为会计程序只从数据库中获取数据。所以，任何到达的订单需要存储到关系数据库中。使用Camel，一种解决方案如图7.13，图中关键点是使用bean把到达的消息体转成SQL语句。这是一种为JDBC组件提供命令消息的常见方式。路由如下： 

from("jms:accounting") 

.to("bean:orderToSql") 

.to("jdbc:dataSource");





这里有两个地方需要进一步的解释。1、配置的JDBC端点用于加载名为dataSource的javax.sql.DataSource类型的数据源。名为orderToSql的bean端点用于将到达的消息转换为一个sql语句。

orderToSql bean的相关代码见代码列表7.8：

public class OrderToSqlBean {

public String toSql(@XPath("order/@name") String name,

@XPath("order/@amount") int amount,

@XPath("order/@customer") String customer) {

StringBuilder sb = new StringBuilder();

sb.append("insert into incoming_orders ");

sb.append("(part_name, quantity, customer) values (");

sb.append("'").append(name).append("', ");

sb.append("'").append(amount).append("', ");

sb.append("'").append(customer).append("') ");

return sb.toString();

}

}

代码中使用XPath解析了到达的订单消息，消息体的格式类似：

<?xml version="1.0" encoding="UTF-8"?>

<order name="motor" amount="1" customer="honda"/>

转成的sql语句如下：

insert into incoming_orders (part_name, quantity, customer)

values ('motor', '1', 'honda')



接着SQL语句成为传给JDBC端点的消息的消息体。这样，你就向数据库中插入了一条新数据。此时不要期望有任何结果返回。但是，Camel会设置CamelJdbcUpdateCount(头部属性)的值为数据库更新的条数。如果在执行sql语句时报错，会抛出SQLException异常。



如果你运行了一个查询语句，Camel将以ArrayList<HashMap<String, Object>>对象的形式返回结果集。HashMap中的每一个key代表一个列名，value代表列值。Camel也会设置CamelJdbcRowCount(头部属性)的值为结果集中的记录条数。



7.6.2 使用JPA组件持久化对象

骑士汽车零部件系统又收到了新的需求：使用POJO模型代替XML格式的订单消息。



第一步要做的就是把到达的XML消息转换为等价的POJO。此外，会计部门的订单持久化路由需要更新，要能够处理POJO类型的消息体。你可以使用代码列表7.8中的方式提取XML消息中的信息，进而持久化。但是，对于持久化对象，有一个更好的解决方式。



Java持久性体系结构(JPA)是一个对象-关系映射(ORM)产品的包装层,如Hibernate,OpenJPA,TopLink等。这些ORM产品实现了java对象与关系数据库表中的数据的映射。



使用JPA组件，需要添加camel-jpa模块到项目中：

<dependency>

<groupId>org.apache.camel</groupId>

<artifactId>camel-jpa</artifactId>

<version>2.5.0</version>

</dependency>



添加ORM产品和数据库使用jar包到项目中，这个例子使用OpenJPA产品和HyperSQL数据库，所以需要添加以下依赖：

<dependency>

<groupId>org.apache.openjpa</groupId>

<artifactId>openjpa-persistence-jdbc</artifactId>

<version>1.2.2</version>

</dependency>

<dependency>

<groupId>hsqldb</groupId>

<artifactId>hsqldb</artifactId>

<version>1.8.0.7</version>

</dependency>



JPA组件的URI配置选项见表7.11：

在JPA中，需要对POJO类使用javax.persistence.Entity注解。术语entity是借用关系数据库术语，大致翻译为：面向对象编程的一个对象。也就是说如果想使用JPA持久化一个POJO类型的订单对象，就要使用这个注解。订单POJO代码列表7.9：

@Entity

public class PurchaseOrder implements Serializable {

private String name;

private double amount;

private String customer;

public PurchaseOrder() {

}

public double getAmount() {

return amount;

}

public void setAmount(double amount) {

this.amount = amount;

}

public String getName() {

return name;

}

public void setName(String name) {

this.name = name;

}

public void setCustomer(String customer) {

this.customer = customer;

}

public String getCustomer() {

return customer;

}

}



POJO对象可以使用到达的XML订单消息对象转换而成。为了测试，你可以使用producer模板发送一个新的PurchaseOrder到会计JMS队列：

PurchaseOrder purchaseOrder = new PurchaseOrder();

purchaseOrder.setName("motor");

purchaseOrder.setAmount(1);

purchaseOrder.setCustomer("honda");

template.sendBody("jms:accounting", purchaseOrder);

对应的路由如下：

from("jms:accounting").to("jpa:camelinaction.PurchaseOrder");



现在,路由已配置好,现在需要配置ORM工具。这是在Camel中使用JPA的时候必须配置的。

主要配置两个地方：把ORM工具的实体管理器连到Camel的JPA组件、配置ORM工具连接到您的数据库。为了方便演示，我们使用Apache OpenJPA这个ORM框架，但是你可以使用JPA允许的任何ORM框架。设置 OpenJPA实体管理器所需的bean，如清单7.10所示。





Spring bean配置文件清单7.10所示

---创建JpaComponent，声明所使用的实体管理器entityManagerFactory

---实体管理器创建的时候，设置persistenceUnitName为Camel，这个持久化单元定义了什么实体类应该被持久化，以及底层数据库的连接信息。在JPA,这个配置存储在类路径下META-INF目录中的persistence.xml文件中，见代码清单7.11.

---实体管理器连接上OpenJPA和HyperSQL数据库

---它还设置实体管理器,以便它可以参与事务



在代码清单7.11中，有两个地方要注意：第一，需要持久化哪些类要定义在这里；可以有多个class元素；第二如果你需要连接到其他数据库，你需要在这里改配置信息；



7.7 内存中的消息传递(Direct、SEDA和VM组件) 


Camel在核心包中提供了三个组件用于处理内存消息。对于同步消息的处理，可以使用Direct组件；对于异步消息的处理，可以使用SEDA和VM组件。SEDA和VM组件的区别在于：SEDA组件用于在CamelContext范围内进行消息传递，VM组件的范围更广一些，可以用于在JVM范围内进行消息传递。如果你的一个应用中加载了两个CamelContext，你可以使用VM组件在两个CamelContext直接发送消息。 


7.7.1 Direct组件的同步消息传递 


Direct组件是最简单的一个组件,但是它是非常有用的。Direct端点URI看起来像这样： 

direct:endpointName 

没有您可以指定或后端配置的选项，只有一个端点的名称。可是这能给你带来什么呢?Direct组件允许您对一个路由进行同步调用,或者相反,将路由开放为同步服务。例如：你有一个路由，第一个端点是Direct组件 

from("direct:startOrder") 

.to("cxf:bean:orderEndpoint"); 


向direct:startOrder发送一个消息，将会调用一个WebService(由CXF组件定义)，如下代码，使用ProducerTemplate发送一个消息到Direct组件： 

String reply = 

template.requestBody("direct:startOrder", params, String.class); 

ProducerTemplate将会在后台创建一个Producer(生产者)，发送到direct:startOrder端点。在大多数其他组件,在生产者和消费者之间发生的一些处理。例如,在一个JMS组件,消息将被发送到一个队列JMS代理(JMSbroker)。对于Direct组件，生产者直接调用消费者，直接，意味着生产者中有一个方法调用了消费者，这种简单性和最小的开销让Direct组件成为开启路由或者同步分解路由成多个碎片路由的一个好方法。但是,其同步的特性并不适合所有的应用程序。如果需要异步操作,你需要SEDA或VM组件。 


7.7.2 使用SEDA和VM进行异步消息传递 


当你看到早些时候在本章中讨论JMS组件的时候(7.3节),利用消息队列作为发送消息的一种手段有许多好处。您还了解了如何将一个路由应用程序被分解成许多的逻辑块(碎片路由)，并使用JMS队列作为桥梁连接起来。但为此在一个主机上使用JMS应用程序为一些用例添加了不必要的复杂性。 

如果你想获得异步消息传递的好处而不关心JMS规范一致性或JMS提供的内置的可靠性,你可能要考虑一个内存消息的解决方案。抛弃规范一致性和任何通信和消息代理(可以是昂贵的),使用内存消息可能是更快的解决方案。请注意,此时没有消息持久化到磁盘,像JMS那样,所以在应用崩溃的时候有丢失消息的风险，你的应用程序应该能够接受这种风险。 

Camel提供了两种内存中的队列组件:SEDA和 VM。他们都在表7.12中列出了配置选项。 

SEDA组件在Camel应用中的最常见应用是连接过个路由组成一个路由应用程序。例如,复习7.3.1中给出的示例,您使用的JMS主题发送副本传入会计和生产部门。在这种情况下,您使用JMS队列把路由连接到了一起，因为会计队列或者生产部门队列不在同一个主机上。如果在同一个主机上，可以有一种更高效的解决方案，见图7.14： 


在同一个CamelContext中任何JMS消息传递都可以使用SEDA来切换。你仍然需要为会计和生产使用JMS队列,因为它们是物理上独立的部门。从图中可以看出SEDA队列替换了JMS的xmlOrders主题。为了使SEDA队列更像一个JMS主题(使用发布/订阅消息模型)，你需要设置URI的配置选项multipleConsumers为true。代码见代码列表7.12。 

这个例子和7.3.1节中的示例一样,除了它使用SEDA端点,而不是JMS。另一个重要的细节在这个清单中的SEDA端点的uri,在消费者和生产者的场景中重用需要彼此完全相同。 


7.8 自动化的任务(Timer和Quartz组件) 

通常在企业项目需要安排任务在指定的时间或定期发生。Camel的Timer和Quartz组件支持这种服务。Timer组件是用于简单重复的任务,但当你需要更多的控制时,就必须使用Quartz组件。 


7.8.1 Timer组件 


TImer组件在Camel的核心包中，使用了JRE中内置的timer机制，定期产生消息。这个组件只支持消费,因为发送一个计时器并没有真正意义。URI配置选项见表格7.3 


作为一个例子,让我们实现一个功能，每隔2秒向控制台打印一条消息。路由如下： 

from("timer://myTimer?period=2000") 

.setBody().simple("Current time is ${header.firedTime}") 

.to("stream:out"); 


URI配置使用java.util.Timer创建了一个名为myTimerand的定时器，每两秒执行一次。 

当定时器触发时，Camel会创建一个exchange，消息体为空，发送到路由上。在这种情况下，你使用表达式设置了消息体。头部的firedTime属性值由Timer组件设置，更多的组件信息参见文档：http://camel.apache.org/timer.html


注意：定时器对象和相应的线程可以共享。如果你在另一条路由中使用相同的计时器的名字,它将重用相同的定时器对象。 



7.8.2 企业级调度(Quartz组件) 


除了具备Timer组件已有的功能，Quartz组件可以让你拥有更多控制。你也可以利用Quartz组件的许多其他企业特性。相关特性见网站 http://www.quartz-scheduler.org/ 常用的URI配置选项见表7.14。 


使用Quartz组件，需要引入第三方包： 

<dependency> 

<groupId>org.apache.camel</groupId> 

<artifactId>camel-quartz</artifactId> 

<version>2.5.0</version> 

</dependency>