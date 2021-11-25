# bootstrap和application

> 微服务中bootstrap和application之间的区别
>
> SpringBoot中有以下两种配置文件(properties和yml)两种

- 加载顺序上的区别

  - bootstrap配置文件先加载
  - application配置文件后加载

  > bootstrap.yml用于应用程序上下文的引导阶段,由父Spring ApplicationContext加载.父ApplicationContext被加载到使用application.yml的之前
  >
  > 在Spring Boot中有两种上下文,一种是bootstrap,另一种是application.bootstrap是应用程序的父上下文,也就是说bootstrap加载优先于application.bootstrap主要用于从额外的资源加载配置信息,还可以在本地外部配置文件中解密属性
  >
  > 这两个上下文公用一个环境,他是任何Spring应用程序的外部属性的来源,bootstrap里面的属性会优先加载,他们默认也不能被本地相同配置覆盖

