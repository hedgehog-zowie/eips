## 第三章 camel中的数据转换

本章包括：

* 使用EIPs和Java转换数据
* 转换xml格式的数据
* 使用已知格式转换数据
* 转换为自定义的数据格式
* 理解camel的类型转换机制

### 3.1 概述

数据转换包含以下两种类型的转换：

* 数据格式转换

  消息体的格式从一种形式转换为另一种形式，例如：CSV转换为XML。

* 数据类型转换

  消息体的格式从一种类型转换为另一种类型，例如：java.lang.String转换为javax.jms.TextMessage。

六种典型的数据转换如下表：

| 转换                                    | 描述                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| 路由中的数据转换                        | 在路由中，你可以显示的进行数据转换，使用Message Translator或者Content Enricher企业集成模式。这样你可以使用java代码进行数据映射。 |
| 使用组件进行数据转换                    | Camel提供了一些组件用于数据转换，例如XSLT组件用于XML转换。   |
| 数据格式转换                            | Camel中的数据格式转换都是成对格式出现的，一种格式转换为另一种格式，反之亦然。 |
| 使用模板进行数据转换                    | Camel提供了一些组件用于模板转换，例如 Apache Velocity组件。  |
| 使用Camel的类型转换机制进行数据类型转换 | Camel有一个复杂的类型转换程序机制，此机制按需激活。这样方便了常见类型间的转换，如java.lang.Integer类型到java.lang.String类型的换行，java.io.File类型到java.lang.String类型间的转换。 |
| 组件适配器中的消息转换                  | Camel的许多组件适应各种常用的协议,因此,需要能够对这些协议的消息进行转换。通常这些组件结合使用自定义数据转换和类型转换器。这种情况无缝地发生,只有组件创建着需要考虑。 |

### 3.2 使用EIP和java代码进行数据转换 

#### 3.2.1 使用消息转换器模式 

Camel提供了三种方式来使用此模式：

1. 使用Processor 
2. 使用Bean 
3. 使用transform方法

#####使用Processor 

Processor是Camel中的一个接口使用Processor，此接口只有一个方法： 
public void process(Exchange exchange) throws Exception; 

从上述方法可以看出，Processor是一个可以直接操作Camel的Exchange对象的API。它让你可以访问所有在Camel的CamelContext中传输的部分。CamelContext对象可以通过Exchange的getCamelContext方法得到。 

如下代码使用Processor将一个自定义格式转换为CSV格式：

```java
import org.apache.camel.Exchange;
import org.apache.camel.Processor;
public class OrderToCsvProcessor implements Processor {
    public void process(Exchange exchange) throws Exception {
        String custom = exchange.getIn().getBody(String.class);
        String id = custom.substring(0, 9);
        String customerId = custom.substring(10, 19);
        String date = custom.substring(20, 29);
        String items = custom.substring(30);
        String[] itemIds = items.split("@");
        StringBuilder csv = new StringBuilder();
        csv.append(id.trim());
        csv.append(",").append(date.trim());
        csv.append(",").append(customerId.trim());
        for (String item : itemIds) {
            csv.append(",").append(item.trim());
        }
        exchange.getIn().setBody(csv.toString());
    }
}
```

首先从exchange中获取自定义格式的内容。它是String类型的，所以你传入了String参数，Exchange返回了String类型的结果。接着你从自定义格式中提取数据到本地变量。自定义格式的内容可以是任意的，但在这个例子中，它是一个定长的自定义格式。接着你通过构建一个以逗号分隔的字符串将自定义格式映射为CSV格式。最后，使用CSV格式的负载替换了自定义格式的负载。 

你可以在下列路由中使用上面的OrderToCsvProcessor： 

```java
from("quartz://report?cron=0+0+6+*+*+?")
    .to("http://riders.com/orders/cmd=received&date=yesterday")
    .process(new OrderToCsvProcessor())
    .to("file://riders/orders?fileName=report-${header.Date}.csv");
```

上述路由使用Quartz组件安排工作在6点每天运行一次。然后它调用返回自定义格式的HTTP服务来检索昨天收到的订单。接下来,它使用OrderToCSVProcessor从自定义格式映射到CSV格式将结果写入文件。 

等价的路由在Spring XML中配置如下： 

