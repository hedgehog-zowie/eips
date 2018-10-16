## 第九章 使用事务

本章包括 

1、为什么使用事务 

2、如何使用和配置事务 

3、本地事务和全局事务的区别 

4、事务回滚时如何返回自定义消息 

5、不支持事务时如何补偿 


为了更好理解什么是事务，让我们看一个真实生活中的例子。你很有可能从Manning的网上书店订购了这本书，如果你这样做了，你可能进行了以下步骤： 

1、找到这本书《Camel in Action》 

2、将这本书放入购物车 

3、可能继续购买其他书籍 

4、去结账 

5、输入物流和信用卡信息 

6、确认购买 

7、等待确认 

8、离开网店 

看起来像一个普通的日常场景，实际上是相当复杂的一系列事件。你结账之前必须将书籍放入购物车；你确认购买之前必须输入物流和信用卡信息；如果你的信用卡被拒绝，那么购买将不会被确认；等等。这笔交易的最终决议是两种状态之一：此次购买被确认，或者购买被拒绝，不扣除信用卡的任何费用。 

这个故事涉及计算机系统,因为它使用了一个在线书店。但同样的要点发生在超市购物过程中。 

通常在软件世界中,事务的解释都是基于SQL语句操作数据库表的上下文中——更新或插入数据。事务正在进行的过程中,可能会发生系统故障,导致事务的参与者处于不一致的状态。这就是为什么一系列的事件被描述为原子性：要么都完成，要么都失败----全有或全无。在事务方面,他们要么提交要么回滚。 

注意：我希望你知道数据库ACID属性，所以我不会解释原子性、一致性、隔离和持久性在事务上下文中的意思。如果你不熟悉ACID属性，维基百科页面是一个学习它的好地方： http://en.wikipedia.org/wiki/ACID.在这一章,我们首先看看您应该使用事务的原因(骑士汽车零部件系统的上下文中)。然后我们将更详细地查看事务，以及在Spring的事务管理中是如何管理事务的。您将了解本地和全局事务之间的区别,以及如何配置和使用事务。 


9.1为什么使用事务? 


有很多好的理由来说明使用事务是有意义的。　但在我们关注Camel事务之前,让我们看看你不使用事务时所出现的错误， 

在本节中,我们将回顾骑士汽车零部件系统，此系统用来收集指标,并发布到一个事件管理系统。我们将会看到应用程序不使用事务时出什么错，然后我们将启用事务。 


9.1.1 骑手汽车零部件合作伙伴集成系统 

最近骑士汽车零部件公司与合作伙伴出现的争端：他们的服务是否满足服务水平协议(SLA)的条款。当这类事件发生时,通常是一个劳动密集型任务，用来调查和补救。 

根据这一点,骑士汽车零部件公司开发了一个应用程序来记录发生了什么,以便在出现争议时作为证据。应用程序会定期评估骑士汽车零部件公司及其外部合作伙伴之间的通信服务器。应用程序记录了运行性能和正常运行的时间指标，并将他们发送到一个JMS队列，等待进一步处理。 

骑士汽车配件公司已经有一个具有web用户界面的事件管理应用程序。所缺少的是一个将收集到的指标填充到事件管理系统的数据库中的一个应用，图9.1展示了这个场景。 

这是一个相当简单的任务:一个JMS消费者监听JMS队列上的新消息。接着数据格式由XML转换为SQL，然后写入数据库。 

很快，你想出一个匹配如图9.1的路由: 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<route id="partnerToDB"> 

<from uri="activemq:queue:partners"/> 

<bean ref="partner" method="toSql"/> 

<to uri="jdbc:myDataSource"/> 

</route> 

</camelContext> 

发送的数据如下： 

<?xml version="1.0"?> 

<partner id="123"> 

<date>200911150815</date> 

<code>200</code> 

<time>4387</time> 

</partner> 

存储数据的数据库表映射也容易创建： 

create table partner_metric 

( partner_id varchar(10), time_occurred varchar(20), 

status_code varchar(3), perf_time varchar(10) ) 

剩下的就是一个相当简单的任务：将XML映射到数据库。因为你是一个务实的人，决定制定一个简单的、优雅的解决方案,以后，每个人都可以很容易进行维护。于是你决定不引入JPA或者Hibernate这样的框架。 

你把下面的映射代码放在了一个老式的bean中。 

代码清单9.1 

import org.apache.camel.language.XPath; 

