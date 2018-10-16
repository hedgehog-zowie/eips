## 第五章 错误处理

本章内容包括：

1、可恢复的错误与不可恢复的错误两者的区别

2、Camel错误处理使用的时机和位置

3、使用返还策略

4、使用onException处理异常、忽略异常

5、错误处理的细粒度控制



在前面三章,我们讨论了任何集成工具包应该提供三个主要功能：路由、转换和中介。在本章中，我们关注当错误发生时应该怎么做。我们提前向你介绍错误处理，是因为我们相信在你设计之初就应该考虑的一个关键点，而不是事后在进行错误处理。

编写应用程序集成异构系统，如何处理意想不到的事件是一个很大的挑战。在单一系统中，由于你有完全的控制权，你可以处理这些事件并恢复它。当时通过网络集成起来的系统有额外的风险：网络连接可能断掉、远程系统可能不会及时响应，或者它可能无缘无故宕机。即使在你的本地服务器,也可能发生意外事件,如服务器的磁盘满了或服务器内存耗尽。不管哪种错误出现,您的应用程序应该准备处理它们。

在这些情况下,日志文件通常是记录意外事件的唯一证据，所以日志非常重要。Camel对日志记录和错误处理有广泛支持，确保您的应用程序可以持续运行。

在本章，你会发现Camel的错误处理是多么灵活,深入和全面，学会如何对它进行定制,以应对大多数情况下的异常。我们将讨论Camel提供开箱即用的所有错误处理程序,以及他们的最佳使用场景,所以你可以选择最适合您的应用程序的处理方式。您还将了解如何配置和掌握返还技术,Camel可以使用返回技术尝试从特定的错误中恢复过来。我们还可以看到异常处理策略，这些策略允许你对错误进行区分，只处理特定的错误，以及定义通用错误处理规则，实现路由级别的错误处理。最后我们看下错误处理的细粒度控制。



5.1理解错误处理

在走进Camel错误处理的世界之前，我们需要退一步,看看更普遍的错误。首先，错误一般分为两大类：可恢复的错误和不可恢复的错误。其次我们需要看看何时何地开始错误处理，因为错误的发生是有先决条件的。



5.1.1 可恢复的错误和不可恢复的错误

当涉及到的错误，我们可以将他们分为可恢复的错误和不可恢复的错误，如图5.1所示。

不可恢复的错误是指一个错误,无论你尝试多少次相同的操作，它仍是一个错误。在集成的项目中,这可能意味着试图访问的数据库表不存在,这将导致JDBC驱动程序抛出SQLException异常。

可恢复的错误,是一个临时错误,在接下来的尝试操作后，可能并不会再次出现这个错误。比如网络连接错误导致的java.io.IOException。在接下来的尝试操作后,网络问题可能已经解决了,您的应用程序可以继续运行。



作为一个Java开发人员在日常生活中,你可能会遇到这样的错误分类： 可恢复的错误和不可恢复的错误。一般来说,异常处理代码使用两种分类之一,如下面两个代码片段所示。

第一个代码片段演示了一个常见的错误处理方式，将所有的异常认为是不可恢复的，放弃进一步的尝试，直接向调用者返回异常：

public void handleOrder(Order order) throws OrderFailedException {

try {

service.sendOrder(order);

} catch (Exception e) {

throw new OrderFailedException(e);

}

}

下一个片段通过添加一些处理改善了这种情况，在抛出异常之前进行的尝试，尝试5次后，如果仍然失败，抛出异常：

public void handleOrder(Order order) throws OrderFailedException {

boolean done = false;

int retries = 5;

while (!done) {

try {

service.sendOrder(order);

done = true;

} catch (Exception e) {

if (--retries == 0) {

throw new OrderFailedException(e);

}

}

}

}

上面的例子都缺少对错误类型的验证，即发生的错误时可恢复的还是不可恢复的？进而采取相应的措施。可恢复的情况下,你可以再试一次,不能恢复的情况下,你可以立即放弃并重新抛出异常。

在Camel中，可恢复的错误由Throwable和Exception代表，可以通过org.apache.camel.Exchange类中的下面两个方法进行存取：

void setException(Throwable cause);

Exception getException();



不可恢复的错误由一个消息代表，此消息有一个错误标识，此标识可以通过org.apache.camel.Exchange来存取。例如，设置"Unknown customer"作为一个错误消息，可以这么做：

Message msg = Exchange.getOut();

msg.setFault(true);

msg.setBody("Unknown customer");

错误标识必须使用setFault(true)方法设置。

那么,在Camel中，为什么这两种类型的错误代表不同?原因有两个：第一，Camel API的设计符合JBI(Java业务集成)规范，此规范中有个错误消息的概念。第二，Camel核心包中内置了错误处理，所以，只要Camel中抛出异常，Camel都可以捕获它，并把它设置到Exchange中作为可恢复错误，如下：

try {

processor.process(exchange);

} catch (Throwable e) {

exchange.setException(e);

}

使用这种模式允许Came捕获和处理所有异常。Camel的错误处理机制就可以决定如何处理捕获的错误---　重试,传播错误返回给调用者,或者做别的事情。Camel的终端用户可以将不可恢复的错误设置为错误消息，Camel能做出相应的反应,停止路由该消息。

现在你已经了解了可恢复和不可恢复的错误，让我们总结下他们在Camel中是如何表示的：

1、异常(Exceptions)表示为可恢复错误。

2、错误信息表示为不可恢复的错误。



现在让我们看看Camel在何时何地进行错误处理。



5.1.2 Camel进行错误处理的位置

Camel的错误处理不会应用于任何地方。要理解为什么，请看图5.2：

图5.2  Camel的错误处理仅适用于一个exchange的生命周期之内。