```xml
<bean id="csvProcessor" class="camelinaction.OrderToCsvProcessor"/>
<camelContext xmlns="http://camel.apache.org/schema/spring">
    <route>
        <from uri="quartz://report?cron=0+0+6+*+*+?"/>
        <to uri="http://riders.com/orders/cmd=received&amp;date=yesterday"/>
        <process ref="csvProcessor"/>
        <to uri="file://riders/orders?fileName=report-${header.Date}.csv"/>
    </route>
</camelContext>
```

> Tips:
>
> 如何使用Exchange的getOut和getIn方法 
>
>  Exchange定义了两个检索消息的方法：getIn和getOut。getIn方法返回传入的消息，getOut方法访问出站消息。 
>  有两种场景，Camel终端用户需要决定使用哪个方法： 
>
> 1. 只读场景，例如打印传入消息的日志； 
> 2. 可写的场景：例如转换消息格式的时候； 
>
>  在第二种场景中，你可能会使用getOut方法，这在理论上是没错的。但是实践中，这种方式有缺点：传入的消息headers 和 attachments将会丢失。这可能不是你想要的。所以你必须拷贝headers 和 attachments到出站消息中，这个步骤通常是冗长乏味的。变通的方式是使用getIn方法设置消息内容的变化，永远不要使用getOut。

##### 使用Bean 

使用bean进行数据转换是一个很好的实践，因为这种方式允许你使用任何你需要的java代码或者java类库。Camel对这点没有强加任何限制。Camel可以调用你开发的任意bean，甚至你可以使用已经存在的bean，而不需要重写或者重新编译他们。 

让我们使用 bean代替前面的那个Processor：

```java
public class OrderToCsvBean {
    public static String map(String custom) {
        String id = custom.substring(0, 9);  
        String customerId = custom.substring(10, 19);
        String date = custom.substring(20, 29);
        String items = custom.substring(30);
        String[] itemIds = items.split("@");
        StringBuilder csv = new StringBuilder();
        csv.append(id.trim());
        csv.append(",").append(date.trim());
        csv.append(",").append(customerId.trim());
        for (String item : itemIds) {
            csv.append(",").append(item.trim());
        }
        return csv.toString();
    }
}
```

相应的DSL和Sping XML如下：

```java
from("quartz://report?cron=0+0+6+*+*+?")
    .to("http://riders.com/orders/cmd=received&date=yesterday")
    .bean(new OrderToCsvBean())
    .to("file://riders/orders?fileName=report-${header.Date}.csv");
```

```xml
<bean id="csvBean" class="camelinaction.OrderToCsvBean"/>
<camelContext xmlns="http://camel.apache.org/schema/spring">
    <route>
        <from uri="quartz://report?cron=0+0+6+*+*+?"/>
        <to uri="http://riders.com/orders/cmd=received&amp;date=yesterday"/>
        <bean ref="csvBean"/>
        <to uri="file://riders/orders?fileName=report-${header.Date}.csv"/>
    </route>
</camelContext>
```

#####使用transform方法

TRANSFORM()是可以用在路由中的一个方法，可以用来进行消息转换。利用表达式，TRANSFORM()具有极大的灵活性，有时还可以节省时间。例如，假设你需要使用<br/>标记取代所有HTML格式数据中的的换行符。此时可以使用Camel内置的表达式和正则表达式的搜索和替换: 

```java
from("direct:start")
    .transform(body().regexReplaceAll("\n", "<br/>"))
    .to("mock:result");
```

> Tips: direct组件
>
> 这个例子中使用到了Direct组件作为路由的输入源（from("direct:start")）。Direct组件提供生产者和消费者之间的直接调用。但是调用只在同一个Camel应用中有效，所以外部系统不能直接给direct组件发消息。这个组件经常用于Camel应用中的路由连接或者路由测试。 

camel也支持自定义表达式：

```java
from("direct:start")
    .transform(new Expression() {
        public <T> T evaluate(Exchange exchange, Class<T> type) {
            String body = exchange.getIn().getBody(String.class);
            body = body.replaceAll("\n", "<br/>");
            body = "<body>" + body + "</body>";
            return (T) body;
        }
    })
    .to("mock:result");
```

