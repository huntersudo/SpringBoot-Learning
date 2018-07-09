#chapter4-5-1：实现邮件发送：简单邮件、附件邮件、嵌入资源的邮件、模板邮件

相信使用过Spring的众多开发者都知道Spring提供了非常好用的JavaMailSender接口实现邮件发送。
在Spring Boot的Starter模块中也为此提供了自动化配置。下面通过实例看看如何在Spring Boot中使用JavaMailSender发送邮件。

## 快速入门

在Spring Boot的工程中的pom.xml中引入spring-boot-starter-mail依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

如其他自动化配置模块一样，在完成了依赖引入之后，只需要在application.properties中配置相应的属性内容。

下面我们以QQ邮箱为例，在application.properties中加入如下配置（注意替换自己的用户名和密码）：
```
spring.mail.host=smtp.qq.com
spring.mail.username=用户名
spring.mail.password=密码
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true
```

通过单元测试来实现一封简单邮件的发送：
```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = Application.class)
public class ApplicationTests {

	@Autowired
	private JavaMailSender mailSender;

	@Test
	public void sendSimpleMail() throws Exception {
		SimpleMailMessage message = new SimpleMailMessage();
		message.setFrom("dyc87112@qq.com");
		message.setTo("dyc87112@qq.com");
		message.setSubject("主题：简单邮件");
		message.setText("测试邮件内容");

		mailSender.send(message);
	}

}
```

到这里，一个简单的邮件发送就完成了，运行一下该单元测试，看看效果如何？

由于Spring Boot的starter模块提供了自动化配置，所以在引入了spring-boot-starter-mail依赖之后，会根据配置文件中的内容去创建JavaMailSender实例，
因此我们可以直接在需要使用的地方直接@Autowired来引入邮件发送对象。

## 进阶使用

在上例中，我们通过使用SimpleMailMessage实现了简单的邮件发送，但是实际使用过程中，我们还可能会带上附件、或是使用邮件模块等。
这个时候我们就需要使用MimeMessage来设置复杂一些的邮件内容，下面我们就来依次实现一下。

### 发送附件
在上面单元测试中加入如下测试用例（通过MimeMessageHelper来发送一封带有附件的邮件）：

```
@Test
public void sendAttachmentsMail() throws Exception {

	MimeMessage mimeMessage = mailSender.createMimeMessage();

	MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
	helper.setFrom("dyc87112@qq.com");
	helper.setTo("dyc87112@qq.com");
	helper.setSubject("主题：有附件");
	helper.setText("有附件的邮件");

	FileSystemResource file = new FileSystemResource(new File("weixin.jpg"));
	
	helper.addAttachment("附件-1.jpg", file);
	helper.addAttachment("附件-2.jpg", file);

	mailSender.send(mimeMessage);

}
```

## 嵌入静态资源
除了发送附件之外，我们在邮件内容中可能希望通过嵌入图片等静态资源，让邮件获得更好的阅读体验，而不是从附件中查看具体图片，
 下面的测试用例演示了如何通过MimeMessageHelper实现在邮件正文中嵌入静态资源。

```
@Test
public void sendInlineMail() throws Exception {

	MimeMessage mimeMessage = mailSender.createMimeMessage();

	MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
	helper.setFrom("dyc87112@qq.com");
	helper.setTo("dyc87112@qq.com");
	helper.setSubject("主题：嵌入静态资源");
	helper.setText("<html><body><img src=\"cid:weixin\" ></body></html>", true);

	FileSystemResource file = new FileSystemResource(new File("weixin.jpg"));
	
	helper.addInline("weixin", file);

	mailSender.send(mimeMessage);

}
```

这里需要注意的是addInline函数中资源名称weixin需要与正文中cid:weixin对应起来

## 模板邮件

通常我们使用邮件发送服务的时候，都会有一些固定的场景，比如重置密码、注册确认等，给每个用户发送的内容可能只有小部分是变化的。
 所以，很多时候我们会使用模板引擎来为各类邮件设置成模板，这样我们只需要在发送时去替换变化部分的参数即可。

在Spring Boot中使用模板引擎来实现模板化的邮件发送也是非常容易的，下面我们以velocity为例实现一下。
```
引入velocity模块的依赖：

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-velocity</artifactId>
</dependency>
```
在resources/templates/下，创建一个模板页面template.vm：
```
<html>
<body>
    <h3>你好， ${username}, 这是一封模板邮件!</h3>
</body>
</html>
```
我们之前在Spring Boot中开发Web应用时，提到过在Spring Boot的自动化配置下，模板默认位于resources/templates/目录下

最后，我们在单元测试中加入发送模板邮件的测试用例，具体如下：
```
@Test
public void sendTemplateMail() throws Exception {

	MimeMessage mimeMessage = mailSender.createMimeMessage();

	MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);
	helper.setFrom("dyc87112@qq.com");
	helper.setTo("dyc87112@qq.com");
	helper.setSubject("主题：模板邮件");

	Map<String, Object> model = new HashedMap();
	model.put("username", "didi");
	String text = VelocityEngineUtils.mergeTemplateIntoString(
			velocityEngine, "template.vm", "UTF-8", model);
	helper.setText(text, true);

	mailSender.send(mimeMessage);
}
```
尝试运行一下，就可以收到内容为你好， didi, 这是一封模板邮件!的邮件。这里，我们通过传入username的参数，在邮件内容中替换了模板中的${username}变量。

完整示例：Chapter4-5-1
