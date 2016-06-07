title: 依赖注入Dependency Injection(DI)
date: 2016-1-1
tags: [Java, DI] #文章标签，可空，多标签请用格式，注意:后面有个空格
---
依赖注入设计模式允许我们消除硬编码依赖，它能使我们的应用松耦合，易于扩展和维护。依赖注入可使依赖解决从编译器（compile-time）到运行期（runtime）。

## 一般实现
假设我们有一个应用使用`EmailService`发送邮件，代码实现如下：

```java
public class EmailService{
	public sendEmail(String msg, String receiver){
		// logic to send email
		System.out.println("Email send to " + receiver + " with message=" + msg);
	}
}
```

`EmailService`中包含一个方法用于发送邮件。我们的应用将会使用`EmailService`来发送邮件, 代码如下：

```java
public class Application{
	private EmailService emailService = new EmailService();
	public void sendMessage(String msg, String receiver){
		//here are some logic to process message
		emailService.sendEmail(msg, receiver);
	}
}
```
客户会使用`Application`完成发送消息的过程，代码如下：

```java
public class Main{
	public static void main(String[] args){
		Application myApplication = new Application();
		// send an email with body hello, recevier zhang san
		myApplication.sendMessage("hello", "zhang san");
	}
}
```
以上的代码实现了发送邮件的过程。代码运行良好，似乎没有什么问题。但是仔细分析之后，我们会发现代码有一些限制：

- 在`Application`中我们直接初始化`EmailService`, 这直接导致硬编码依赖（hard-code dependency）。如果将来我们想使用其他的`EmailService`，我们将不得不直接改变`Application`的代码。这导致我们的应用难以扩展，更糟糕的是，如果`EmailService`在多个地方使用的话，我们将不得不每个地方都进行更改。
- 测试会变得非常困难。因为`Application`直接使用真实地`EmailService`实例,这将导致我们无法在我们的测试中使用Mock类。
- 将来如果我们想使用其他发送消息的方式，如：SMS, Facebook Message等，我们将不得不增加新的`Application`。这会导致`Application`和客户端代码改变。

## 依赖注入实现
使用依赖注入Injection Dependency(DI)设计模式，可以解决以上问题。

- Service应被设计为Interface或者base class
- Application应根据Service接口进行设计
- 注入器完成Service和Application的初始化工作

首先我们定义一个Service接口`MessageService`:

```java
public interface MessageService{
	public void sendMessage(String msg, String receiver);
}
```
我们有两种实现方式：SMS和Email:

`EmailServiceImapl`实现如下：

```java
public class EmailServiceImpl implements MessageService{
	public void sendMessage(String msg, String receiver){
		System.out.println("Email sent to "+ receiver + " with Message="+msg);
	}
}
```
`SMSServiceImapl`实现如下：

```java
public class SMSServiceImpl implements MessageService{
	public void sendMessage(String msg, String receiver){
		System.out.println("SMS sent to "+ receiver + " with Message="+msg);
	}
}
```
Service部分完成，下面我们实现Application部分。首先我们定义一个接口（不是必须）：

```java
public interface Application{
	public void sendMessage(String msg, String receiver);
}
```
我们定义一个实现：

```java
public class MyApplication implements Application{
	private MessageService messageService;

	public MyApplication(MessageService messageService){
		this.messageService = messageService;
	}

	public void sendMessage(String msg, String receiver){
		// here are some logic to manipulate msg
		messageService.sendMessage(msg, receiver);
	}
}
```
在我们的实现中，我们的应用只是使用`MessageService`, 它并不对其进行初始化。这很好的实现了分离。这也使我们的测试更加容易。现在我们只剩下最后一步，实现一个注入器，完成`MessageService`的初始化并将其注入到`MyApplication`中。

```java
public class InjectorImpl{
	public getApplication(String name){
		if(name.toLowerCase().equals("email")){
			return new MyApplication(new EmailServiceImpl());
		} else if(name.toLowerCase().equals("sms")){
			return new MyApplication(new SMSServiceImpl());
		}
	}
}
```
这是一种简单的使用构造函数实现依赖注入的方式，它会根据服务的名字来返回Application实例。客户代码如下：

```java
public class Main{
	public static void main(String[] args){
		// send an email
		Application myApplication = InjectorImpl.getApplication("Email");
		myApplication.sendMessage("hello", "zhang san");
	}
}
```
另外一种实现依赖注入的方式是使用setter方式。

```java
public class MyApplication implements Application{
	private MessageService messageService;

	public MyApplication(){
	}

	public setMessageService(MessageService messageService){
		this.messageService = messageService;
	}

	public void sendMessage(String msg, String receiver){
		// here are some logic to manipulate msg
		messageService.sendMessage(msg, receiver);
	}
}
```

```java
public class InjectorImpl{
	public getApplication(String name){
	Application application = new MyApplication();
		if(name.toLowerCase().equals("email")){
			application.setMessageService(new EmailServiceImpl());
		} else if(name.toLowerCase().equals("sms")){
			application.setMessageService(new SMSServiceImpl());
		}

		return application;
	}
}
```
我们既可以通过构造函数来实现依赖注入，又可以通过setter方式实现。具体采用哪种方式根据你的具体需求。当你的应用必须需要一个service才能运行的情况下，则采用构造函数的方式。

## DI库
Spring, Guice, J2EE CDI是通过反射和java annotation的方式来实现依赖注入的库。它使依赖注入更加容易实现，我们只需要简单的配置便可以实现。在下一篇中我将会对Guice实现做出说明。