在Spring XML中使用<transform>进行数据转换，首先得声明一个常规的bean用于消息转换：

```java
public class HtmlBean {
    public static String toHtml(String body) {
        body = body.replaceAll("\n", "<br/>");
        body = "<body>" + body + "</body>";
        return body;
    }
}
```

然后在xml配置文件中写： 

```xml
<bean id="htmlBean" class="camelinaction.HtmlBean"/>
<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring">
    <route>
        <from uri="direct:start"/>
        <transform>
            <method bean="htmlBean" method="toHtml"/>
        </transform>
        <to uri="mock:result"/>
    </route>
</camelContext>
```

#### 使用Content Enricher企业集成模式 

这种模式描述了这样一种场景，用来自另一个消息源返回的数据来丰富一个消息的数据。 

Camel提供了两个DSL操作用于实现这个模式： 

1. pollEnrich---这个操作使用一个消费者合并来自另一个源的数据 
2. enrich---这个操作使用一个生产者合并来自另一个源的数据 

pollEnrich和enrich两个操作的区别 :

两者的区别在于pollEnrich使用一个消费者合并来自另一个源的数据,而enrich使用一个生产者合并来自另一个源的数据.了解其中的不同是非常重要的：file组件可以使用这两个操作，但是使用enrich操作，会把信息写入到一个文件中；使用pollEnrich操作，将会读取文件内容作为消息源，这种情况非常类似上面那个场景。HTTP组件只能使用enrich操作，它允许您调用一个外部HTTP服务，使用它的返回结果作为源。 

### 3.4 数据格式转换

在Camel中，数据格式转换时可插拔的数据转换，可将消息从一种形式转换为另一种形式。Camel中，每一种数据格式转换都由接口org.apace.camel.spi.DataFormat代表，此接口包含两个方法： 

1. marshal方法：封存一个消息为另一种形式，比如封存java对象为XML, CSV, EDI, HL7等数据模型；即对象--->二进制 
2. unmarshal方法：进行一种反向操作，将某种格式的数据转换为消息，即二进制--->对象 

#### 3.4.1 camel提供的数据格式

| Data format   | Data model           | Artifact          | Description                                                  |
| ------------- | -------------------- | ----------------- | ------------------------------------------------------------ |
| Bindy         | CSV,FIX,fixed length | camel-bindy       | Binds various data models to model objects using annotations |
| Castor        | XML                  | camel-castor      | Uses Castor for XML binding to and from Java objects         |
| Crypto        | Any                  | camel-crypto      | Encrypts and decrypts data using the Java Cryptography Extension |
| CSV           | CSV                  | camel-csv         | Transforms to and from CSV using the Apache Commons CSV library |
| Flatpack      | CSV                  | camel-flatpack    | Transforms to and from CSV using the FlatPack library        |
| GZip          | Any                  | camel-gzip        | Compresses and decompresses files (compatible with the popular gzip/gunzip tools) |
| HL7           | HL7                  | camel-hl7         | Transforms to and from HL7, which is a well-known data format in the health care industry |
| JAXB          | XML                  | camel-jaxb        | Uses the JAXB 2.x standard for XML binding to and from Java objects |
| Jackson       | JSON                 | camel-jackson     | Transforms to and from JSON using the ultra-fast Jackson library |
| Protobuf      | XML                  | camel-protobuf    | Transforms to and from XML using the Google Protocol Buffers library |
| SOAP          | XML                  | camel-soap        | Transforms to and from SOAP                                  |
| Serialization | Object               | camel-core        | Uses Java Object Serialization to transform objects to and from a serialized stream |
| TidyMarkup    | HTML                 | camel-tagsoup     | Tidies up HTML by parsing ugly HTML and returning it as pretty well-formed HTML |
| XmlBeans      | XML                  | camel-xmlbeans    | Uses XmlBeans for XML binding to and from Java objects       |
| XMLSecurity   | XML                  | camel-xmlsecurity | Facilitates encryption and decryption of XML documents       |
| XStream       | XML                  | camel-xstream     | Uses XStream for XML binding to and from Java objects        |
| XStream       | JSON                 | camel-xstream     | Transforms to and from JSON using the XStream library        |
| Zip           | Any                  | camel-core        | Compresses and decompresses messages; it’s most effective when dealing with large XML- or text-based payloads |