图5.2显示了一个简单的文件转换路由。使用一个文件消费者和生产者作为路由的输入和输出，在两者之间是Camel的路由引擎，将路由信息打包进一个exchange。Camel的错误处理适用于这个exchange的生命周期之内。这样就给了输入方一些处理错误的操作空间---文件消费者必须保证能够成功读取文件，初始化Exchange对象以及在Camel错误处理机制运行之前启动路由。这适用于任何类型的Camel消费者。 
 那么当文件消费者不能成功读取文件时，将会发生什么呢？答案是以组件而定，每一个Camel组件都必须有处理这种情况的方式。一些组件可能会忽略这个消息，其他组件可能会尝试一定的次数，还有一些组件可能会优雅地恢复这个错误。 

 提示：许多Camel组件都提供了自己的错误处理特性：File, FTP, Mail, iBATIS, RSS, Atom, JPA, 以及 SNMP组件。这些组件都是基于ScheduledPollConsumer类的，这个类提供了可插拔的PollingConsumerPollStrategy策略，你可以利用它创建你自己的错误处理策略。在Camel的网站上你可以了解更多这方面的内容： <http://camel.apache.org/polling-consumer.html.>

 了解这些背景资料就足够了----让我们开始深入研究Camel的错误处理机制是如何工作的。在下一节，我们将看到Camel的一系列错误处理器。 
 5.2 Camel中的错误处理 
 在前面的章节中，你已经学到Camel将所有的异常认为是可恢复的错误，并使用setException(Throwable cause)方法将他们存储在Exchange中。这意味着Camel错误处理机制只会对设置到Exchange上的异常做出反应。默认情况下，如果一个不可恢复的错误设置为一个错误消息，此时Camel的错误处理机制不会做出反应。也就是说在exchange.getException() != null.时，Camel的错误处理机制才会触发。 
 提示：5.3.4节中,您将学习如何设置Camel错误处理程序对错误信息做出反应。 
 骆驼提供一系列的错误处理器。他们在表5.1中列出。 

 表5.1 Camel的错误处理器 
 错误处理器 
 DefaultErrorHandler 
 描述 
 在没有配置其他错误处理器时，这是默认自动启用的错误处理器。 

 错误处理器 
 DeadLetterChannel 
 描述 
 此错误处理器实现了死信通道企业集成模式。 

 错误处理器 
 TransactionErrorHandler 
 描述 
 这是一个能够感知事务的错误处理器，扩展自默认的错误处理器。事务将在第九章中讲述，本章只会简短涉及到。在第九章中，我们会再次学习这个错误处理器。 

 错误处理器 
 NoErrorHandler 
 描述 
 这个处理器用来完全禁用错误处理器。 

 错误处理器 
 LoggingErrorHandler 
 描述 
 这个处理器只是记录异常日志。 

 乍一看,有五个错误处理程序似乎很复杂,但在大多数情况下你只会使用默认的错误处理器。 

 前三个错误处理器都继承自RedeliveryErrorHandler类。这个类包含了错误处理的主要逻辑。后两个错误处理程序功能有限，没有继承RedeliveryErrorHandler类。 
 我们来看看每一个错误处理器。 

 5.2.1 默认的错误处理器 
 DefaultErrorHandler是Camel默认的错误处理器，涵盖了Camel的大多数用例。要理解它，先看下面这个路由： 
 from("direct:newOrder") 
 .beanRef("orderService, "validate") 
 .beanRef("orderService, "store"); 

 默认的错误处理器是预配置的，不需要在路由中显示声明。那么，如果orderService bean的validate方法中抛出一个异常，将会发生什么？ 
 要回答这个问题，我们需要深入了解Camel内部的处理过程，以及错误处理所处的位置。在每一个Camel路由中，在路由图中的每两个节点中间，都有一个Channel，如图5.3所示。 

 Channel位于每个节点之间的路由路径上，确保它可以作为一个控制器,在运行时监视和控制路由。这是路由丰富特性(错误处理，消息跟踪，拦截器等等)之一。目前，你只需知道这是错误处理器所处的位置。 
 回到这个例子中路由上，假设从orderService bean的validate方法中抛出一个异常。就是说在图5.3中，标为3的Processor将抛出一个异常，这个异常将传播给他前面的channel，在这里，错误处理器将会捕获这个异常。这给了Camel做出相应反应的机会。Camel可以再尝试一次(redeliver),或者可以将这个消息路由到其他路由路径上(使用异常策略进行绕道)。在默认配置下，Camel会将异常传播给调用者。 
 默认的错误处理器有如下配置： 
 1、No redelivery 不进行再次尝试 
 2、异常传播给调用者 
 这与你在Java中使用异常类似，所以Camel的终端用户不会对此感到惊奇。 
 让我们继续学习下一个错误处理器，DeadLetterChannel。 

 5.2.2 DeadLetterChannel错误处理器 
 DeadLetterChannel错误处理器与默认的错误处理器类似，除了以下不同点： 
 1、DeadLetterChannel是唯一一个支持将错误消息传送到死信队列的错误处理器； 
 2、与默认错误处理器不同，DeadLetterChannel在默认情况下会处理异常，并将失败消息传送到死信队列。 
 3、当消息移动到死信队列时，DeadLetterChannel支持使用原始输入消息。 

 让我们来看看这些不同点的更多细节。 
 死信队列 

 DeadLetterChannel错误处理器是一个实现了Dead Letter Channel企业集成模式规则的错误处理器。这个企业集成模式的涵义是：如果一个消息不能被处理或者被重新发送，此消息应该被传送到一个死信队列。图5.4展示了这个模式。 
 正如你所看到的，标注1的消费者消费了一个新消息，此消息应该被路由到标注3的Processor上，标注2的channel是标注1、3之间的控制器，如果消息不能被发送到标注3上，channel会调用DeadLetterChannel错误处理器，将消息传送到死信队列(标注4)。　　这保证了消息的安全,并允许应用程序继续运行。 
 这种模式通常与消息传递一起使用。它不是让一个失败的消息阻止新消息的传递，而是将失败的消息移除当前路由(转到死信队列)。 

 同样的概念也适用于Camel的死信通道错误处理程序。此错误处理程序与一个作为端点(endpoint)的死信队列相连，允许你选择使用任何Camel端点。例如，你可以使用数据库，文件或者只是打印出错误消息日志。 
 当你选择使用DeadLetterChannel错误处理器，你必须配置死信队列作为一个endpoint，这样错误处理器才知道将错误消息转移到什么地方。这一点在java DSL和Spring XML中的实现有些不同。例如，下面是将日志消息以ERROR级别输出的两种实现： 
 errorHandler(deadLetterChannel("log:dead?level=ERROR")); 

 <errorHandler id="myErrorHandler" type="DeadLetterChannel" 
 deadLetterUri="log:dead?level=ERROR"/> 

 现在,让我们看看死信通道错误处理程序在他将消息传到死信队列时是如何处理异常的。 

 默认的异常处理方式 
 默认情况下，Camel用制止异常的方式来处理它，将异常从Exchange中移除，然后将异常存储到exchang的Properties中。在消息传送到死信队列之后，Camel停止路由此消息，调用者认为此消息被处理。 
 当一个消息被传送到死信队列，你可以通过exchange的Exchange.CAUSED_EXCEPTION属性来得到异常： 
 Exception e = exchange.getProperty(Exchange.CAUSED_EXCEPTION,Exception.class); 

 现在让我们看看使用原始消息的方式。 
 使用原始消息和死信通道 
 假设你有一个路由，在这个路由中，在消息到达目的地之前，消息会经历一系列的处理步骤，每一步都会对消息进行修改，就像下面的代码这样： 
 errorHandler(deadLetterChannel("jms:queue:dead")); 
 from("jms:queue:inbox") 
 .beanRef("orderService", "decrypt") 
 .beanRef("orderService", "validate") 
 .beanRef("orderService", "enrich") 
 .to("jms:queue:order"); 

 现在，假设validate方法抛出异常，deadLetterChannel错误处理器将消息发送到死信队列。此时一个新消息到达，并在enrich方法中抛出异常，那么此消息被发送到同一个死信队列。如果你先重新发送这些消息，你能直接将他们放到inbox队列中吗？ 
 理论上，你可以这么做，但是被发送到死信队列中的这些消息与到达inbox队列时的消息已经不再匹配了----他们在路由中被修改了。你所需要的是将原始消息发送到死信队列，这样你就可以使用原始消息进行重新发送。 
 useOriginalMessage配置选项可以使Camel将原始消息发送到死信队列。使用useOriginalMessage配置错误处理器方式如下： 
 errorHandler(deadLetterChannel("jms:queue:dead").useOriginalMessage()); 

 <errorHandler id="myErrorHandler" type="DeadLetterChannel" 
 deadLetterUri="jms:queue:dead" useOriginalMessage="true"/> 
 5.2.3 TransactionErrorHandler 
 TransactionErrorHandler错误处理器基于默认的错误处理器创建，具有默认错误处理器所有的功能，而且他还支持事务路由。第九章将讲解事务并详细讨论这个错误处理器，所以在这里我们不再多说。你只需要知道有这个TransactionErrorHandler错误处理器，他是Camel核心的一部分。 
 剩下的两个错误处理器很少使用，并且更简单。 
 5.2.4 NoErrorHandler 
 NoErrorHandler用来禁用错误处理器。Camel当前的体系结构要求必须配置一个错误处理器，所以,如果你想禁用错误处理,就需要提供一个空壳错误处理器，这就是NoErrorHandler。 
 5.2.5 LoggingErrorHandler 
 LoggingErrorHandler随异常记录失败消息的日志。日志记录器可以使用log4j或者Java Util Logger。 
 Camel默认会使用 org.apache.camel.processor.LoggingErrorHandler类，在ERROR级别来记录失败消息的日志。当然，你可以对其进行个性个配置。 
 学完了Camel提供的五个错误处理器，让我们来看下这些错误处理器的主要特性。 
 5.2.6 错误处理器的特性 
 默认错误处理器，DeadLetterChannel和TransactionErrorHandler错误处理器都是基于org.apache.camel.processor.RedeliveryErrorHandler创建，所以他们都有共同的几个主要特性。表5.2列出了这些特性。 
 表5.2 错误处理器的主要特性 
 特性 
 Redelivery policies(返还策略) 
 描述 
 返还策略允许你定义是否尝试返还的策略。此策略也可以定义尝试返还的次数、间隔等等。 

 特性 
 Scope 
 描述 
 Camel错误处理程序有两个错误处理范围：context(高级别的范围)和route(低级别的范围)。context范围允许多个路由公用同一个错误处理器，route级别，这个错误处理器只能被一个路由使用。 

 特性 
 Exception policies 
 描述 
 异常策略允许你为特殊的异常定义特殊的策略 

 特性 
 Error handling 
 描述 
 此特性允许你决定此错误处理器是否应该处理这个错误。你可以设置让错误处理器处理这个错误，或者留给调用者处理这个错误。 

 现在，你可能非常渴望看下错误处理器具体是如何使用的。在5.4.6节,我们将创建一个介绍错误处理的用例，到那时会有更多的机会使用错误处理器。但首先,让我们看看主要的特性。在5.3节中，我们将学习redelivery和scope两个特性。在5.4节中，我们将学习Exception policies和error handling两个特性。 
 5.3 使用错误处理器和返还策略 
 与远程服务器进行交互依赖网络连接，而网络连接是不可靠的或者可能发生中断。幸运的是这些中断导致的错误都是可恢复的错误---网络连接可能在几秒或几分钟后恢复。远程服务也可能是出现问题的根源，比如一个服务被管理员重启。为了解决这些问题，Camel支持返还机制,允许您控制处理可恢复错误。 
 在本节中,我们将看一个现实生活中的错误处理场景，然后专注于Camel如何控制返还,以及如何配置和使用返还策略。还会看到如何使用错误处理器来处理错误消息。在本节最后，将学习错误处理器的作用域。以及在不同作用域支持多个错误处理器。 
 5.3.1 错误处理用例 
 假设你为骑士骑车配件系统开发了一个集成应用程序，每隔一小时将本地目录中的文件上传到一个HTTP服务器，此时你的老板问你，在最近几天，为什么没有更新的文件。你很惊讶,因为这个应用程序已经成功运行了一个月。此时就是Camel错误处理机制出场的时候了。 
 集成路由如下： 
 from("file:/riders/files/upload?delay=1h") 
 .to("[http://riders.com?user=gear&password=secret](http://riders.com/?user=gear&password=secret)"); 
 此路由将周期性地扫描/riders/files/upload目录，如果有文件存在，将会把文件上传到HTTP服务器。但是路由中没有显式配置错误处理器，所以当错误发生了，默认的错误处理器被触发。默认错误处理器不处理异常，而是将异常传播给调用者。因为路由中的调用者是file消费者，它将记录异常,做一个文件回滚，这意味着文件会留在服务器上，在下一个轮询周期中继续上传。 
 此时,您需要重新考虑在应用程序中如何处理错误。问题不是很严重，因为没有文件丢失---Camel只有将成功处理的文件移除目录---处理失败的文件留在原目录。 
 发生错误的时间点是将文件发送到HTTP服务器的时候，所以你查看日志文件，很快就确定了是Camel无法连接到远程HTTP服务器导致的异常。你的老板决定应用程序应该重新上有错误的文件,不应该需要等到下一个小时再次上传。 
 要实现这一点,您可以配置错误处理器，设置返还次数(5此)以及延迟时间(10秒)： 
 errorHandler(defaultErrorHandler() 
 .maximumRedeliveries(5).redeliveryDelay(10000)); 
 配置返还策略是如此的简单。但是让我们深入看下如何使用返还策略。 
 5.3.2 使用返还策略 

表5.1中的前三个错误处理器都支持返还。返还的实现在他们所继承的RedeliveryErrorHandler类中。RedeliveryErrorHandler类必须知道是否进行尝试返还，这就是返还策略出现的目的。

返还策略用来定义如何以及是否尝试返还。表5.3列出了返还策略的配置选项以及他们的默认值。

表5.3 返还策略配置选项

| 配置选项                 | 类型    | 默认值 | 描述                                                         |
| ------------------------ | ------- | ------ | ------------------------------------------------------------ |
| AsyncDelayedRedelivery   | boolean | false  | Camel是否应该使用异步延迟返还。当要进行一个周期性的返还时，Camel一般情况下会阻塞当前现成，知道开始返还。通过启用这个配置选项，Camel将使用调度器,创建一个异步线程执行返还。保证了在等待返还的周期内，没有线程被阻塞。 |
| BackOffMultiplier        | double  | 2.0    | 指数补偿乘数，乘以每个顺向延迟。RedeliveryDelay开启了延迟，指数补偿默认被禁用。 |
| CollisionAvoidanceFactor | double  | 0.15   | 用于计算一个随机延迟偏移时的百分比。避免在下次尝试时使用相同的延迟。随着RedeliveryDelay开启延迟而启动。默认情况下禁用。 |
| DelayPattern             | String  | -      | 用于计算延迟的一个模式。该模式允许您指定固定延迟间隔组。例如，模式"0:1000;    5:5000;10:30000"的意思是：第0至4次尝试的延迟是1秒，第5至9次的尝试间隔是5秒，剩下的尝试间隔是30秒。 |

| RetryAttemptedLogLevel   | LoggingLevel | DEBUG | 尝试返还时所使用的日志级别                           |
| ------------------------ | ------------ | ----- | ---------------------------------------------------- |
| RetriesExhaustedLogLevel | LoggingLevel | ERROR | 所有的返还都失败时的日志级别                         |
| LogStackTrace            | boolean      | true  | 当所有的返还尝试都失败时，是否需要记录异常堆栈信息。 |
| LogRetryAttempted        | boolean      | true  | 返还尝试是否应该被记录                               |
| LogExhausted             | boolean      | true  | 当所有的返还尝试都失败时，是否需要记录日志           |
| LogHandled               | boolean      | false | 被处理的异常是否需要记录日志                         |

 使用Java DSL，Camel提供了方便的构建方法，用来在错误处理器中配置返还策略。例如，当进行返还时，你想尝试5次，使用exponential backoff(指数退避),使用WARN级别的Camel日志级别，你可以使用下面的代码：

errorHandler(defaultErrorHandler()

.maximumRedeliveries(5)

.backOffMultiplier(2)

.retryAttemptedLogLevel(LoggingLevel.WARN));

使用Spring XML：

<errorHandler id="myErrorHandler" type="DefaultErrorHandler"

<redeliveryPolicy maximumRedeliveries="5"

retryAttemptedLogLevel="WARN"

backOffMultiplier="2"

useExponentialBackOff="true"/>

</errorHandler>



在Spring XML中有两个地方的配置需要注意。使用<errorHandler>标签的type配置选项，选择要使用的错误处理器的类型。本例中，使用了默认错误处理器。你必须通过设置useExponentialBackOff配置选项为true,来启用exponential backoff(指数退避)。

目前我们完成了一下功能，Camel使用返还策略中的信息来决定如何进行返还。但是，Camel内部发生了什么呢?还记得图5.3吗？Camel的路由中的每个处理步骤之间包含了一个Channel，这就是错误处理所发生的位置。错误处理器检测到每一个发生的异常并对已进行响应，决定下一步怎么做，比如返还或者放弃。

现在你知道了很多关于DefaultErrorHandler错误处理器知识,是时候看一点例子了。



DefaultErrorHandler与返还错略

在本书的源代码chapter5/errorhandler目录中，你可以看到一个例子。例子使用了下面的路由配置：



errorHandler(defaultErrorHandler()

.maximumRedeliveries(2)

.redeliveryDelay(1000)

.retryAttemptedLogLevel(LoggingLevel.WARN));

from("seda:queue.inbox")

.beanRef("orderService", "validate")

.beanRef("orderService", "enrich")

.log("Received order ${body}")

.to("mock:queue.order");



代码首先定义了一个context范围的错误处理器，最多进行两次尝试，延迟1秒。当尝试返还是，将在WARN级别输出日志。当消息到达enrich方法时会报错。

在chapter5/errorhandler目录中使用mvn test -Dtest=DefaultErrorHandlerTest命令运行这个例子。将会在控制台上看到下面的日志输出：

2009-12-16 14:28:16,959 [a://queue.inbox] WARN DefaultErrorHandler

\- Failed delivery for exchangeId: 64bc46c0-5cb0-4a78-a4a8-9159f5273601.

On delivery attempt: 0caught: camelinaction.OrderException: ActiveMQ in

Action is out of stock

2009-12-16 14:28:17,960 [a://queue.inbox] WARN DefaultErrorHandler

\- Failed delivery for exchangeId: 64bc46c0-5cb0-4a78-a4a8-9159f5273601.

On delivery attempt: 1caught: camelinaction.OrderException: ActiveMQ in

Action is out of stock

这些日志条目说明Camel未能正确路由消息，Camel输出了日志，日志中包含了尝试的次数，以及exchangeId值、异常信息。

Camel做返还尝试的位置是错误发生的源头位置，比如例子中，当调用enrich方法时发生错误，那么Camel将尝试返还路由中的.beanRef("orderService", "enrich")步骤。

当所有的尝试都失败的时候，Camel默认输出ERROR级别的日志(你可以用表5.3中的配置选项对其进行个性化配置)：

2009-12-16 14:28:18,961 [a://queue.inbox] ERROR DefaultErrorHandler

\- Failed delivery for exchangeId: 64bc46c0-5cb0-4a78-a4a8-9159f5273601.

Exhausted after delivery attempt: 3caught:

camelinaction.OrderException: ActiveMQ in Action is out of stock



提示：默认的错误处理器有许多配置选项,如表5.3所示。我们鼓励你尝试使用IDE加载这个例子，更改错误处理器的配置选项,看看会发生什么。



前面的错误日志中输出了尝试次数的id信息(第几次尝试)，那么Camel是如何知道的呢？Camel将此信息存储到了Exchange中。表5.4显示了该信息的存储。

表5.4 Exchange中与错误处理相关的头部

| 头部                          | 类型    | 描述                                 |
| ----------------------------- | ------- | ------------------------------------ |
| Exchange.REDELIVERY_COUNTER   | int     | 当前返还尝试次数                     |
| Exchange.REDELIVERED          | boolean | Exchange是否进行了尝试               |
| Exchange.REDELIVERY_EXHAUSTED | boolean | Exchange是否进行了所有的尝试仍然失败 |

 表5.4中的头部只有在Camel进行返还尝试时有效。这些头部在第一次返还尝试之前不存在，只有在返还尝试被触发了，这些头部才会被设置。 

 使用异步延迟返还 

 在前面的示例中,错误处理器配置为使用1秒延迟返还。当进行返还时，在进行返还之前，Camel将等待一秒。 

 看下控制台的输出，可以看到返还日志条目间隔1秒，从[a://queue.inbox]日志信息可以判定，进行返还尝试的是同一个线程。这被称为同步延迟返还。此时也许你会想到异步延迟返还，那么它是什么意思呢？ 

 假设有两个订单被发送到 seda:queue:inbox端点(endpoint)。消费者首先从队列中获得第一个订单并处理它。当处理失败了，开始周期性的尝试。在同步的情况下，消费者线程被阻塞，等待返还结束。这意味着队列中的第二个订单只有等待第一个订单完成后才会被处理。 

 在异步模式中却不是这个样子。消费者线程不再被阻塞，将开启新线程，获取队列中的第二个订单，继续处理。这有助于获得更高的可伸缩性,因为线程不再被阻塞。相反，线程可以继续服务新请求。 

 提示：我们将在第十章学习线程模型，解释Camel如何使用新线程处理周期性的尝试。通过启用asyncDelayed配置选项，可以使Delayer和Throttler企业集成模式，有类似的异步延迟模式。 

 这本书的源代码包含一个示例,该示例演示了同步和异步延迟返还之间的不同，在chapter5/errorhandler目录中，使用下面的Maven命令： 

 mvn test -Dtest=SyncVSAsyncDelayedRedeliveryTest 

 示例包含两个方法:一个用于同步模式,另一个用于异步模式。 

 同步模式的控制台输出应该是以下顺序： 

 [a://queue.inbox] INFO - Received input amount=1,name=ActiveMQ in Action 

 [a://queue.inbox] WARN - Failed delivery for exchangeId: xxxx 

 [a://queue.inbox] WARN - Failed delivery for exchangeId: xxxx 

 [a://queue.inbox] WARN - Failed delivery for exchangeId: xxxx 

 [a://queue.inbox] INFO - Received input amount=1,name=Camel in Action 

 [a://queue.inbox] INFO - Received order amount=1,name=Camel in Action,id=123,status=OK 

 异步模式的控制台输出应该是以下顺序： 

 [a://queue.inbox] INFO - Received input amount=1,name=ActiveMQ in Action 

 [a://queue.inbox] WARN - Failed delivery for exchangeId: xxxx 

 [a://queue.inbox] INFO - Received input amount=1,name=Camel in Action 

 [a://queue.inbox] INFO - Received order amount=1,name=Camel in Action,id=123,status=OK 

 [rRedeliveryTask] WARN - Failed delivery for exchangeId: xxxx 

 [rRedeliveryTask] WARN - Failed delivery for exchangeId: xxxx 

 5.3.3 错误处理器和范围 

 范围可以用来在不同的级别定义错误处理器。Camel支持两种范围：context和route。 

 Camel允许你定义一个context范围的全局错误处理器作为默认错误处理器，如果有必要，你可以配置route级别的错误处理器，只对特定的路由起作用。如代码清单5.1所示： 

 errorHandler(defaultErrorHandler() 

 .maximumRedeliveries(2) 

 .redeliveryDelay(1000) 

 .retryAttemptedLogLevel(LoggingLevel.WARN)); 

 from("file://target/orders?delay=10000") 

 .beanRef("orderService", "toCsv") 

 .to("mock:file") 

 .to("seda:queue.inbox"); 

 from("seda:queue.inbox") 

 .errorHandler(deadLetterChannel("log:DLC") 

 .maximumRedeliveries(5).retryAttemptedLogLevel(LoggingLevel.INFO) 

 .redeliveryDelay(250).backOffMultiplier(2)) 

 .beanRef("orderService", "validate") 

 .beanRef("orderService", "enrich") 

 .to("mock:queue.order"); 

 清单5.1是一个对前面错误处理的例子的改善。默认处理器的配置和前面的例子中相同，但是有一个获取文件的新路由，此路由处理文件，然后路由到第二个路由上。第一个路由使用默认错误处理器，第二个路由有一个路由级别的错误处理器deadLetterChannel。请注意,它有与前一个错误处理器不同的选项配置。 

 这本书的源代码包含这个示例,您可以在chapter5/errorhandler目录中运行Maven命令： 

 mvn test -Dtest=RouteScopeTest 

 错误处理器与SPRING XML 

 让我们使用Spring XML修改清单5.1中的示例: 

 <bean id="orderService" class="camelinaction.OrderService"/> 

 <camelContext id="camel" errorHandlerRef="defaultEH" 

 xmlns="

http://camel.apache.org/schema/spring

"> 

 <errorHandler id="defaultEH"> 

 <redeliveryPolicy maximumRedeliveries="2" redeliveryDelay="1000" 

 retryAttemptedLogLevel="WARN"/> 

 </errorHandler> 

 <errorHandler id="dlc" 

 type="DeadLetterChannel" deadLetterUri="log:DLC"> 

 <redeliveryPolicy maximumRedeliveries="5" redeliveryDelay="250" 

 retryAttemptedLogLevel="INFO" 

 backOffMultiplier="2" useExponentialBackOff="true"/> 

 </errorHandler> 

 <route> 

 <from uri="file://target/orders?delay=10000"/> 

 <bean ref="orderService" method="toCsv"/> 

 <to uri="mock:file"/> 

 <to uri="seda:queue.inbox"/> 

 </route> 

 <route errorHandlerRef="dlc"> 

 <from uri="seda:queue.inbox"/> 

 <bean ref="orderService" method="validate"/> 

 <bean ref="orderService" method="enrich"/> 

 <to uri="mock:queue.order"/> 

 </route> 

 </camelContext> 

 在Spring XML中配置context级别的错误处理器，你必须使用camelContext标签的errorHandlerRef属性。errorHandlerRef属性引用一个<errorHandler>，在本例中为"defaultEH".还有一个错误处理器DeadLetterChannel，作为第二个路由的路由级别的处理器。 

 5.3.4 处理fault 

 在5.2节中,我们提到,默认情况下Camel错误处理器只会对exception做出反应。因为fault并不表示为exception，而是一个启用了fault标识的消息，因此fault不会被错误处理器识别并处理。 

 可能有时你需要Camel错误处理器能够处理fault。假设一个Camel路由调用了一个远程的webservice，此webservice返回一个错误消息，你希望将这个错误消息作为异常对待，将其发送到死信队列。 

 我们使用一个单元测试实现了这个场景，使用了一个bean来模拟远程的webservice： 

 errorHandler(deadLetterChannel("mock:dead")); 

 from("seda:queue.inbox") 

 .beanRef("orderService", "toSoap") 

 .to("mock:queue.order"); 

 现在,想象一下orderService bean返回SOAP错误如下: 

 <?xml version="1.0" encoding="UTF-8" standalone="yes"?> 

 <ns2:Envelope xmlns:ns2="

http://schemas.xmlsoap.org/soap/envelope/

" 

 xmlns:ns3="

http://www.w3.org/2003/05/soap-envelope

"> 

 <ns2:Body> 

 <ns2:Fault> 

 <faultcode>ns3:Receiver</faultcode> 

 <faultstring>ActiveMQ in Action is out of stock</faultstring> 

 </ns2:Fault> 

 </ns2:Body> 

 </ns2:Envelope> 

 在正常情况下，Camel错误处理器不会对这个SOAP错误做出反应。如果让错误处理器做出反应，你必须启用Camel的fault处理配置。 

 在context范围启用fault处理配置： 

 getContext().setHandleFault(true); 

 在route范围启用fault处理配置： 

 from("seda:queue.inbox").handleFault() 

 .beanRef("orderService", "toSoap") 

 .to("mock:queue.order") 

 一旦启用了fault处理配置，Camel错误处理器将识别SOAP错误并做出反应。在内部，利用拦截器，SOAP错误被转换为一个Exception。 

 使用Spring XML来启用fault处理配置: 

 <route handleFault="true"> 

 <from uri="seda:queue.inbox"/> 

 <bean ref="orderService" method="toSoap"/> 

 <to uri="mock:queue.order"/> 

 </route> 

 在chapter5/errorhandler目录中运行： 

 mvn test -Dtest=HandleFaultTest 

 mvn test -Dtest=SpringHandleFaultTest 

 提示：可能返回错误消息的组件有： CXF, SOAP, JBI,NMR。 

 5.4 使用异常策略 

 异常策略用来拦截、处理特定的异常。例如，在运行时，异常策略可以影响错误处理器使用的返还策略。异常策略也可以处理异常或者错误消息。 

 注意：在Camel中，异常策略使用路由中的onException方法来声明，所以我们将交换使用onException和“exception policy.”两个术语。 

 我们将详细讨论异常策略，看下异常策略是如何捕获异常的，如何与返还策略一起工作，如何处理异常。然后我们将看看自定义错误处理和一个完整的例子。 

 5.4.1 理解onException如何捕获异常 

 我们先看下Camel如何检查异常层次结构，以此来决定如何处理错误的。这有助于你更好的理解如何更好地使用onException。 

 假设你有下面这个异常层次结构： 

 org.apache.camel.RuntimeCamelException (wrapper by Camel) 

 \+ com.mycompany.OrderFailedException 

 \+ java.net.ConnectException 

 出现错误的真正原因是ConnectException异常，但是ConnectException异常包装成了OrderFailedException异常，又进一步包装成了RuntimeCamelException异常。 

 Camel将自下而上遍历这个异常层次结构，以找到匹配的onException异常。在这种情况下，Camel先检查java.net.ConnectException异常，再检查com.mycompany.OrderFailedException异常，最后检查RuntimeCamelException异常。对每一个异常，Camel都会与onException中定义的异常作比较，以选择最佳匹配异常策略的异常。如果没有匹配的异常，Camel的处理依赖于错误处理器的配置。我们将深入看看匹配工作是如何进行的，现在你可能会认为Camel使用了instanceof语句块来检查异常层次中的异常匹配， 

 假设你有一个路由，路由中的onException定义如下： 

 onException(OrderFailedException.class).maximumRedeliveries(3); 

 前面提到的ConnectException异常被抛出，Camel错误处理器试着处理这个异常。因为你定义了一个异常处理策略，此策略将检查所抛出的异常是否符合策略中的异常。匹配过程如下： 

 1、Camel首先匹配 java.net.ConnectException异常，将这个异常与onException(OrderFailedException.class)比较。Camel会检查这两个异常是否是同一个类型的异常，此时的结果很明显，他们不是同一个类型。 

 2、Camel接着检查ConnectException是否是OrderFailedException的子类，很明显也不是。到此，Camel没有发现一个匹配的异常。 

 3、Camel沿着异常层次结构向上继续寻找，这次找到的是OrderFailedException异常，这一次是一个精确匹配的，因为他们都是OrderFailedException类型。 

 不需要更多的匹配了---Camel找到了一个精确匹配的，此时异常策略被使用。 

 当一个异常策略被选中，其配置的策略将被错误处理器使用。在这个例子中，策略定义了最大尝试次数为3次，所以当对应的异常抛出时，错误处理器将最多尝试3次。 

 异常策略中配置的值将会覆盖错误处理器中对应的配置选项值。例如，假设错误处理器有maximumRedeliveries值为5的配置选项，因为onException有一个同样的配置选项，那么onException的这个选项的值3将被使用。 

 注意：这本书的源代码有一个示例,该示例演示了刚刚学到的内容。看下chapter5/onexception目录中的OnExceptionTest类，他有过个测试方法，每一个方法演示了onException工作的一种场景。 

 让我们做更有趣的例子，添加第二个onException定义： 

 onException(OrderFailedException.class).maximumRedeliveries(3); 

 onException(ConnectException.class).maximumRedeliveries(10); 

 如果抛出的异常层次结构与前面的示例相同，Camel将会选中第二个onException，因为他直接匹配了ConnectionException。这允许您为不同的异常定义不同的策略。在本例中，ConnectException异常比OrderFailedException异常有更多的返还次数。 

 提示:这个例子演示了onException如何影响错误处理器使用的返还策略。如果错误处理器配置了2次错误尝试，当ConnectException异常抛出时，onException配置的10次尝试将会覆盖2次尝试的配置。 

 但是,如果没有直接匹配的呢？让我们看另外一个例子。这一次，假设java.io.IOException被抛出。Camel将其与OrderFailedException/ConnectException.异常匹配，因为两者不匹配(既不是同一个类型也不是子类)。在这种情况下，所有的onException都不匹配，Camel将会使用配置的错误处理器。 

 具体例子可以在chapter 5/onexception目录中运行命令： 

 mvn test -Dtest=OnExceptionFallbackTest 

 ONEXCEPTION AND GAP DETECTION 

 如果没有直接匹配的，Camel可以做的更好吗？答案是肯定的。因为Camel使用了GAP DETECTION(异常间隙检测)机制，计算抛出异常和onException异常直接的间隙，接着间隙最低的异常。听起来有些迷惑，来看一个例子。 

 假设你有一下onException定义，每一个都定义了不同的尝试策略： 

 onException(ConnectException.class).maximumRedeliveries(5); 

 onException(IOException.class).maximumRedeliveries(3).redeliveryDelay(1000); 

 onException(Exception.class).maximumRedeliveries(1).redeliveryDelay(5000); 

 假设抛出一下异常： 

 org.apache.camel.OrderFailedException 

 \+ java.io.FileNotFoundException 

 哪一个onException将会被选中呢？ 

 Camel首先进行java.io.FileNotFoundException和onException之间的匹配，结果是不精确匹配，Camel使用GAP DETECTION机制。在本例中，只有onException(IOException.class)和onException(Exception.class)与java.io.FileNotFoundException部分匹配，因为java.io.FileNotFoundException是java.io.IOException和java.lang.Exception的子类。 

 下面是FileNotFoundException的异常层次结构: 

 java.lang.Exception 

 \+ java.io.IOException 

 \+ java.io.FileNotFoundException 

 可以看出，java.io.FileNotFoundException是java.io.IOException的直接子类，gap(间隙)为1。java.lang.Exception和java.io.FileNotFoundException的gap为2.此时，gap(间隙)为1的异常为最近候选异常。 

 接着，Camel对抛出的异常层次中的下一个异常OrderFailedException进行相同的匹配过程，这一次，只有onException(Exception.class)部分匹配，OrderFailedException和Exception之间的gap也是1： 

 java.lang.Exception 

 \+ OrderFailedException 

 那么现在应该怎么办呢？有两个gap都为1.这种情况下，Camel总是选中第一个匹配的，因为引起错误的异常最有可能是层次结构中的最底层的异常，本例中为FileNotFoundException异常，所以胜者为onException(IOException.class). 

 这本书的源代码中提供了这个实例，在chapter5/onexception目录中，可以运行下面这个命令： 

 mvn test -Dtest=OnExceptionGapTest 

 Gap Detection允许你定义粗粒度的策略(context范围)，也可以定义细粒度的策略(route范围)来否决粗粒度的范围。 

 一个ONEXCEPTION多个异常 

 到目前为止，你只看到了一个ONEXCEPTION一个异常的例子，实际上你可以在同一个ONEXCEPTION中定义多个异常： 

 onException(XPathException.class, TransformerException.class) 

 .to("log:xml?level=WARN"); 

 onException(IOException.class, SQLException.class, JMSException.class) 

 .maximumRedeliveries(5).redeliveryDelay(3000); 

 使用Spring XML如下定义： 

 <camelContext xmlns="

http://camel.apache.org/schema/spring

"> 

 <onException> 

 <exception>javax.xml.xpath.XPathException</exception> 

 <exception>javax.xml.transform.TransformerException</exception> 

 <to uri="log:xml?level=WARN"/> 

 </onException> 

 <onException> 

 <exception>java.io.IOException</exception> 

 <exception>java.sql.SQLException</exception> 

 <exception>javax.jms.JmsException</exception> 

 <redeliverPolicy maximumRedeliveries="5" redeliveryDelay="3000"/> 

 </onException> 

 </camelContext> 

 5.4.2 理解onException如何与返还一起工作 

 假设有如下路由： 

 from("jetty:

http://0.0.0.0/orderservice

") 

 .to("mina:tcp://erp.rider.com:4444?textline=true") 

 .beanRef("orderBean", "prepareReply"); 

 使用Camel的Jetty组件，暴露了一个HTTP服务，用于根据订单状态查询订单。订单状态信息来自于一个远程的ERP系统，由MINA组件提供交互。你已经知道如何配置错误处理器和onException了。 

 假设当有一个IO异常时(比如网络连接错误)，你希望Camel尝试调用TCP服务，要做到这一点，你可以使用onException并配置返还策略： 

 onException(IOException.class).maximumRedeliveries(5); 

 那么两次尝试之间的延迟是多少呢？ 

 在本例中，延迟是1秒。Camel将使用表5.3列出的返还策略配置项的默认值，并用onException中的配置值覆盖默认值。因为onException没有设置延迟时间，所以使用默认的1秒延迟。 

 提示：当你配置返还策略，就覆盖了当前错误处理器中已有的返还策略。 

 现在让我们使它更加复杂: 

 errorHandler(defaultErrorHandler().maximumRedeliveries(3).delay(3000)); 

 onException(IOException.class).maximumRedeliveries(5); 

 from("jetty:

http://0.0.0.0/orderservice

") 

 .to("mina:tcp://erp.rider.com:4444?textline=true") 

 .beanRef("orderBean", "prepareReply"); 

 此时当IOException抛出时，延迟时间是多少呢？对，是3秒，因为onException使用了错误处理器中定义的返还策略。 

 现在，让我们从onException中移除maximumRedeliveries(5)配置项： 

 errorHandler(defaultErrorHandler().maximumRedeliveries(3).delay(3000)); 

 onException(IOException.class); 

 from("jetty:

http://0.0.0.0/orderservice

") 

 .to("mina:tcp://erp.rider.com:4444?textline=true") 

 .beanRef("orderBean", "prepareReply"); 

 此时当IOException抛出时，延迟时间是多少呢？你可能会说是3秒，因为错误处理器中定义了。这种情况，答案是0秒。Camel不会做任何返还，因为onException默认会将maximumRedeliveriesd的值置为0(即返还默认是禁用的)，除非你显式设置了maximumRedeliveries配置选项。 

 Camel这样做的原因见下节：使用onException处理异常。 

5.4.3 理解onException如何处理异常 

假设您有一个复杂的路由,包含多个步骤，每一步都对消息做了处理，并且每一步都有可能抛出异常(表明消息无法正确处理应该被抛弃)。这是就该onException出场处理异常了。 

使用onException处理异常与在java中处理异常类似。你可以将其看做使用一个try...catch代码块。 

一个例子最能说明这一点。假设你需要实现一个ERP端的服务，用来提供订单状态。下面是上一节你调用的ERP服务： 

public void configure() { 

try { 

from("mina:tcp://0.0.0.0:4444?textline=true") 

.process(new ValidateOrderId()) 

.to("jms:queue:order.status") 

.process(new GenerateResponse()); 

} catch (JmsException e) { 

.process(new GenerateFailureResponse()); 

} 

} 

这个伪代码片段在生成响应时用了多个步骤。如果报错，捕获异常并返回错误响应。 

之所以称之为伪代码，是因为他显示了你的意图但是代码无法通过编译。因为javaDSL有自己的构建语法，路由中的方法调用应该在一起。java中标准的try ... catch机制在configure方法运行时捕获抛出的异常，但是此种情况下，configure方法只会被调用一次，就是在Camel启动的时候(初始化并建立路由路径,以便在运行时使用)。 

不要失望，Camel在DSL中有一个与try ... catch ... finally对应的语句块： 

doTry ... doCatch ... doFinally. 


USING DOTRY, DOCATCH, AND DOFINALLY 

代码清单5.3向你展示了能够通过编译并且效果和try ... catch一样的实现方法： 

public void configure() { 

from("mina:tcp://0.0.0.0:4444?textline=true") 

.doTry() 

.process(new ValidateOrderId()) 

.to("jms:queue:order.status") 

.process(new GenerateResponse()); 

.doCatch(JmsException.class) 

.process(new GenerateFailureResponse()) 

.end(); 

} 

doTry……doCatch代码块有点钻牛角尖，但是他是有用的，因为它有助于弥补普通Java代码和DSL之间思考方式的不同。 


USING ONEXCEPTION TO HANDLE EXCEPTIONS 使用ONEXCEPTION来处理异常 

doTry ... doCatch代码块有一个限制，他只是route范围的。这个代码块只在所处路由中起作用。onException，另一方面，工作在context和route两种范围中，所以你可以使用onException修改代码清单5.3。如代码清单5.4所示： 

onException(JmsException.class) 

.handled(true) 

.process(new GenerateFailueResponse()); 

from("mina:tcp://0.0.0.0:4444?textline=true") 

.process(new ValidateOrderId()) 

.to("jms:queue:order.status") 

.process(new GenerateResponse()); 

doCatch和onException之间的一个不同是：doCatch将处理异常，而onException默认不处理异常。这就是使用handled(true)方法让Camel处理异常的原因。结果，当一个JmsException抛出时，应用会使用java的 try ... catch机制来捕获JmsException异常。 

在代码清单5.4中，你应该注意到关注点是如何被分离的，正常的路由定义没有和异常处理混淆在一起。 

假设一条消息到达了TCP端点，Camel应用路由这条消息。这条消息通过processor验证，发送到JMS队列，但是这个操作失败并抛出JmsExceptioin异常。图5.5是一个顺序图，展示了这情况在Camel内部发生的步骤。展示了onException如何触发来处理异常。 


在图5.5中，显示了JmsProducer如何抛出JmsException异常到Channel(错误处理器所在地)中。这个路由中定义了一个OnException，当JmsException抛出时，OnException做出反应，处理消息。GenerateFailureResponse processor生成了一个定制的错误消息，用于返回给调用者。因为OnException被配置为处理异常---handled(true)--Camel会停止路由消息，将错误信息返回给最初的消费者，即返回自定义回复消息。 

注意：OnException默认是不处理异常的，所以代码清单5.4使用handled(true)设置，让OnException处理异常。记住这一点是重要的，因为当你处理异常时，必须显式设置才会有效。处理异常，将不会继续从异常抛出节点往下路由。Camel将停止正常路由，继续进行OnException上的路由。如果你想忽略异常，继续进行正常路由，你必须使用continued(true)方法，此方法将在5.4.5节中讨论。 

在往下继续学习之前，让我们花点时间看看代码清单5.4对应的Spring XML形式，语法有一点不同，清单5.5： 

<camelContext xmlns="http://camel.apache.org/schemas/spring"> 

<onException> 

<exception>javax.jms.JmsException</exception> 

<handled><constant>true</constant></handled> 

<process ref="failureResponse"/> 

</onException> 

<route> 

<from uri="mina:tcp://0.0.0.0:4444?textline=true"/> 

<process ref="validateOrder"/> 

<to uri="jms:queue:order.status"/> 

<process ref="generateResponse"/> 

</route> 

</camelContext> 

<bean id="failureResponse" 

class="camelinaction.FailureResponseProcessor"/> 

<bean id="validateOrder" class="camelinaction.ValidateProcessor"/> 

<bean id="generateResponse" class="camelinaction.ResponseProcessor"/> 

注意onException是如何设置的---你必须在exception标签中定义这些异常处理，同样，handled(true)对应的代码有点长，你必须将其包含在<constant>表达式中。在下面的路由定义中没有其他区别。 

清单5.5中使用了一个自定义的Processor来生成错误响应。让我们来详细看下这一点。 


 5.4.4自定义异常处理方式 

假设您想返回一个自定义的失败消息，如清单5.5中，这意味着你不仅要知道问题本身信息还要知道当前传输的消息的细节信息。怎么做呢？

代码清单5.5展示了如何使用OnException来实现，清单5.6展示了错误处理器Processor的实现方式。

public class FailureResponseProcessor implements Processor {

public void process(Exchange exchange) throws Exception {

String body = exchange.getIn().getBody(String.class);

Exception e = exchange.getProperty(Exchange.EXCEPTION_CAUGHT,

Exception.class);

StringBuilder sb = new StringBuilder();

sb.append("ERROR: ");

sb.append(e.getMessage());

sb.append("\nBODY: ");

sb.append(body);

exchange.getIn().setBody(sb.toString());

}

}



首先，获取你需要的信息：消息体和异常。异常的获取是通过一个属性实现的，而不是使用exchange.getException，这看起来有点奇怪。你这样做是因为你使用onException处理了异常(清单5.5中)。那么，Camel将异常从Exchange对象中移到了Exchange.EXCEPTION_CAUGHT属性上。剩下的代码构建了自定义消息返回给调用者。

你可能想知道，在Camel进行错误处理时，是否设置了其他的属性。请见表5.5。但是从一个终端用户的角度看，只需注意表中的前两个属性。另外两个属性用于Camel内部(用于错误处理和路由引擎)。

使用FAILURE_ENDPOINT属性的例子是：当你使用接收列表(Recipient List)企业集成模式路由消息时，此模式向一系列动态endpoint上发送消息的拷贝。如果不用这个属性，你无法知道哪一个端点出错了。



值得注意的是,清单5.6中使用了Camel处理器,它强迫你依赖CamelAPI。您也可以使用一个bean,如下:

public class FailureResponseBean {

public String failMessage(String body, Exception e) {

StringBuilder sb = new StringBuilder();

sb.append("ERROR: ");

sb.append(e.getMessage());

sb.append("\nBODY: ");

sb.append(body);

return sb.toString();

}

}

正如您看到的,您可以使用Camel的参数绑定，声明你所需要的参数类型。第一个参数时消息体，第二个参数是exception。



5.4.5 忽略异常 

在5.4.3节中我们学习了onException如何处理异常，处理一个异常意味着Camel将停止正常路由。但是有些时候，你想做的只是捕获异常后继续路由。在Camel中要实现这一点，就要使用continued(true)方法来代替handled(true)方法。 

在代码清单5.4中，假设你想忽略路由中抛出的ValidationException异常，代码清单5.7展示了操作方式： 

onException(JmsException.class) 

.handled(true) 

.process(new GenerateFailueResponse()); 

onException(ValidationException.class) 

.continued(true); 

from("mina:tcp://0.0.0.0:4444?textline=true") 

.process(new ValidateOrderId()) 

.to("jms:queue:order.status") 

.process(new GenerateResponse()); 

正如你所看到的，你需要做的就是添加另一个onException，并设置continued(true)。 

注意：你不能在同一个onException中同时使用handled和continued方法；continued方法会自动启用handled方法。 

现在假设一条消息又一次到达了TCP端点，Camel应用路由此消息，但是此时，验证处理器Processor抛出了ValidationException异常。此种情况如图5.6所示。 

当ValidateProcessor抛出异常ValidationException,异常传播回Channel。此路由定义了一个onException(continued(true))，指示Channel继续路由消息。 

当消息到达下一个Channel，此Channel觉察不到已抛出异常(就像异常从未发生一样).这与你在5.4.3节使用handled(true)方法时所看到的有较大差别，handled(true)会引起处理中断，路由停止。 

我们已经学习了很多新知识，现在让我们看一个错误处理的例子，在实践中使用我们学过的知识。 

5.4.6 实现一个错误处理解决方案 

假设你的老板给你分配了新的任务。这一次，用于上传文件的远程HTTP服务器是不可靠的，老板希望在错误发生时，再次上传时，通过FTP上传到FTP服务器。 

你已经学习了Camel in Action，已经知道Camel对错误处理有扩展支持，你可以使用onException来实现这个需求。带着巨大的耐心，你打开了编辑器，修改了路由，如代码清单5.8所示： 


errorHandler(defaultErrorHandler() 

.maximumRedeliveries(5).redeliveryDelay(10000)); 

onException(IOException.class).maximumRedeliveries(3) 

.handled(true) 

.to("ftp://gear@ftp.rider.com?password=secret"); 


from("file:/rider/files/upload?delay=3600000") 

.to("http://rider.com?user=gear&password=secret"); 

代码清单中为路由添加了一个onException，告诉Camel，在IOException异常抛出时，应该尝试返还三次，每次间隔10秒。如果尝试完成后，仍然有错误，Camel将处理这个异常，将消息路由到FTP端点上。Camel的路由引擎的威力和灵活性在此处展现了出来！这个onException就是另一个路由，Camel将用这个路由替代原始路由。 

注意：在代码清单5.8中，只是在onException的尝试用尽后才会将消息路由到FTP端点。 

本书源码chapter5/usecase目录中有这个例子的源码，这个例子包含了一个服务器端和一个客户端，可以使用maven启动： 

mvn exec:java -PServer 

mvn exec:java -PClient 

其控制台输出的内容显示了下一步要做的。 

在本章结束之前，我们必须再学习一些错误处理的性质。虽然这些性质很少使用，但是当时需要进行细粒度控制时，这些特性就显现出了威力。 


5.5 其他错误处理的特性 

1、onWhen--Allows you to dictate when an exception policy is in use 

2、onRedeliver--在消息被返还之前允许你执行一些代码 

3、retryWhile--在运行时，允许你决定是继续返还还是放弃 


5.5.1 使用onWhen 

onWhen这个过滤谓词，允许进行更细粒度的控制，如何时onException应该被触发。 

假设你应用中的代码清单5.8出现了新问题。HTTP服务拒绝了上传的数据，并返回了HTTP500错误和一个常量文本"ILLEGAL DATA"。你的老板希望你处理这个异常，并把文件移到一个单独的文件夹，以便可以手工查看被拒绝的原因。 

首先，你需要确定什么时候发生了HTTP 500错误，错误中是否包含一个常量文本"ILLEGAL DATA"。你决定创建一个java方法来验证这一点。如代码清单5.9所示： 

public final class MyHttpUtil { 

public static boolean isIllegalDataError( 

HttpOperationFailedException cause) { 

int code = cause.getStatusCode(); 

if (code != 500) { 

return false; 

} 

return "ILLEGAL DATA".equals(cause.getResponseBody().toString()); 

} 

} 

当一个HTTP操作没有成功，Camel的HTTP组件将抛出org.apache.camel.component.http.HttpOperationFailedException异常，里面包含了失败信息。异常的getStatusCode()方法返回HTTP状态码。 


接着，你需要在你的路由中(清单5.8)使用这个工具类。但是首先，你添加了一个onException来处理HttpOperationFailedException异常，将消息路由到illegal文件夹： 

onException(HttpOperationFailedException.class) 

.handled(true) 

.to("file:/rider/files/illegal"); 

现在，当HttpOperationFailedException异常抛出时Camel会见消息移到ilegal文件夹。 

如果你能够在onException触发时进行更细粒度的控制就更好了。代码清单5.9如何与onException联合起来？ 

可能你已经才到了怎么做---没错，你可以使用onWhen谓词。你所需要做的就是在onException中插入onWhen方法： 

onException(HttpOperationFailedException.class) 

.onWhen(bean(MyHttpUtil.class, "isIllegalData")) 

.handled(true) 

.to("file:/acme/files/illegal"); 

Camel会适应你的POJO类并使用它。多亏Camel的强大参数绑定功能。这是一种开发应用的好方式，不与Camel API耦合。onWhen是一个普通的功能，在Camel的其他地方也存在，例如拦截器中和onCompletion，所以你可以在不同的场景中使用此技术。 


5.5.2 使用onRedeliver 

onRedeliver的作用是：在进行返还尝试之前，执行一些代码。这给了你进行个性化处理的机会。例如，你可以使用它给消息添加一个头部，告诉接收者这是一个返还尝试。onRedeliver使用一个org.apache.camel.Processor来存放你要执行的代码。 

OnRedeliver可以在错误处理器上配置，也可以在onException上配置，或者两者同时配置，如下： 

errorHandler(defaultErrorHandler() 

.maximumRedeliveries(3) 

.onRedeliver(new MyOnRedeliveryProcessor()); 

onException(IOException.class) 

.maximumRedeliveries(5) 

.onRedeliver(new MyOtherOnRedeliveryProcessor()); 


OnRedeliver也是有范围的，所以如果OnRedeliver配置在onException上，it overrules any onRedeliver set on the error handler. 

在Spring DSL中，OnRedeliver被配置为一个spring bean的引用： 

<onException onRedeliveryRef="myOtherRedelivery"> 

<exception>java.io.IOException</exception> 

</onException> 

<bean id="myOtherRedelivery" 

class="com.mycompany.MyOtherOnRedeliveryProceossor"/> 

5.5.3 使用retryWhile 

当你想对返还次数进行更细粒度的控制时，可以使用RetryWhile。他也是一个有范围的谓词，所以你可以在错误处理器和onException上定义。 

使用RetryWhile，你可以实现你自己的返还规则。代码清单5.10展示了实现的骨架代码： 

public class MyRetryRuleset { 

public boolean shouldRetry( 

@Header(Exchange.REDELIVERY_COUNTER) Integer counter, 

Exception causedBy) { 

... 

} 

当你使用MyRetryRuleset类，你可以实现你自己的逻辑，来决定是否继续返还。如果方法返回true，继续返还，如果返回fales，则放弃。 

使用你自定义个规则，可以在onException中配置retryWhile： 

onException(IOException.class).retryWhile(bean(MyRetryRuletset.class)); 

对应的Spring XML： 

<onException> 

<exception>java.io.IOException</exception> 

<retryWhile><method ref="myRetryRuleset"/></retryWhile> 

</onException> 

<bean id="myRetryRuleset" class="com.mycompany.MyRetryRuleset"/> 