public class PartnerServiceBean { 

public String toSql(@XPath("partner/@id") int id, 

@XPath("partner/date/text()") String date, 

@XPath("partner/code/text()") int statusCode, 

@XPath("partner/time/text()") long responseTime) { 

StringBuilder sb = new StringBuilder(); 

sb.append("INSERT INTO PARTNER_METRIC (partner_id, time_occurred, 

status_code, perf_time) VALUES ("); 

sb.append("'").append(id).append("', "); 

sb.append("'").append(date).append("', "); 

sb.append("'").append(statusCode).append("', "); 

sb.append("'").append(responseTime).append("')"); 

return sb.toString(); 

} 

} 

编写代码清单9.1中的10行左右的代码比使用框架快多了。首先定义方法来接受这四个值映射。请注意,您使用@XPath注解从XML文档中来抓取数据。接着使用StringBuilder构建insert Sql语句。 

为了验证这一点,可以启动一个单元测试如下: 

public void testSendPartnerReportIntoDatabase() throws Exception { 

String sql = "select count(*) from partner_metric"; 

assertEquals(0, jdbc.queryForInt(sql)); 

String xml = "<?xml version=\"1.0\"?> 

+ <partner id=\"123\"><date>200911150815</date> 

+ <code>200</code><time>4387</time></partner>"; 

template.sendBody("activemq:queue:partners", xml); 

Thread.sleep(5000); 

assertEquals(1, jdbc.queryForInt(sql)); 

} 

这个测试方法概述了原则。首先检查数据库是空的，然后构造样本XML数据，并使用Camel的ProducerTemplate将其发送到JMS队列中。由于JMS消息的处理是异步的,你必须等一等,让它处理。最后,你检查数据库包含一行数据。 

9.1.2 设置JMS代理(JMS broker)和数据库 

运行单元测试,需要使用一个本地JMS代理和一个数据库。你可以使用Apache ActiveMQ作为JMS代理，HSQLDB作为数据库。HSQLDB可以作为一个内存数据库运行，不需要独立运行。Apache ActiveMQ是一个功能强大的JMS代理，甚至还可嵌入到单元测试中。所有你要做的就是掌握一点Spring XML魔法，用其来设置JMS代理和数据库。如代码清单9.2所示： 

<beans xmlns="http://www.springframework.org/schema/beans" 

xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 

xmlns:broker="http://activemq.apache.org/schema/core" 

xsi:schemaLocation=" 
http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
http://camel.apache.org/schema/spring
http://camel.apache.org/schema/spring/camel-spring.xsd
http://activemq.apache.org/schema/core
http://activemq.apache.org/schema/core/activemq-core.xsd"> 

<bean id="partner" class="camelinaction.PartnerServiceBean"/> 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<route id="partnerToDB"> 

<from uri="activemq:queue:partners"/> 

<bean ref="partner" method="toSql"/> 

<to uri="jdbc:myDataSource"/> 

</route> 

</camelContext> 

<bean id="activemq" 

class="org.apache.activemq.camel.component.ActiveMQComponent"> 

<property name="brokerURL" value="tcp://localhost:61616"/> 

</bean> 

<broker:broker useJmx="false" persistent="false" brokerName="localhost"> 

<broker:transportConnectors> 

<broker:transportConnector uri="tcp://localhost:61616"/> 

</broker:transportConnectors> 

</broker:broker> 

<bean id="myDataSource" 

class="org.springframework.jdbc.datasource.DriverManagerDataSource"> 

<property name="driverClassName" value="org.hsqldb.jdbcDriver"/> 

<property name="url" value="jdbc:hsqldb:mem:partner"/> 

<property name="username" value="sa"/> 

<property name="password" value=""/> 

</bean> 

</beans>

在代码清单9.2中，你首先定义了一个id为partner的Spring Bean，以备路由中使用。接着，为了使Camel与ActiveMQ连接，你必须将ActiveMQ定义为一个Camel组件。brokerURL属性值为一个远程的ActiveMQ代理的URL，在本例中，ActiveMQ代理与Camel运行在同一个服务器上面。然后建立一个本地嵌入式的ActiveMQ代理，将其配置为使用TCP连接。最后配置JDBC数据源。



使用嵌入式ActiveMQ代理时，可以使用VM协议代替TCP协议 。

如果你使用了一个嵌入式ActiveMQ代理，你可以使用VM协议代替TCP协议，这样做绕过整个TCP协议栈，速度会快得多。例如，在代码清单9.2中， 你可以使用vm://localhost代替tcp://localhost:61616。

实际上，vm://localhost中的localhost就是代理名称，而不是一个网络地址，例如，你可以使用 vm://myCoolBroker作为代理名称，在broker标签中配置对应的brokerName属性:brokerName="myCoolBroker".

你使用vm://localhost的一个可能的原因是：工程师们很懒，他们改变了TCP协议为VM协议，但是代理名称localhost没改。



此示例的完整源代码位于chapter9/riderautoparts-partner目录，你可以尝试运行以下命令：

mvn test -Dtest=RiderAutoPartsPartnerTest



在源代码中,您还将看到如何准备数据库：创建表，并在测试完成后删除表。





9.1.3 消息丢失的故事

前面的测试案例测试的是一个正确的情况，但是如果连接数据库失败了，会发生什么呢？如何测试这种情况呢？

第6章介绍了如何使用Camel拦截器模拟连接失败。编写一个单元测试，将所有的处理逻辑放到一个方法中，如代码清单9.3所示：

public void testNoConnectionToDatabase() throws Exception {

RouteBuilder rb = new RouteBuilder() {

public void configure() throws Exception {

interceptSendToEndpoint("jdbc:*")

.throwException(new ConnectException("Cannot connect"));

}

};

RouteDefinition route = context.getRouteDefinition("partnerToDB");

route.adviceWith(context, rb);

String sql = "select count(*) from partner_metric";

assertEquals(0, jdbc.queryForInt(sql));

String xml = "<?xml version=\"1.0\"?>

\+ <partner id=\"123\"><date>200911150815</date>

\+ <code>200</code><time>4387</time></partner>";

template.sendBody("activemq:queue:partners", xml);

Thread.sleep(5000);

assertEquals(0, jdbc.queryForInt(sql));

}

为了测试数据库连接失败的情况，你需要拦截到数据库的路由并模拟抛出错误。你使用RouteBuilder做到了一点。接着你需要将拦截器添加到已有路由中，可以使用adviceWith方法。剩下的代码与之前的测试几乎是相同的，只是这次测试,没有行被添加到数据库中。



提示：你可以阅读6.3.3节中关于使用拦截器模拟错误的内容。



测试运行成功。但您发送到JMS的消息队列的消息怎么了?它没有存储在数据库中,那么它去了哪里?

原来,消息丢失了,因为你没有使用事务。默认情况下,JMS消费者使用auto-acknowledge(自动认证)模式,意味着，如果客户端收到了消息，就认为发送成功，此消息就会从JMS代理的队列中移除。

你必须做的是使用transacted acknowledge(事务型认证)模式。我们将在9.3节中看看如何做到这一点的,但首先,我们将讨论事务在Camel中是如何工作的。



9.2 事务基础

一个事务是一系列的事件。事务的开始通常称为begin，事务的结束通常称为commit(或者rollback)。图9.2展示了这一点：

为了测试图9.2中的事务情形，您可以编写所谓的本地管理事务，即在代码中手工管理事务。如下代码，代码基于JPA事务管理：

EntityManager em = emf.getEntityManager();

EntityTransaction tx = em.getTransaction();

try {

 tx.begin();

 

 tx.commit();

} catch (Exception e) {

tx.rollback();

}

em.close();

使用begin方法开启了事务，接着处理一系列的事件，最后根据是否有异常抛出来决定提交事务或者回滚事务。

您可能已经熟悉这一原则，Camel中的事务使用了同样的原则，并在更高层次上进行了抽象。在Camel事务中，你不需要在代码中调用begin和commit方法---你使用的是声明式的事务，配置在Spring XML文件中。我们在下一节中讨论如何工作的细节,所以如果现在不清楚，不需要担心。

使用声明式事务有什么好处呢？利用Spring，你把所有的配置信息都写入了Spring XML文件中，而不用关注使用的运行环境是什么。即不需要修改java代码来适应目标环境。Spring支持使用最少的配置来使用不同的环境。Spring的事务支持是一个伟大的技术,这就是为什么Camel利用Spring的事务而不是推出自己的事务框架。

提示：关于Spring的事务管理的更多信息,请参见Spring框架参考文档第十章:“Transaction Management”,网址：http: //static.springsource.org/spring/docs/3.0.x/spring-framework-reference/html/transaction.html.



既然我们已经证实Camel采用了Spring的事务支持,让我们看看他们是如何一起工作的。

9.2.1 关于Spring的事务支持 

如图9.3所示，Spring负责事务管理，Camel负责其余的工作。 

图9.4展示了更多的细节，JMS代理也是事务的一部分，可以看出JMS代理、Camel和Spring的JmsTransactionManager是如何 一起工作的。JmsTransactionManager负责协调参与事务的资源：JMS代理。 


当JMS队列消费了一个消息，输入到Camel应用中，Camel向JmsTransactionManager.发出开启事务的信号，JmsTransactionManager.根据Camel路由是否成功完成来决定JMS代理是提交还是回滚。 

现在是时候来看看实战中这是如何工作的了。在下一节中,我们将通过添加事务来解决消息丢失的问题。 

9.2.2 添加事务 

在9.1节,消息丢失的问题还没有解决，因为你没有使用事务。现在你的任务是使用事务来解决消息丢失的问题。 

你会通过在Spring XML文件中引入Spring事务，并做相应调整配置。代码清单9.4显示了这是如何实现的。 

<bean id="activemq" 

class="org.apache.activemq.camel.component.ActiveMQComponent"> 

<property name="transacted" value="true"/> 

<property name="transactionManager" ref="txManager"/> 

</bean> 

<bean id="txManager" 

class="org.springframework.jms.connection.JmsTransactionManager"> 

<property name="connectionFactory" ref="jmsConnectionFactory"/> 

</bean> 

<bean id="jmsConnectionFactory" 

class="org.apache.activemq.ActiveMQConnectionFactory"> 

<property name="brokerURL" value="tcp://localhost:61616"/> 

</bean> 

首先你需要开启ActiveMQ组件的事务型认证模式：<property name="transacted" value="true"/>。接着使用JmsTransactionManager来引用Spring的事务管理器，对JMS消息事务进行管理。事务管理器需要知道如何连接到JMS代理，所以设置了connectionFactory属性。在jmsConnectionFactory定义中，配置了brokerURL指向了JMS代理。 


提示：JmsTransactionManager还有其他配置事务行为的配置项，如超时、回滚策略等等。请参考Spring文档：http://static.springsource.org/spring/docs/3.0.x/spring-framework-reference/html/transaction.html.


到目前为止，你还没有配置Camel路由相关的事务信息。Camel提供了约定优于配置的事务支持，你所需要做的就是在路由中<from>标签后添加<transacted/>标签： 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<route id="partnerToDB"> 

<from uri="activemq:queue:partners"/> 

<transacted/> 

<bean ref="partner" method="toSql"/> 

<to uri="jdbc:myDataSource"/> 

</route> 

</camelContext> 

当路由中声明了<transacted/>，Camel将为此路由添加事务支持。在后台，Camel查找Spring事务管理器并利用它。这是约定优于配置在起作用。 

在Java DSL中使用transacted也非常容易： 

from("activemq:queue:partners") 

.transacted() 

.beanRef("partner", "toSql") 

.to("jdbc:myDataSource"); 

约定优于配置仅适用于只有一个Spring事务管理器配置的场景。在更复杂的场景中，有多个事务管理器，你要做额外的配置来设置事务，我们将在9.4.1节讨论。 


注意：当在java DSL中使用transacted()方法，你必须将此方法添加到from()后面，以确保路由启用事务。 


9.2.3 测试事务 

当对Camel路由中的事务进行测试时，一般都使用真实的资源，如一个真正JMS broker和一个数据库。本书源码中使用了Apache ActiveMQ和HSQLDB，为了演示这是如何工作的,我们将回到骑士汽车零部件的例子。 

上次运行单元测试,在连接数据库失败时，丢失了消息。让我们再次运行这个单元测试，不过这次单元测试有了事务的支持。在本书源码chapter9/riderautoparts-partner目录中运行： 

mvn test -Dtest=RiderAutoPartsPartnerTXTest 


当您运行单元测试时,你会注意到很多异常堆栈在控制台上打印,他们会包含以下代码片段:





从堆栈中可以看出，EndpointMessageListener输出了错误日志，意味着事务回滚了。EndpointMessageListener是一个javax.jms.MessageListener，当一个消息到达JMS队列时，EndpointMessageListener会被调用，如果有异常抛出，EndpointMessageListener将回滚事务。

那么此时消息去哪儿了？它应该还在JMS队列中，让我们在清单9.3的测试方法中添加一些代码来验证一下：

Object body = consumer.receiveBodyNoWait("activemq:queue:partners");

assertNotNull("Should not lose message", body);



现在您可以运行单元测试,以确保消息没有丢失---单元测试将失败，出现断言错误:





我们使用了事务,并且事务被正确配置,但消息还是丢失了。错在哪里了？如果你深入研究异常堆栈,　你会发现信息总是尝试发送六次，然后不再尝试。

提示：如果你在使用Apache ActiveMQ，建议你阅读《ActiveMQin Action》一书，这本书详细解释了如何使用事务和返还。



ActiveMQ按照默认设置，在放弃前进行了六次的尝试返还，接着将消息移到了死信队列(见5.4节中Dead Letter Channel EIP)中。ActiveMQ实现了死信通道模式，这确保了JMS代理不会被有害消息(队列中无法成功处理的消息和造成新到消息堆积的消息)破坏。

所以，你应该到ActiveMQ默认的死信队列(队列名为;ActiveMQ.DLQ)中去找丢失的消息。

如果你改变相应的代码(以粗体显示),测试将通过:



你需要测试另一种情况：第一次调用数据库，连接失败，但是下一次调用连接成功。下面是测试方法：



思路就是只在第一次抛出连接异常。你依赖了一个事实：JMS队列消费的消息都有一系列的JMS头部，头部JMSRedelivered是一个Boolean类型，表明JMS消息是否被尝试重发过。

拦截器逻辑在RouteBuilder中完成。你使用了Content-Based Router(基于内容的消息路由) EIP来测试JMSRedelivered头部，只在头部值为false时抛出异常，也就是第一次发送时。其余的单元测试应该验证行为是否正确,所以你首先检查在JMSqueue发送消息之前数据库是否为空。接着休眠 一会儿，让路由运行完。之后，检查数据库中是否存在一行记录。最后检查死信队列是否为空。

我们刚刚讨论过的例子使用了所谓的本地事务，因为他们只使用了一个基于事务的资源---Spring只协调了JMS代理。例子中的数据库资源没有在事务控制范围内。让JMS代理和数据库都作为资源参与同一个事务，需要做更多的工作，下一节解释关于使用单个和多个资源在一个事务中。现在，我们先从企业集成模式的角度看下事务。

9.3 事务客户端(Transactional Client )EIP

Transactional Client EIP解决了这样一个问题：当与消息一起工作时，一个客户端如何控制事务？如图9.5所示。



图9.5显示了这种模式在《Enterprise Integration Patterns》一书中是如何描绘的，所以它可能有点难以理解：他和Camel的事务有什么关系？图中显示的是,一个发送方和接收方可以在同一个事务中协同工作。当接收者(receiver)启动事务,直到事务提交时，消息才会被发送，才会从队列中移除。当发送者(sender)启动事务，直到事务提交，消息对消费者(consumer)来说才是可用的。图9.6说明了这一原则。





图9.6的顶部一行使用EIP图标展示了路由，在事务中，一个消息由队列A移动到队列B。图9.6的其余部分展示了一个消息移动过程的用例。

图9.6的中部，展示了一个消息移动的快照。此时消息仍然驻留在队列A，尚未到达队列B。在事务提交前，消息都会驻留在队列A中，这保证了消息不会因为发生异常而丢失。

图9.6的底部，展示了事务提交的情况。此时消息从队列A中删除，插入到队列B中。事务客户端(Transactional clients)保证了整个过程是一个原子的、隔离的、一致的操作。

在谈到事务时,我们需要区分单个和多个资源的事务。单个资源的事务也称为本地事务(localtransactions),多个资源的事务也成为全局事务(globaltransactions);在接下来的两个部分,我们来看看这两种事务。

9.3.1 使用本地事务

图9.7描述了使用单一资源(即JMS代理)事务的情况。



在这种情况下,JmsTransactionManager负责协调单一资源事务。Spring的JmsTransactionManager事务管理器只负责协调基于JMS的资源。所以，数据库被排除在事务之外。

在9.1节骑士骑车配件例子中，数据库没有作为一个事务资源参与事务，但无论如何,这种方式看起来似乎也可以正常工作。

这是因为如果数据库决定回滚事务,将会抛出异常，Camel的TransactionErrorHandler会将这个异常传播给JmsTransactionManager，JmsTransactionManager做出反应：回滚事务。

这种情况并不完全等同于数据库也参加了事务,因为它仍然会出现可能会让系统处于不一致状态的错误。例如，在成功更新数据库之后，在JMS消息提交之前，JMS代理抛出异常。要确保JMS broker和数据库同步,你必须使用全局事务。



9.3.2使用全局事务

本地事务适用于只有一个资源参与事务的情况。但形势总是下不断变化,当你需要同一事务跨多个资源，如JMS和JDBC两个资源，如图9.8所示。



图9.8中，我们使用了可以处理多个资源的JtaTransactionManager事务管理器。Camel消费了队列中的一个消息，begin开启事务，消息被处理，更新数据库，事务提交。

那么JtaTransactionManager是个什么东东？他与前面章节使用的JmsTransactionManager有什么不同？回答这个问题，你需要先学一点关于全局事务和JTA方面的知识。

在Java中，JTA是XA标准协议的实现，这是一个全局事务协议。为了能够利用XA，事务中资源的驱动程序(如JDBC驱动和JMS驱动)必须兼容XA。JTA是JAVAEE规范的一部分，这意味着任何可以运行JAVAEE应用的应用服务器都支持XA。这是JAVAEE服务器的优点之一：JTA开箱即用,不像一些轻量级的替代品,比如Apache Tomcat不支持XA。

如果在JAVA EE应用服务器外部使用JTA，你需要多做一些设置工作，因为你必须找到和使用一个JTA事务管理器。如下其中一个:



然后你需要弄清楚在容器和单元测试中如何安装和使用它。在Camel和Spring的项目中使用JTA比较简单，只需要进行一系列的配置。 

注意：JTA的更多信息，参考维基百科页面:http://en.wikipedia.org/wiki/Java_Transaction_API。对XA也进行了简要的讨论：http://en.wikipedia.org/wiki/X/Open_XA.

JTA(全局事务)与本地事务的使用还是有一些不同。首先，你必须使用支持XA的驱动，意味着你要想让ActiveMQ参与到全局事务中，你必须使用 ActiveMQXAConnectionFactory。 

<bean id="jmsXaConnectionFactory" 

class="org.apache.activemq.ActiveMQXAConnectionFactory"> 

<property name="brokerURL" value="tcp://localhost:61616"/> 

</bean> 

对应的JDBC驱动也要支持XA。HSQLDB不支持XA，那么你可以使用Atomikos，Atomikos可以为不支持XA的JDBC的驱动模拟提供XA的能力： 

<bean id="myDataSource" 

class="com.atomikos.jdbc.nonxa.AtomikosNonXADataSourceBean"> 

<property name="uniqueResourceName" value="hsqldb"/> 

<property name="driverClassName" value="org.hsqldb.jdbcDriver"/> 

<property name="url" value="jdbc:hsqldb:mem:partner"/> 

<property name="user" value="sa"/> 

<property name="password" value=""/> 

<property name="poolSize" value="3"/> 

</bean> 

在实际的生产系统中，你应该选择一个支持XA的JDBC驱动，因为Atomikos的模拟实现XA是有缺点的。你可以在Atomikos官网上找到更多的信息。 

配置了XA驱动，你需要配置Spring的JtaTransactionManager事务管理器，引用真正的XA事务管理器的实现，在Atomikos中为： 

<bean id="jtaTransactionManager" 

class="org.springframework.transaction.jta.JtaTransactionManager"> 

<property name="transactionManager" ref="atomikosTransactionManager"/> 

<property name="userTransaction" ref="atomikosUserTransaction"/> 

</bean> 

剩余的配置都是关于Atomikos本身的配置，可以参见本书源码中chapter9/xa/src/test/resources/camel-spring.xml文件。 

假设您想在图9.8中的路由中添加一个额外的步骤：在消息插入到数据库中之后，对消息进一步处理。这个额外的步骤会影响事务的结果，无论是否抛出异常。假设它确实抛出一个异常,如图9.9中描述的。 

在图9.9中消息路由,在最后一步抛出异常。JtaTransactionManager会对此异常做出反应：回滚JMS代理和数据库事务。因为这个场景使用了全局事务，JMS代理和数据库事务全部回滚，最终结果是,好像整个事务从未发生。





这本书包含了这个例子的源代码，在chapter9/xa目录中运行：

mvn test -Dtest=AtomikosXACommitTest

mvn test -Dtest=AtomikosXARollbackBeforeDbTest

mvn test -Dtest=AtomikosXARollbackAfterDbTest

我们将离开这个全局事务的话题,继续学习更多关于配置和使用事务的内容。

9.4 配置和使用事务

到目前为止，在Camel中配置事务，使用的是约定优于配置的方式，通过在路由中添加<transacted/>标签实现参与事务。这是经常使用的方式，但是有些情况下，你可能需要对事务进行更精细的控制，例如设置事务传播特性。



9.4.1 配置事务

配置事务时,您将遇到事务传播(transaction propagation)这个词。在本节中,您将了解这个词是什么意思,以及它与配置事务的关系。

如果你曾经使用过EJB，您可能已经熟悉了事务传播。事务传播选项指定了如果一个方法调用和事务上下文已经存在,那么新事务应该怎么做,例如,应该加入现有的事务吗?应该开始一个新的事务?或者应该失败?

在大多数情况下,你会使用以下两个选项之一:加入现有的事务(PROPAGATION_REQUIRED)或开始一个新的事务(PROPAGATION_REQUIRES_NEW)。

使用事务传播,您必须在Spring XML文件中进行配置，如下面的例子，例子中使用了PROPAGATION_REQUIRED事务配置：



首先，定义一个id为required的bean，bean类型为SpringTransactionPolicy。这个bean必须配置相应的事务管理器和事务传播选项。在Camel路由的<transacted>标签中，使用ref属性引用了这个bean。 

如果想使用PROPAGATION_REQUIRES_NEW配置项，只需修改上述bean的属性propagationBehaviorName： 

<property name="propagationBehaviorName" value="PROPAGATION_REQUIRES_NEW"/> 


如果您曾经使用过Spring事务,你可以直接用事务传播类型来命名bean,如下: 

<bean id="PROPAGATION_REQUIRED" 

class="org.apache.camel.spring.spi.SpringTransactionPolicy"> 

<property name="transactionManager" ref="txManager"/> 

</bean> 

这两种配置方式没有实际的区别---看你倾向哪一种配置了。 

注意，bean中的propagationBehaviorName属性不是必须设置的，这是因为Camel会使用约定优于配置的方式来检查bean的id是否与事务传播项名称匹配。在本例中，bean的id为"PROPAGATION_REQUIRED"，Camel就会使用PROPAGATION_REQUIRED形式的事务传播特性。 

让我们看看还有什么其他使用约定优于配置的地方。 

在Camel路由中使用约定优于配置的方式来配置事务 

在9.2节中,您使用了默认的事务配置，这依赖于约定优于配置。当你想使用required事务传播项的时候，路由会正常工作。 

9.2.2节中的例子可以精简为只剩路由的配置： 

<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring"> 

<route id="partnerToDB"> 

<from uri="activemq:queue:partners"/> 

<transacted/> 

<bean ref="partner" method="toSql"/> 

<to uri="jdbc:myDataSource"/> 

</route> 

</camelContext> 

你只是在路由中使用了<transacted/>标签，此时，Camel将使用PROPAGATION_REQUIRED事务传播项，将自动寻找Spring的事务管理器。 

注意：这是一个常见的情况。通常你所要做的就是配置Spring事务管理器和在Camel路由中添加<transacted/>。 

现在,您已经了解了在Camel中如何配置和使用事务。但是还有更多的内容要学习。在下一节中,我们将看看当有多个路由时，当需要不同的传播特性时，事务是如何工作的。 

9.4.2 在多个路由中使用事务 

在Camel中，会出现多个路由，一个路由重用另一个路由发送的消息的场景。本节中我们看下当一个或多个路由参与事务时是如何工作的。然后我们看下在请求--响应形式的消息传输中使用事务有什么影响。 

我们首先看下事务路由调用一个非事务路由时，会发生什么。 


使用事务和非事务的路由 

清单9.6显示了单元测试的一部分，从中可以看下当一个事务路由调用一个非事务路由会发生什么。



在清单9.6中,首先导入Spring XMl文件,文件中包含了JMS代理、Camel的ActiveMQ组件以及Spring本身的配置。内容与清单9.4中的一样。

getExpectedRouteCount方法看上去有些奇怪，它是有用的，它告诉CamelSpringTestSupport，SpringXML中没有路由的定义----方法返回值：0。接着定义了两个路由，第一个路由是有事务的，将消息从队列A移到队列B。在移动过程中，消息被无事务的路由处理。注意到，当消息中包含"Donkey",无事务路由会抛出异常。

在本书源码chapter9/multuple-routes目录中运行命令：

mvn test -Dtest=TXToNonTXTest



单元测试有三个方法：两个测试提交事务的情况,一个测试回滚事务的情况。这里有两个测试提交和回滚的情况:

public void testWithCamel() throws Exception {

template.sendBody("activemq:queue:a", "Hi Camel");

Object reply = consumer.receiveBody("activemq:queue:b", 10000);

assertEquals("Camel rocks", reply);

}

public void testWithDonkey() throws Exception {

template.sendBody("activemq:queue:a", "Donkey");

Object reply = consumer.receiveBody("activemq:queue:b", 10000);

assertNull("There should be no reply", reply);

reply = consumer.receiveBody("activemq:queue:ActiveMQ.DLQ", 10000);

assertNotNull("It should have been moved to DLQ", reply);

}



你可以从这里学到什么?这个单元测试证明了：当一个事务路由调用了一个非事务路由，非事务路由会参与到事务中，成为事务路由。最后一个单元测试证明了：当一个非事务路由抛出了异常，事务路由探测到这个异常，回滚事务。从JMS代理的死信队列中的消息可以看到这一点。

这是好消息。事务路由调用非事务路由是安全的。

注意：事务管理器要求消息在同一个线程上下文中被处理，这样事务管理器才能提供事务支持。这意味着当您使用多个路由,你必须将他们联系在一起,确保在同一线程中处理消息。清单9.6中使用了Direct组件连接在一起，保证了位于同一线程中，如果使用SEDA组件将可能失败，因为SEDA组件会开启一个新线程路由消息。



两个事务路由的事务



现在，让我们修改下代码清单9.6的单元测试，将其中的非事务路由改成事务路由。如代码清单9.7：





运行单元测试的命令：

mvn test -Dtest=TXToTXTest

抛出异常时,整个路线回滚,这正是你所期望的。当消息到达第二个路由时，第二个是事务性的,它将参与现有的事务。这是因为当你不设置事务策略时，默认使用PROPAGATION_REQUIRED事务策略。

接下来,我们将通过使用两个不同的事务传播策略，使例子更具挑战性。



一个Exchange使用多个事务

某些情况下，你可能需要在同一个Exchange中使用多个事务，如图9.10所示：图中一个exchange正在Camel中路由，开始使用了PROPAGATION_REQUIRED类型的事务，接着你需要开启一个独立于现有事务的新事务。通过使用PROPAGATION_REQUIRES_NEW来实现这一点，PROPAGATION_REQUIRES_NEW会开启一个新事务，不管当前是否存已在事务。  当exchange处理完成时，事务管理器会同时提交两个事务。

注意：图9.10中概述的例子需要事务管理器支持事务的暂停和恢复。不是所有事务管理器实现都支持事务的暂停和恢复。



在Camel中，一个路由最多只能配置一个事务传播特性，这意味着图9.10中的情形，必须使用两个路由，第一个路由使用PROPAGATION_REQUIRED事务特性，第二个路由使用PROPAGATION_REQUIRES_NEW事务特性。

假设你有一个应用，用来更新订单到数据库中。应用程序必须在审计日志中存储所有传入的订单，然后进行更新或者插入到订单数据库。审计日志应该总是插入一个记录,即使订单的后续处理失败。在Camel中实现这一点需要两个路由，如下：

from("direct:orderToDB")

.transacted("PROPAGATION_REQUIRED")

.beanRef("orderDAO", "auditOrder")

.to("direct:saveOrderInDB");

from("direct:saveOrderInDB")

.onException(Exception.class).markRollbackOnlyLast().end()

.transacted("PROPAGATION_REQUIRES_NEW")

.beanRef("orderDAO", "updateOrInsertOrder");



第一个路由使用PROPAGATION_REQUIRED配置，使路由开启事务，而第二个路由使用PROPAGATION_REQUIRES_NEW配置，保证了在此路由中开启新事务。

现在假设在第二个路由处理过程中抛出了异常---你可以使两个路由同时回滚，也可以只回滚第二个路由。默认情况下，Camel会同时回滚两个路由，如果你只回滚第二个路由，你要告诉Camel。可以通过声明一个路由级别的onException，利用markRollbackOnlyLast方法来指示Camel只回滚最后一个(当前)路由的事务。如果要回滚两个路由事务，你可以删除onException声明，也可以用markRollbackOnly方法替换markRollbackOnlyLast方法。

在下一节，我们将回到骑士骑车零部件系统，看一个常见的例子：使用webservice和事务。如果事务失败，如何让webservice返回一个自定义的响应？



9.4.3 一个事务失败时返回一个自定义的响应信息

骑士汽车零部件系统有一个Camel应用程序，暴露了一个web服务，供一定数量的商业伙伴使用。商业伙伴使用这个服务提交订单。如图9.11所示。





正如你在图中所看到的，业务伙伴调用web服务提交一个订单。收到的订单被存储在数据库中以便进行审核。接着订单被ERP系统处理，最后返回给商业伙伴一个响应信息。 

暴露的web服务尽可能的简单，这样商业伙伴就可以在他们的IT系统中很容易调用这个web服务。有一个返回代码表明订单处理结果：成功还是失败了。下面的代码片段是响应的WSDL定义的一部分(outputOrder)： 

<xs:element name="outputOrder"> 

<xs:complexType> 

<xs:sequence> 

<xs:element type="xs:string" name="code"/> 

</xs:sequence> 

</xs:complexType> 

</xs:element> 

如果订单被成功处理，code字段值应该包含"OK"字符串；其他值可以看做处理失败。这意味着Camel应用必须处理抛出的任何异常，并且返回响应的错误消息，而不是将异常传播给webservice。 

你的Camel应用需要做下面三件事情： 

1、捕获异常，处理异常，禁止异常传播； 

2、回滚事务； 

3、构造一个响应消息：如code值中含"ERROR"信息的消息 

可以支持这种复杂的用例情况,因为你可以利用onException(详见第五章)。你需要做的就是在CamelContext中添加一个onException： 

<onException> 

<exception>java.lang.Exception</exception> 

<handled><constant>true</constant></handled> 

<transform><method bean="order" method="replyError"/></transform> 

<rollback markRollbackOnly="true"/> 

</onException> 

首先你告诉Camel这个onException应该处理任何类型的异常，接着将异常标记为已处理(这样就从exchange中移除了异常)，因为 你想用一个自定义消息来代替这个异常。 

注意：<rollback/>定义必须总是位于onException标签中的最后，因为他会停止消息进一步路由。这意味着你必须在事务回滚之前准备好响应消息。 

接着你使用order Bean的replyError方法来构建响应消息： 

public OutputOrder replyError(Exception cause) { 

OutputOrder error = new OutputOrder(); 

error.setCode("ERROR: " + cause.getMessage()); 

return error; 

} 

如你所见，这很容易做到。首先定义具有一个Exception类型参数的replyError方法---参数中将包含抛出的异常。接着创建Output对象，在对象中放入ERROR字符串和异常信息。 

这本书包含了这个例子的源代码，在chapter9/riderautoparts-order目录中运行命令： 

mvn camel:run 

然后你可以发送web服务请求到 http://localhost:9000/order.WSDL地址：http://localhost:9000/order?wsdl 

使用这个示例,您需要使用web服务。SoapUI是一个流行的Webservice测试框架。也很容易设置和启动。您创建一个新的项目并导入WSDL文件：http://localhost:9000/order?wsdl. 然后您创建一个示例web服务请求和填写请求参数,如图9.12所示。然后点击绿色的按钮,发送请求,响应将显示在右侧窗格中。



图9.12显示了一个抛出异常的示例。你可以改变请求参数进行测试。

到目前为止,我们一直在使用支持事务的资源,如JMS和JDBC，但大部分的组件不支持事务。所以你能做什么呢?在下一节中,我们将看看不支持事务的补偿。



9.5 补偿不被支持的事务

可以参与事务的资源的数量是有限的--他们大多局限于JMS和基于jdbc的资源。本文介绍了你能做什么来弥补没有事务支持的其他资源。补偿,在Camel中,涉及到工作单元的概念。

首先,我们来看看一个工作单元Camel中是如何表示的以及如何使用这个概念。然后我们将介绍一个示例，演示工作单元如何模拟实现事务管理器的功能。

9.5.1引入UnitOfWork(工作单元)

工作单元的概念是将一组任务关联在一起作为一个编排单元。思路是：使用工作单元作为模仿事务边界的一种方式。

在Camel中，工作单元由org.apache.camel.spi.UnitOfWork接口来表示，此接口提供了一系列的方法：

void addSynchronization(Synchronization synchronization);

void removeSynchronization(Synchronization synchronization);

void done(Exchange exchange);

addSynchronization和removeSynchronization方法用来注册和注销一个 Synchronization(一个回调函数)。当工作单元完成时，调用done方法，此方法会调用注册的Synchronization(一个回调函数)。