#### 3.4.2 使用csv格式的数据

```java
from("file://rider/csvfiles")
    .unmarshal().csv()
    .split(body()).to("activemq:queue.csv.record");
```

在spring中使用有一点不同，如下：

```xml
<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring">
    <route>
        <from uri="file://rider/csvfiles"/>
        <unmarshal><csv/></unmarshal>
        <split>
            <simple>body</simple>
            <to uri="activemq:queue.csv.record"/>
        </split>
    </route>
</camelContext>
```

#### 3.4.3 使用二进制格式

```java
package camelinaction.bindy;
import java.math.BigDecimal;
import org.apache.camel.dataformat.bindy.annotation.CsvRecord;
import org.apache.camel.dataformat.bindy.annotation.DataField;
@CsvRecord(separator = ",", crlf = "UNIX")
public class PurchaseOrder {
    @DataField(pos = 1)
    private String name;
    @DataField(pos = 2, precision = 2)
    private BigDecimal price;
    @DataField(pos = 3)
    private int amount;
}
```

#### 3.4.4 使用json格式

JSON(JavaScript对象表示法)是一种数据交换格式,Camel提供了两个组件,支持JSON数据格式:camel-xstream和camel-jackson。

回到骑士汽车配件系统,你现在必须实现一个新的服务,此服务返回JSON格式的订单摘要。在Camel中实现这个服务是非常容易的。

```xml
<bean id="orderService" class="camelinaction.OrderServiceBean"/>
<camelContext id="camel" xmlns="http://camel.apache.org/schema/spring">
    <dataFormats>
        <json id="json" library="Jackson"/>
    </dataFormats>
    <route>
        <from uri="jetty://http://0.0.0.0:8080/order"/>
        <bean ref="orderService" method="lookup"/>
        <marshal ref="json"/>
    </route>
</camelContext>
```

首先你需要设置的JSON数据格式，指定使用Jackson类库。然后你定义一个路由,使用Jetty作为HTTP服务端点。这个服务将请求路由到orderService bean上面，调用bean的lookup方法。此方法返回的结果被marshal为json格式的数据，然后返回给HTTP客户端。 
 orderService bean可以有这样一个方法： 
 public PurchaseOrder lookup(@Header(name = "id") String id) 
 这个签名允许您实现查找逻辑。在4.5.3节中你会了解更多关于@Header注释。 
 注意这个bean方法返回一个POJO对象，JSON类库可以对这个POJO进行marshal，比如你使用了代码列表3.7中的PurchaseOrder对象，那么会生成下面的json字符串： 

 {"name":"Camel in Action","amount":1.0,"price":49.95} 
 HTTP服务本身可以被一个GET请求调用，使用id作为请求参数： 
[http://0.0.0.0](http://0.0.0.0/):8080/order/service?id=123. 

#### 3.4.6 自定义数据格式 

在项目中，一个经常要做的事情就是用自定义的数据格式进行数据转换。

开发自定义数据格式是非常简单的，因为你只需要实现Camel的一个接口：org.apache.camel.spi.DataFormat

### 3.6 关于camel类型转换器 

Camel提供了一个内置的类型转换系统,对常用类型之间进行自动转换。这个类型转换系统使Camel的各个组件直接可以很容易的协调工作，不会出现类型转换错误。从Camel用户的角度来看，在很多地方类型转换是内置在API中的，没有侵入性。例如：

```java
String custom = exchange.getIn().getBody(String.class);
```

getBody方法接收了你所期望返回的类型为参数。在底层，类型转换系统将返回的结果转换成了Sring类型。

#### 3.6.1 Camel类型转换器的运行机制

在Camel启动时，所有的类型转换器都会注册到TypeConverterRegistry中。在运行时，Camel使用TypeConverterRegistry的lookup方法查找一个合适的TypeConverter来使用：

TypeConverter lookup(Class<?> toType, Class<?> fromType);

使用TypeConverter的convertTo方法，Camel可以将一种类型转换为另一种类型：<T> T convertTo(Class<T> type, Object value);

> Tips: 
>
> Camel提供了150以上的开箱即用的类型转换器，涵盖了常用的类型转换。

##### 加载类型转换器到注册表中

Camel启动时，使用org.apache.camel.impl.converter.AnnotationTypeConverterLoader类通过扫描类路径的方式将所有的类型转换器加载到TypeConverterRegistry中。为了避免扫描数量巨大的类，Camel会读取META-INF文件夹中的一个服务发现文件：META-INF/services/org/apache/camel/TypeConverter.这是一个文本文件，包含一个java包列表，这些包中包含了Camel的类型转换器。利用这个特殊的文件，避免了扫描数量巨大的类，提高了性能。此文件告诉Camel哪个jar包中包含类型转换器。camel-core核心包中有这样一个文件，文件内容包含如下三个条目：

```java
org.apache.camel.converter
org.apache.camel.component.bean
org.apache.camel.component.file
```

AnnotationTypeConverterLoader类将会扫描这些包以及子包，寻找有`@Converter`的类以及这样的类中有`@Converter`的公有方法。每一个这样的方法被称为一个类型转换器。

例如：下面的代码是camel-core包中的IOConverter类的代码片段：

```java
@Converter
public final class IOConverter {
    @Converter
    public static InputStream toInputStream(URL url) throws IOException {
        return url.openStream();
    }
}
```

Camel会看每一个@Converter注解的方法的签名。方法的第一个参数是from(输入消息)的类型，返回的类型是to(输出消息)的类型。在这个例子中，这个转换器可以将URL类型转换为InputStream类型。

#### 使用Camel类型转换器

如前所述，Camel类型转换器被广泛使用，而且常常自动起作用。你可能要在一个路由中使用它们转换一个特定的类型，假设你需要将一些文件路由到JMS队列，并且使用javax.jmx.TextMessage类型的消息。你可以将每一个文件转换为String类型，迫使JMS组件使用TextMessage，Camel是很容易做到的---使用convertBodyTo方法，如下：

```java
 from("file://riders/inbox")
     .convertBodyTo(String.class)
     .to("activemq:queue:inbox");
```

如果使用Spring XML，如下：

```xml
<route>
    <from uri="file://riders/inbox"/>
    <convertBodyTo type="java.lang.String"/>
    <to uri="activemq:queue:inbox"/>
</route>
```

你可以省略java.lang.前缀：

<convertBodyTo type="String"/>.

如果想使用某一特定的编码读取文件：

```java
from("file://riders/inbox")
    .convertBodyTo(String.class, "UTF-8")
    .to("activemq:queue:inbox");
```

提示：如果你对路由中的负载或者负载类型比较迷惑，可以试着在路由开始增加.convertBodyTo(String.class)部分，将负载转换为String类型，如果转换失败，抛出NoTypeConversionAvailableException异常。

这就是在路由中使用Camel类型转换器的方法。

#### 3.6.3 自定义类型转换器

在Camel中编写自己的类型转换器是很容易的。在3.6.1节你已经看到类型转换器是什么样子.假设你想写一个converter用于将一个byte[]转换为PurchaseOrder对象(代码列表3.7中的一个对象)。那么，你需要创建一个@Converter注解的类，类中包含转换方法：

```java
@Converter
public final class PurchaseOrderConverter
    @Converter
    public static PurchaseOrder toPurchaseOrder(byte[] data, Exchange exchange) {
        TypeConverter converter = exchange.getContext().getTypeConverter();
        String s = converter.convertTo(String.class, data);
        if (s == null || s.length() < 30) {
            throw new IllegalArgumentException("data is invalid");
        }
        s = s.replaceAll("##START##", "");
        s = s.replaceAll("##END##", "");
        String name = s.substring(0, 9).trim();
        String s2 = s.substring(10, 19).trim();
        BigDecimal price = new BigDecimal(s2);
        price.setScale(2);
        String s3 = s.substring(20).trim();
        Integer amount = converter
            .convertTo(Integer.class, s3);
        return new PurchaseOrder(name, price, amount);
	}
}
```

代码中，通过Exchange获取CamelContext，进而获取上级TypeConverter，用其进行String和byte转换。

现在你所需要做的是添加服务发现文件,文件名为：TypeConverter，位于META-INF目录中。正如前面所解释的那样,这个文件包含一行内容，用于Camel扫描这个包进而发现@Converter注解类。内容为：`camelinaction`。

