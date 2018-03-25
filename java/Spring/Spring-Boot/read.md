

@RestController
@RequestMapping


@EnableAutoConfiguration
此注解告诉Spring Boot根据您添加的jar包依赖来“猜测”您将如何配置Spring


@SpringBootApplication 注解相当于使用 @Configuration， @EnableAutoConfiguration 和 @ComponentScan



1，可以定制定制 SpringApplication
2，可以添加自定义的FailureAnalyzers 用于处理启动异常打印的日志信息

3，自定义启动监听器：

23.5 应用程序事件与监听器
除了常见的Spring Framework事件，比如 ContextRefreshedEvent，SpringApplication 还会发送一些其他应用程序事件。

[Note]
在创建 ApplicationContext 之前，实际上触发了一些事件，因此您不能在 @Bean 上注册一个监听器。您可以通过 SpringApplication.addListeners(​) 或者 SpringApplicationBuilder.listeners(​) 方法注册它们。
如果您希望自动注册这些监听器，无论应用创建的方式如何，您都可以将 META-INF/spring.factories 文件添加到项目中，并使用 org.springframework.context.ApplicationListener 键引用您的监听器。
org.springframework.context.ApplicationListener=com.example.project.MyListener
当您运行应用时，应用程序事件将按照以下顺序发送：

在开始运行时，ApplicationStartingEvent 在监听器注册和初始化之后被发送。
当 ApplicationEnvironmentPreparedEvent 发现 Environment 被上下文使用时，将被发送，但是在上下文被创建之前。
ApplicationPreparedEvent 在启动刷新之前，bean定义被加载之后被发送。
在刷新并且处理了用于指示应用准备处理请求的相关回调之后，ApplicationReadyEvent 将被发送。
如果启动时发生异常，ApplicationFailedEvent 将被发送。