 Synchronization回滚是Camel用户最感兴趣的部分，因为他是在exchange处理完成时执行自定义逻辑的接口，其由org.apache.camel.spi.Synchronization接口表示，提供了两个方法：

void onComplete(Exchange exchange);

void onFailure(Exchange exchange);

在exchange处理完成时，会根据处理结果的成功或者失败来调用上述方法之一。

图9.13展示了这些概念之间的关系。如你所见，每一个Exchange都有一个UnitOfWork，你可以使用Exchange的getUnitOfWork方法来获取对应的工作单元。UnitOfWork是Exchange私有对象，不与其他Exchange共享。





UnitOfWork是如何编排的？

当Exchange路由时，Camel会自动将一个新的UnitOfWork注入到一个Exchange中。这是通过一个内部的处理器实现的：UnitOfWorkProcessor，此处理器在每一个路由开始就会参与进来。当Exchange处理完成时，处理器会回调注册的Synchronization。UnitOfWork的边界总是在Camel路由处理Exchange的开始和结束位置。

当一个Exchange路由完成时，会触发UnitOfWork的结束边界，此时，注册的Synchronization回调函数将会被一个接一个的调用。Camel组件使用了相同的机制来添加他们的Synchronization回调函数到Exchange中。例如，File和FTP组件使用这个机制来执行after-processing(后续处理)操作：移除或者删除已处理的文件。

提示：当Exchange处理完成时，可以使用Synchronization回调函数来执行任何你需要的后续处理操作。不需要担心从自定义回调函数中抛出异常---Camel将捕获那些异常并用WARN级别的日志信息输出，接着继续调用下一个回调函数。这保证了即使有一个回调函数抛出异常，所有的回调函数也会被全部调用。

理解这是如何工作的好方法是回顾一个例子,我们现在就做。



9.5.2 使用Synchronization回调函数

骑士汽车零部件有一个Camel应用程序：给客户发送包含发票详细信息的电子邮件。首先,电子邮件内容是自动生成的,其次，在邮件发送前，邮件会备份到一个文件中。每当一个发票信息发送到客户时，Camel应用就会参与。图9.14展示了这个应用的规则。



想象下，如果发送一封电子邮件时出现了问题，将会发生什么。你不能使用事务来回滚，因为文件系统资源不支持事务。相反，你可以使用自定义逻辑，通过删除文件来进行补偿。

实现补偿逻辑很简单,如下所示:

public class FileRollback implements Synchronization {

public void onComplete(Exchange exchange) {

}

public void onFailure(Exchange exchange) {

String name = exchange.getIn().getHeader(

Exchange.FILE_NAME_PRODUCED, String.class);

LOG.warn("Failure occurred so deleting backup file: " + name);

FileUtil.deleteFile(new File(name));

}

}

在onFailure方法中，你删除了备份文件，利用Camel的File组件的头部Exchange.FILE_NAME_PRODUCED获取了文件名。

下一步你要做的是让Camel调用这个FileRollBack类进行补偿。那么，你可以使用addSynchronization方法将这个类添加到UnitOfWork中，如代码清单9.13所示：





这本书包含了这个例子的源代码，在chapter9/uow目录中运行命令：

mvn test -Dtest=FileRollbackTest



如果你运行这个示例,它将输出类似以下内容到控制台:



可能困扰你的一件事是,你必须使用一个内联的Processor来添加FileRollback类。

Camel为Exchange提供了一个方便的方法，你可以利用这个方法：

exchange.addOnCompletion(new FileRollback());

但它仍然需要内联Processor。有没有一个更加方便的方式?是的，有,这就是onCompletion。



9.5.3 使用OnCompletion

OnCompletion将Synchronization带入了路由的世界，使你很容易地添加FileRollback类作为Synchronization。

那么让我们看看是如何做到的。OnCompletion可以用在Spring XML中也可以用在javaDSL中。

在Spring XML中：



可以看出，<onCompletion>独立于Camel路由定义，将在常规路由完成后执行。你所关注的是当exchange处理出错时onCompletion方法的执行，所以你可以显示设置onFailureOnly属性为true。

这本书包含了这个例子的源代码：

mvn test -Dtest=SpringFileRollbackTes



当你运行它,你会发现它的作用就像前面的例子：如果交易失败它将删除备份文件。



onCompletion和Synchronization的区别

使用onCompletion和Synchronization，有一个主要区别：两者使用的线程模型。 Synchronization使用同一个线程来执行任务，所以他将发生阻塞。onCompletion转由一个单独的线程执行exchange。

这样设计的原因是：onCompletion不应该影响原始了exchange以及exchange的输出。假设在使用onCompletion时抛出了异常----然后会发生什么?如果onCompletion意外或故意改变了exchange的内容,这将影响发送回调用者的响应内容?底线是onCompletion使用一个单独的基于完成的Exchange的路由。它不会影响原来路由的结果。



OnCompletion也可以用在Exchange处理成功的情况中。假设你想记录exchange的处理日志。例如：在javaDSL中可以这么做：

onCompletion().beanRef("logService", "logExchange");



OnCompletion也支持范围，与onException一样，有两个范围：context和route。你可以创建一个Java DSL版本的清单9.8，使用route-scoped onCompletionas，如下:

