## 第四章 Camel中bean的使用

本章包括：



1、理解Service Activator企业设计模式



2、Camel如何使用注册中心查找bean



3、Camel如何调用bean方法



4、单个参数绑定与多个参数绑定





4.1 使用bean的简单方式和复杂方式



在本节中，我们看一个例子，这个例子展示了使用bean的复杂方式(不建议这样使用)，接着看看使用bean的简单方式。



假设你有一个已经存在的bean，这个bean提供了一个操作(服务)，此操作在集成应用中使用。例如：HelloBean提供了hello方法作为服务:



public class HelloBean {



public String hello(String name) {



return "Hello " + name;



}



}



让我们看看在您的应用程序中使用这个bean的一些不同的方式。



4.1.1 纯java调用bean



使用Camel的Processor，可以实现纯java调用bean，见代码列表4.1public class InvokeWithProcessorRoute extends RouteBuilder {



public void configure() throws Exception {



from("direct:hello")



.process(new Processor() {



public void process(Exchange exchange) throws Exception {



String name = exchange.getIn().getBody(String.class);



HelloBean hello = new HelloBean();



String answer = hello.hello(name);



exchange.getOut().setBody(answer);



}



});



}





代码列表中展示了一个定义了路由的RouteBuilder类，使用了内联的Processor，可以在Processor的process方法中使用java代码操作消息。首先，从输入消息中获取消息体作为调用bean方法的参数。接着，实例化bean，并调用。最后，把方法的返回结果设置到输出消息中。



现在，你已经使用javaDSL完成了bean的调用，下面看下使用Spring XML的方式。



4.1.2 调用在spring中定义的一个bean



见代码列表4.2 4.3





<bean id="helloBean" class="camelinaction.HelloBean"/>



<bean id="route" class="camelinaction.InvokeWithProcessorSpringRoute"/>



<camelContext id="camel" xmlns="

http://camel.apache.org/schema/spring

">



<routeBuilder ref="route"/>



</camelContext>





public class InvokeWithProcessorSpringRoute extends RouteBuilder {



@Autowired



private HelloBean hello;



public void configure() throws Exception {



from("direct:hello")



.process(new Processor() {



public void process(Exchange exchange) throws Exception {



String name = exchange.getIn().getBody(String.class);



String answer = hello.hello(name);



exchange.getOut().setBody(answer);



}



});



}



}



到目前为止,您已经看到了两个在路由中使用bean的例子，为什么这是一种使用bean的复杂方式？下面是原因：



---你必须使用java代码调用bean



---你必须使用Processor，和路由耦合在一起，造成难以理解(路由逻辑与业务实现逻辑耦合在了一起)。



---你必须从消息中提取数据，然后传给bean，而且需要将bean的响应设置到Camel消息中。



---你需要手动实例化bean，或者使用依赖注入



下面我们来看下简单方式的bean使用：





4.1.3 使用bean的简单方式



假设你在Spring XML中定义了Camel路由，而不是使用RouteBuilder类。见下面的代码片段：



<bean id="helloBean" class="camelinaction.HelloBean"/>



<camelContext id="camel" xmlns="

http://camel.apache.org/schema/spring

">



<route>



<from uri="direct:start"/>



< What goes here >



</route>



</camelContext>





首先在Spring中定义bean，接着定义路由，使用direct:start作为路由输入。在之后，你准备调用HelloBean，但是你迷茫了，因为这是在XML中，你不能在XML中编写java代码。



在Camel中，使用bean的简单方式是：用<bean>标签：



<bean ref="helloBean" method="hello"/>



完整的配置如下：



<camelContext id="camel" xmlns="

http://camel.apache.org/schema/spring

">



<route>



<from uri="direct:start"/>



<bean ref="helloBean" method="hello"/>



</route>



</camelContext>





在java DSL中，Camel提供了相同的解决方式：



public void configure() throws Exception {



from("direct:hello").beanRef("helloBean", "hello");



}



代码从8行减为1行。而且这一行代码很容易理解。你甚至可以省略hello方法，因为bean中只有一个方法：



public void configure() throws Exception {



from("direct:hello").beanRef("helloBean");



}





使用<bean>标签是一个优雅的使用bean的解决方案。不使用<bean>标签，你需要使用Processor调用bean，这是一个乏味的解决方案。





提示：



在java DSL中，你不需要在注册表中预先注册bean。相反，你可以提供bean的类名，camel将在启动时实例化这个bean。前面的例子可以简写为下面这样：



from("direct:hello").bean(HelloBean.class);





现在让我们从企业集成模式的视角看看如何在Camel中使用bean。





4.2 企业集成模式---Service Activator模式



此模式的含义是：一个服务，既能通过各种消息传递技术调用，也能通过非消息传递技术调用。图4.1说明了这一原则。





图4.1展示了一个服务激活组件，此组件依据请求调用相应的服务并返回应答。服务激活组件扮演了请求和POJO服务之间的中介角色。请求者发送一个请求到服务激活组件，服务激活组件负责将请求转换为POJO服务理解的格式，然后传递请求到POJO服务上。POJO服务会返回一个应答到服务激活组件，服务激活组件将应答传递给正在等待的请求者。



正如图4.1中所展示的，服务由POJO提供，服务激活器是Camel中的一个组件，可以转换请求和调用服务。这个组件就是Bean组件，此组件实际使用了org.apache.camel.component.bean.BeanProcessor类完成了这个任务。我们在4.4节中学习BeanProcessor。你应该注意的是，在Camel中，Camel使用Bean组件实现了Service Activator模式。





图4.2展示了图4.1与4.1.3节中路由代码之间的对应。



在你使用一个bean之前,你需要知道到哪里去寻找它。这就需要用到注册表。让我们来看看Camel是如何与不同的注册表协调工作的。





4.3 Camel的bean注册表





当Camel与bean一起工作时，Camel会在注册表中找到相应的bean。Camel的哲学是：利用最好的成熟框架，使用一个可插拔的注册体系结构来集成这个框架。Spring就是这样一个框架，图4.3展示了注册表是如何工作的。



图4.3展示了：Camel注册表是调用者和真实注册表之间的一个抽象。当一个请求需要查找一个bean，就会使用Camel的注册表，接着Camel注册表会到真实注册表中查找这个bean，最后bean返回响应。这种结构是一个松耦合、可插拔的结构，可以与多个注册表集成。请求者只需知道如何与Camel注册表交互。



在Camel中，Camel注册表只是一个服务提供接口：



org.apache.camel.spi.Registry



包含以下方法：



Object lookup(String name);



<T> T lookup(String name, Class<T> type)



<T> Map<String, T> lookupByType(Class<T> type)





你经常使用的会是前两个方法，通过bean名称查找bean。例如，查找HelloBean：



HelloBean hello = (HelloBean) context.getRegistry().lookup("helloBean");



为了避免类型转换，可以使用第二个方法：



HelloBean hello = context.getRegistry().lookup("helloBean", HelloBean.class);



提示：



第二种方法提供类型安全的查找,因为你提供了预期的类作为第二个参数。在内部，Camel使用类型转换机制将bean转换为所需的类型,如果必要的话。





接口中的最后一个方法，lookupType,经常由Camel内部使用，支持约定优于配置。Camel可以直接根据bean类型查找，而不需要知道bean名称。





注册表本身是抽象的,因此只是一个接口。表4.1中列出了Camel自带的四个实现类：



SimpleRegistry



这个简单实现用于单元测试或者在Google App engine中运行Camel时。





JndiRegistry



此实现使用已经存在的JNDI来查找bean







ApplicationContextRegistry



此实现与Spring一起工作，在Spring的ApplicationContext中查找bean。当在Spring环境中使用Camel时，此实现自动被使用。





OsgiServiceRegistry



此实现在OSGi服务注册表中查找bean。在OSGI环境中使用Camel时，此实现自动被使用。



在下面几节中,我们将学习这四个实现。





4.3.1 SimpleRegistry





SimpleRegistry是一个基于Map的注册表，用于单元测试和Camel独立运行的时候。



例如，如果你想对HelloBean进行单元测试，你可以注册HelloBean到SimpleRegistry中，然后在路由中引用它。





public class SimpleRegistryTest extends TestCase {



private CamelContext context;



private ProducerTemplate template;



protected void setUp() throws Exception {



SimpleRegistry registry = new SimpleRegistry();



registry.put("helloBean", new HelloBean());



context = new DefaultCamelContext(registry);



template = context.createProducerTemplate();



context.addRoutes(new RouteBuilder() {



public void configure() throws Exception {



from("direct:hello").beanRef("helloBean");



}



});



context.start();



}



protected void tearDown() throws Exception {



template.stop();



context.stop();



}



public void testHello() throws Exception {



Object reply = template.requestBody("direct:hello", "World");



assertEquals("Hello World", reply);



}



}





首先创建一个SimpleRegistry实例，注册HelloBean，注册名为helloBean。接着，注册表与Camel联合使用(将注册表作为一个参数传递给DefaultCamelContext构造函数)。为了方便测试，创建了一个ProducerTemplate，用于在测试方法中向Camel发送消息，最后，当测试完成时，清理资源。在路由中，使用beanRef方法调用HelloBean。







4.3.2 JndiRegistry



JndiRegistry,顾名思义,基于JNDI的注册表。这是Camel集成的第一个注册表，当你创建Camel实例时没有提供注册表，默认注册表就是JndiRegistry：



CamelContext context = new DefaultCamelContext();





JndiRegistry注册表，与SimpleRegistry注册表类似，常用于单元测试或者Camel独立运行的时候。Camel中的许多单元测试都使用了JndiRegistry，因为这个测试用例在SimpleRegistry注册表添加到Camel之前就创建好了。





当Camel和JavaEE应用服务器(提供了一个开箱即用的JNDI注册表)一起使用时，JndiRegistry是非常有用的。假如你想利用WebSphere应用服务器的JNDI注册表，可以使用如下代码片段：



protected CamelContext createCamelContext() throws Exception {



Hashtable env = new Hashtable();



env.put(Context.INITIAL_CONTEXT_FACTORY,



"com.ibm.websphere.naming.WsnInitialContextFactory");



env.put(Context.PROVIDER_URL,



"corbaloc:iiop:myhost.mycompany.com:2809");



env.put(Context.SECURITY_PRINCIPAL, "username");



env.put(Context.SECURITY_CREDENTIALS, "password");



Context ctx = new InitialContext(env);



JndiRegistry jndi = new JndiRegistry(ctx);



return new DefaultCamelContext(jndi);



}



你需要使用一个Hashtable来存储JNDI注册表信息，然后，创建一个JndiRegistry需要的javax.naming.Context对象。





Camel允许你在Spring XML中使用JndiRegistry。你需要做的就是定义一个bean，Camel会自动加载：



<bean id="registry" class="org.apache.camel.impl.JndiRegistry"/>



可以使用Spring依赖注入语法，注入Hashtable到JndiRegistry的构造函数中。





4.3.3 ApplicationContextRegistry



Camel与Spring一起使用时，ApplicationContextRegistry是默认注册表。更准确地说,当你在Spring XML中如下设置Camel时，ApplicationContextRegistry是默认注册表。



<camelContext id="camel" xmlns="

http://camel.apache.org/schema/spring

">



<route>



<from uri="direct:start"/>



<bean ref="helloBean" method="hello"/>



</route>



</camelContext>



使用<camelContext>标签定义Camel，会使Camel使用ApplicationContextRegistry注册表。此注册表允许你在Spring XML文件中定义bean。例如，是可以定义helloBean：



<bean id="helloBean" class="camelinaction.HelloBean"/>



没有比这更简单的了。当你使用Camel和Spring时，你可以像平时一样使用Spring bean，Camel会自动引用到这些bean，无需任何配置。





4.3.4 OsgiServiceRegistry



当Camel用于OSGi环境中，Camel将使用两步走的查找策略。第一步，在OSGi服务注册表中查找对应名称的服务，如果没有，第二步，在常规注册表中查找，例如ApplicationContextRegistry.





假设你想将HelloBean暴露为OSGI的一个服务，你可以这样做：



<osgi:service id="helloService" interface="camelinaction.HelloBean"



ref="helloBean"/>



<bean id="helloBean" class="camelinaction.HelloBean"/>



在Spring动态模块(Spring DM)提供的osgi:service命名空间的帮助下你将HelloBean注册为OSGI注册表的服务，服务名称为helloService。与4.3.3节中Camel引用HelloBean的方式一样，Camel通过OSGI服务名称来引用服务：



<camelContext id="camel" xmlns="

http://camel.apache.org/schema/spring

">



<route>



<from uri="direct:start"/>



<bean ref="helloService" method="hello"/>



</route>



</camelContext>





　它是那么简单。你需要记住的是暴露的服务的名称。Camel会在OSGi服务注册表和Spring bean容器中查找服务。这是约定优于配置





提示:在第十三章中，我们会再次学习OSGI。





有关注册表的旅行到此结束。接下来学习Camel如何选择Bean方法。





4.4 选择Bean方法



你已经从路由的视角看到了Camel与bean如何协调工作的。现在，是时候深入学习一下了。你首先要理解的是Camel选择调用Bean方法的内部机制。





记住，Camel扮演了一个服务激活器的角色(利用BeanProcessor)，位于调用者和真实bean之间。在编译阶段，没有直接的绑定，JVM不能链接到bean的调用者--Camel在运行是解决这个问题。





图4.4展示了BeanProcessor如何利用注册表来查找要调用的bean的：





在运行是，Camel的exchange被路由，在路由中的某个点，到达BeanProcessor。BeanProcessor处理exchange，处理步骤如下：



1、在注册表中查找bean



2、选择所调用bean的方法



3、绑定参数到选中的方法上(例如，使用输入消息的body作为一个参数，具体细节在4.5节讨论)



4、调用方法



5、处理调用过程中的错误(从bean中抛出的任何异常都会设置到Camel的exchange中，以备进一步处理)



6、设置方法的响应结果(如果有的话)到输出消息的body中





我们在4.3节讨论了注册表的查找。第二步、第三步比较复杂，将在本章剩下的内容中讲解。比较复杂的原因：Camel必须在运行阶段计算调用哪个bean方法，而java代码的链接是在编译阶段。





Camel为什么需要选择一个方法？



答案是bean中可能会有重载方法，而且在某些情况下，方法名也不明确，意味着Camel必须在所有的方法中选择。



假设bean中包含如下方法：



String echo(String s);



int echo(int number);



void doSomething(String something);



一共有三个方法供Camel选择。如果你明确告诉Camel选择echo方法，仍然有两个要选择。我们看一下Camel是如何解决这一困境的。





我们首先看一下Camel选择方法是的算法。接着看一些例子，看看可能出现的错误,以及如何避免这些错误。





4.4.1 Camel如何选择bean方法 

与在编译阶段不同，Java编译器可以将方法调用链接在一起，Camel的BeanProcessor需要在运行阶段选择所调用的方法。假设你有如下一个java类：

public class EchoBean {

String echo(String name) {

return name + " " + name;

}

}

在编译阶段，你可以像下面这样调用echo方法：

EchoBean echo = new EchoBean();

String reply = echo.echo("Camel");



这样保证了在运行阶段echo方法被调用。

另一方面，假设你在Camel的路由中使用EchoBean：

from("direct:start").bean(EchoBean.class, "echo").to("log:reply");

当编译器编译这行代码时，编译器不知道你要调用EchoBean的echo方法。从编译器的视角来看，"EchoBean.class"和"echo"都是bean方法的参数。编译器能检查的是EchoBean类是否存在。如果你将"echo"方法名输错为"ekko"，编译器是不会发现这个错误的。这个错误会在运行阶段被捕获，此时BeanProcessor抛出MethodNotFoundException异常，指出ekko方法不存在。



Camel还允许不明确指定方法名。例如，你可以像下面这样改写上面的路由：

from("direct:start").bean(EchoBean.class).to("log:reply");



不管你是否显示指定方法名，Camel都必须计算调用哪个方法。让我们看看Camel是如何选择的。





4.4.2 Camel方法选择算法



BeanProcessor使用了一种复杂的算法来选择bean中所调用的方法。你不必理解或者记住算法中的每一步---我们只是想列出Camel内部到底发生了什么,让使用bean尽可能简单。



图4.5展示了算法的第一部分，第二部分在图4.6中。

下面是该算法的选择要调用的方法的步骤：

1、如果Camel的消息包含一个key为CamelBeanMethodName的头部，它的值用于显示指定方法名。调转到步骤5.

2、如果方法显示指定，Camel将使用它，就像本节开头提到的那样。跳转到步骤5.

3、如果所调用的bean可以转换为Processor类型(使用Camel的类型转换机制。)，转换后的Processor用来处理消息。这似乎有点奇怪的，但Camel允许把任何bean转换为消息驱动bean。例如，利用这种技术，Camel允许直接调用javax.jms.MessageListener Bean，无需额外的集成操作。Camel的最终用户很少使用这种方式,但它可以是一个有用的技巧。

4、如果Camel的消息体可以被转换为org.apache.camel.component.bean.BeanInvocation对象，此对象用于调用bean方法以及向bean传递参数。这一点也很少被Camel的最终用户使用。

5、算法的第二部分,如图4.6所示。



图4.6有一点复杂，但是它的主要目标是缩小方法选择范围，直至选中一个方法(如果这个方法存在的话)。如果现在你对这个算法没有完全了解，不用担心，在下一节我们来看几个例子，就会使你更容易理解。

让我们继续学习算法的最后几个步骤:

6、如果路由中提供了方法名，但是bean中没有对应的方法名，则抛出异常：MethodNotFoundException。

7、如果只有一个方法使用@Handler注解标记，则选中它。

8、如果只有一个方法使用了Camel的参数绑定注解(例如：@body，@header等等)，则选中它(参数绑定将在4.5.3节中介绍)。

9、如果在bean所有的方法中，只有一个方法的参数个数为1，则选中这个方法。例如，这种情况适用于4.1.1节中的EchoBean，EchoBean中只有一个echo方法，echo方法只有一个参数。单个参数的方法被优先选中，原因是，他们直接映射到了Camel中exchange的负载。

10、到这一步，算法变得有点复杂了。存在有多个候选方法，Camel必须确定是否有一个方法是最适合的。策略是过滤掉候选方法中不适合的方法。Camel通过匹配候选方法的第一个参数;如果不是同一类型的参数,而且不能进行强制类型转换,该方法过滤掉。最后,如果只剩下一个方法,该方法被选中。

11、如果最后没有选中方法，抛出AmbigiousMethodCallException异常，异常信息中包含模糊方法的列表。

显然，Camel对bean方法的选择做了很多。随着时间的推移你将学会欣赏这一切----约定优于配置。



提示： 这本书中提出的算法是基于Apache Camel 2.5版本。这种方法选择的算法在未来可能会改变，以适应新特性。



现在是时候看看这个算法在实践中应用了。



4.4.3 一些方法选择的例子



为了进一步了解算法是如何工作的，我们使用4.4.1节中的EchoBean作为例子，为了更好地解释有多个候选方法时的处理，我们给EchoBean添加一个bar方法。

public class EchoBean {

public String echo(String echo) {

return echo + " " + echo;

}

public String bar() {

return "bar";

}

}

我们将从这条路由开始：

from("direct:start").bean(EchoBean.class).to("log:reply");



如果你向路由发送消息："Camel"，响应日志会打印出"Camel Camel"。尽管EchoBean有两个方法：echo和bar，但是只有echo方法有一个参数，符合算法第9步---如果在bean所有的方法中，只有一个方法的参数个数为1，则选中这个方法。



为了使这个例子更具有挑战性，让我们改一下这个bar方法：

public String bar(String name) {

return "bar " + name;

}

你现在预计将会发生什么?现在有两个方法的参数个数为1.这种情况，Camel无法确定选择哪个，所以，抛出AmbigiousMethodCallException异常----符合算法第11步。



你如何解决这个问题?一种解决方法是在路由中显示指定方法名，例如显示指定bar方法：

from("direct:start").bean(EchoBean.class, "bar").to("log:reply");



还有另一种解决方法，不需要在路由中显示指定方法名。你可以在其中一个方法上使用@Handler注解---符合算法第7步。@Handler注解是一个Camel声明式的注解，你可以在方法上使用它。它只是告诉Camel默认调用这个方法。

@Handler

public String bar(String name) {

return "bar " + name;

}

现在，AmbigiousMethodCallException异常不会抛出，因为@Handler注解告诉Camel选择bar方法。



提示：无论是在路由中显示指定方法名或者是使用@Handler注解，都是一个好主意。这样可以保证Camel选中你所期望选中的方法。



假设你改变了EchoBeanto的两个方法，并使其有不同的参数类型:

public class EchoBean {

public String echo(String echo) {

return echo + " " + echo;

}

public Integer double(Integer num) {

return num.intValue() * num.intValue();

}

}

echo方法参数类型是String，double方法的参数类型是Integer。如果你没有显示指定方法名，BeanProcessor必须在两个方法中选择。

算法第10步显示：允许Camel智能决定选择某个方法。Camel通过检查两个或两个以上的候选方法的消息负载，并且与消息体的类型对比，检查是否有一个精确类型匹配的方法。

假设你发送了一个消息到路由，消息体的类型是String类型，内容为"Camel"。不难猜测，Camel将会选择echo方法，因为echo方法的参数类型为String。另一方面，如果你发送的消息体是一个Integer类型的数字5，Camel将会选择double方法，因为double方法的参数类型是Integer类型。

尽管如此,仍然可能出现问题,让我们来看几个常见的情况。



4.4.4 潜在的方法选择问题 
 在运行时调用bean，会出现几种错误： 
 1、Specified method not found---如果Camel找不到显式指定的方法，会抛出MethodNotFoundException。此异常只会在你显式指定方法名时发生。 
 2、Ambiguous method---如果Camel不能挑出一个方法来调用，会抛出AmbigiousMethodCallException异常，异常中会包含模糊方法信息。即使指定了方法名，也可能抛出这个异常，因为方法可以被重载。 
 3、Type conversion failure---在Camel调用所选方法之前，Camel需要将消息负载类型转换为方法参数类型，如果类型转换失败，抛出NoTypeConversionAvailableException异常。 

 让我们看看这三种情况的例子，我们使用下面的这个EchoBean来演示： 
 public class EchoBean { 
 public String echo(String name) { 
 return name + name; 
 } 
 public String hello(String name) { 
 return "Hello " + name; 
 } 
 } 
 首先，显式指定一个EchoBean中并不存在的方法： 
 <bean ref="echoBean" method="foo"/> 
 在这里你试着去调用foo方法，但是foo方法并不存在，所以Camel抛出MethodNotFoundException异常。 
 另一方面，你可以省略显式指定的方法： 
 <bean ref="echoBean"/> 
 在这种情况下，Camel无法选出一个方法，因为echo和hello两个方法是模糊的。此时，Camel会抛出AmbigiousMethodCallException异常，异常信息中会包含echo和hello两个方法的信息。 
 最后一种情况，假设你有如下类: 
 public class OrderServiceBean { 
 public String handleXML(Document xml) { 
 ... 
 } 
 } 
 而且你需要在如下路由中使用上述bean： 
 from("jms:queue:orders") 
 .beanRef("orderService", "handleXML") 
 .to("jms:queue:handledOrders"); 

 handleXML方法需要org.w3c.dom.Document类型的参数，但是如果JMS队列发送的是javax.jms.TextMessage类型的消息，并且消息中不含有任何的XML数据，而是一个纯文本消息，如"Camel rocks"。在运行时会出现以下异常： 
 Caused by: org.apache.camel.NoTypeConversionAvailableException: No type 
 converter available to convert from type: java.lang.byte[] to the 
 required type: org.w3c.dom.Document with value [[B@b3e1c9](mailto:B@b3e1c9)
 at 
 org.apache.camel.impl.converter.DefaultTypeConverter.mandatoryConvertTo 
 (DefaultTypeConverter.java:115) 
 at 
 org.apache.camel.impl.MessageSupport.getMandatoryBody(MessageSupport.java 
 :101) 
 ... 53 more 
 Caused by: org.apache.camel.RuntimeCamelException: 
 org.xml.sax.SAXParseException: Content is not allowed in prolog. 
 at 
 org.apache.camel.util.ObjectHelper.invokeMethod(ObjectHelper.java:724) 
 at 
 org.apache.camel.impl.converter.InstanceMethodTypeConverter.convertTo 
 (InstanceMethodTypeConverter.java:58) 
 at 
 org.apache.camel.impl.converter.DefaultTypeConverter.doConvertTo 
 (DefaultTypeConverter.java:158) 
 at 
 org.apache.camel.impl.converter.DefaultTypeConverter.mandatoryConvertTo 
 (DefaultTypeConverter.java:113) 
 ... 54 more 

 发生异常的原因是Camel试着将javax.jms.TextMessage类型转换为a org.w3c.dom.Document类型，当时类型转换失败。在这种情况下,Camel对错误进行了包装,把它作为一个NoTypeConverterException异常抛出。 
 通过进一步查看这个异常堆栈,您可能会注意到出现这一问题的原因是,xml解析器无法将数据解析为XML。异常报告指出："起始内容不允许"，意思是XML的声明(<?xml version="1.0"?>)丢失了。 

 如果这种情况发生在运行时，你可能想知道会发生什么。在这种情况下，Camel的错误处理系统将会介入并处理它。错误处理的内容将在第5章讲述。 
 这是所有你需要知道的关于Camel在运行时选择方法的内容。现在我们需要看看bean参数绑定过程，这一动作发生在Camel选择方法之后。 

 4.5 Bean参数绑定 
 在上一节中，我们介绍了Camel选择Bean方法的过程。本节讨论接下来会发生什么---Camel如何将参数适配到方法签名上。任何bean方法都可以有多个参数，Camel必须向参数传递有意义的值。这一过程被称为Bean参数绑定。 
 到目前为止，我们已经看过参数绑定的许多例子了。这些例子通常的共同点都是使用的单一参数，Camel将输入消息体绑定到参数上。图4.7使用EchoBean展示了一个例子： 

 BeanProcessor使用输入消息，将输入消息的消息体绑定到方法的第一个参数上，即String name参数。Camel通过创建一个表达式，将消息体转换为String类型。这就保证了在Camel调用echo方法时，参数类型是匹配的。 

 那么，当一个方法有多个参数时，会发生什么呢？这就是我们将在本章的剩余部分要学习的。 
 4.5.1 多个参数绑定 
 图4.8展示了多个参数绑定的原则。 
 首先，图4.8看起来似乎有些复杂。当处理多个参数时，出现了许多新类型。标题为Bean parameter bindings的大方框包含下面四个小方框： 
 1、Camel built-in types(Camel内建类型)---Camel提供了一系列特殊绑定的概念。这些内容在4.5.2节中学习。 
 2、Exchange---这是Camel的Exchange，它允许绑定到输入消息，比如它的body(消息体)和headers(消息头部)。Camel的exchange是参数绑定值的来源地，在下一节中学习。 
 3、Camel annotations---在处理多个参数时,可以使用注解来区分它们。这些内容在4.5.3节中学习。 
 4、Camel language annotations---　这是一个不常用的功能,允许你将参数绑定到Camel语言注解。在处理XML消息时使用这个功能是个好主意，可以使用XPath进行参数绑定。这些内容将在4.5.4节中学习。 

 使用多个参数 
 用多个参数比使用单一参数更复杂。遵循下面这些法则通常是一个好主意: 
 1、使用第一个参数作为消息体，可以使用@Body注解； 
 2、对其他参数使用一个内置类型或Camel注解。 
 在我们的经验中,多个参数不遵守时这些指导方针会变得复杂，但是Camel会尽力进行参数匹配。 
 让我们先看如何使用Camel内置类型。 

 4.5.2 使用Camel内置类型进行参数绑定 
 Camel为参数绑定提供了一组固定的类型。你所需要做的就是声明一个参数类型为表4.2中列出的类型之一。 
 理解这一点是非常重要的，因为大部分的bean方法都只有一个参数。第一个参数对应的就是输入消息的消息体，Camel会自动将消息体类型转换为与参数相同的类型。 
 表4.2 Camel自动绑定的参数类型 
 类型 Exchange 
 描述 Camel的exchange，包含将要绑定到方法参数的值。 

 类型 Message 
 描述 Camel的输入消息。包含绑定到方法第一个参数的消息体。 

 类型 CamelContext 
 描述 CamelContext，可以用在你要访问Camel的所有组件的情况下。 

 类型 TypeConverter 
 描述 Camel的类型转换机制。 当您需要进行类型转换时可以使用。在3.6节已经学习过类型转换机制。 

 类型 Registry 
 描述 bean注册表。这允许您在注册表中查找bean。 

 类型 Exception 
 描述 异常类型。只有在exchange中有错误或异常的时候，Camel参会绑定这个类型。利用此类型可以再bean中进行错误处理。 

 让我们看看几个使用表4.2中的类型的例子。假设你在echo方法中添加了第二个参数，参数类型为内建类型CamelContext： 
 public string echo(String echo, CamelContext context) 
 在这个例子中，你绑定了CamelContext，使你可以访问Camel的所有组件。 

 如果您需要在注册表总查找一些bean，你可以绑定注册表类型： 
 public string echo(String echo, Registry registry) { 
 OtherBean other = registry.lookup("other", OtherBean.class); 
 ... 
 } 
 你不会被局限于只有一个额外的参数,你可以使用尽可能多的参数。例如，你可以同时绑定 CamelContext和the registry： 
 public string echo(String echo, CamelContext context, Registry registry) 

 到目前为止,你一直绑定到消息体;你将如何绑定到一个消息头呢?下一节将解释。 

 4.5.3 使用Camel注解进行绑定

Camel提供了一系列的注解，用于exchange和bean参数之间的绑定。如果你想对参数绑定有更多的控制，你应该使用这些注解。例如，如果没有这些注解，Camel总是会将消息体绑定到方法的第一个参数上，但是如果使用@body注解，你可以将消息体绑定到方法的任一参数上。

假设你有如下bean方法：

public String orderStatus(Integer customerId, Integer orderId)

而且你有一个Camel消息，消息包含如下数据：

1、body(消息体)，包含订单id，类型为String

2、header(消息头部)，包含客户id，类型为Integer



利用Camel注解，你可以像下面这样将Exchange绑定到方法签名上面：

public String orderStatus(@Header("customerId") Integer customerId,

@Body Integer orderId)

注意：你使用了@Header注解将消息头部绑定到了第一个参数上，使用@Body注解将消息体绑定到了第二个参数上。

表4.3列出了所有的Camel参数绑定注解。



注解

@Attachments

描述

将参数绑定到消息附件上。此时的参数类型必须是java.util.Map类型。



注解

@Body

描述

将参数绑定到消息体上



注解

@Header(name)

描述

将参数绑定到给定的消息头上。



注解

@Headers

描述

将参数绑定到所有的输入头部上。此时的参数类型必须是java.util.Map类型。



注解

@OutHeaders

描述

将参数绑定到所有的输出头部上。此时的参数类型必须是java.util.Map类型。利用这个注解，你可以向输出消息中添加头部。



注解

@Property(name)

描述

将参数绑定到给定的exchange属性上



注解

@Properties

描述

将参数绑定到所有的exchange属性上。此时的参数类型必须是java.util.Map类型。



让我们再看几个例子。例如，你可以使用@Headers注解将输入头部绑定到Map类型上：

public String orderStatus(@Body Integer orderId, @Headers Map headers) {

Integer customerId = (Integer) headers.get("customerId");

String customerType = (String) headers.get("customerType");

...

}

当消息有多个头部时，可以使用此注解，这样你就不需要为每个头部都添加一个对应的参数了。

当你使用的是请求/响应形式的消息(也确定为InOut消息交换模式)时，可以使用@OutHeaders注解。@OutHeaders可以直接操作输出消息的头部，意味着你可以在bean中直接操纵这些消息头部。下面是一个例子：

public String orderStatus(@Body Integer orderId, @OutHeaders Map headers) {

...

headers.put("status", "APPROVED");

headers.put("confirmId", "444556");

return "OK";

}



你使用@OutHeaders注解了第二个参数。与@Headers不同, 当方法调用时，@OutHeaders是空的。这里的理念是：你负责把需要保存到输出消息中的头部设置到这个map中。



4.5.4 使用Camel语言注解进行参数绑定



此部分略