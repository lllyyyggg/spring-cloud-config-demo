对于配置的重要性，我想我不用进行任何强调，大家都可以明白其重要性。在普通单体应用，我们常使用配置文件(application(*).properties(yml))管理应用的所有配置。这些配置文件在单体应用中非常胜任其角色，并没有让我们感觉到有头疼的地方。但随着微服务框架的引入，微服务数量就会在我们产品中不断增加，之前我们重点考虑的是系统的可伸缩、可扩展性好，但随之就是配置管理的问题就会一一暴露出来。起初微服务器各自管各自的配置，在开发阶段并没什么问题，但到了生产环境管理就会很头疼，如果要大规模更新某项配置，困难就可想而知。


为此，在分布式系统中，Spring Cloud提供一个Config子项目，该项目核心就是配置中心，通过一个服务端和多个客户端实现配置服务。我们可使用配置服务器集中的管理所有服务的各种环境配置文件。配置服务中心默认采用Git的方式进行存储，因此我们很容易部署修改，并可以对环境配置进行版本管理。

Spring Cloud Config具有中心化、版本控制、支持动态更新和语言独立等特性。其特点是:

* 提供服务端和客户端支持(Spring Cloud Config Server和Spring Cloud Config Client);
* 集中式管理分布式环境下的应用配置;
* 基于Spring环境，实现了与Spring应用无缝集成;
* 可用于任何语言开发的程序;
* 默认实现基于Git仓库(也支持SVN)，从而可以进行配置的版本管理;

Spring Cloud Config的结构图如下:

![](https://upload-images.jianshu.io/upload_images/1488771-197762860d5c607f.png?imageMogr2/auto-orient/)

从图中可以看出Spring Cloud Config有两个角色(类似Eureka): `Server`和`Client`。`Spring Cloud Config Server`作为配置中心的服务端承担如下作用:

* 拉取配置时更新Git仓库副本，保证是配置为最新;
* 支持从yml、json、properties等文件加载配置;
* 配合Eureke可实现服务发现，配合Cloud Bus(这个后面我们在详细说明)可实现配置推送更新;
* 默认配置存储基于Git仓库(可以切换为SVN)，从而支持配置的版本管理.

而对于，Spring Cloud Config Client则非常方便，只需要在启动配置文件中增加使用Config Server上哪个配置文件即可。


example:

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
```		

然后使用`@EnableConfigServer`来开启Config Server。

然后配置`application.yml`

```
server:
  port: 8888
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/lanyage/spring-cloud-configsample
          username:
          password:
          search-paths: '{profile}'     
```

然后再在git里面创建一个配置文件。

`name=product-service`

然后开启`ConfigServer`。

然后在浏览器里面获取`http://localhost:8888/application/dev/master`获取。

获取到json

```
{
	name: "application",
	profiles: [
	"dev"
	],
	label: "master",
	version: "1e31656564caafac0e8735f8fbc6b1cd1f2087c6",
	state: null,
	propertySources: [
		{
			name: "https://github.com/lanyage/spring-cloud-configsample/dev/application.yml",
			source: {
				user.username: "dev/lyg",
				user.sex: "男",
				user.age: 18
			}
		}
	]
}
```

匹配规则。

http请求地址和资源文件映射如下: 

* /{application}/{profile}[/{label}] 
* /{application}-{profile}.yml 
* /{label}/{application}-{profile}.yml 
* /{application}-{profile}.properties 
* /{label}/{application}-{profile}.properties

> 创建Config Client

```
 <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```
然后在bootstrap.properties文件中配置如下：

```
server.port=8082
#spring.application.name=config-client
spring.application.name=application
#spring.application.name=USER-SERVICE

# 仓库的分支
spring.cloud.config.label=master
spring.cloud.config.uri=http://localhost:8888/
spring.cloud.config.profile=dev
```
编写controller.

```
@Value("${user.username}")
private String username;
@RequestMapping(value = "/value")
public String value() {
    return username;
}
```
启动。

访问。

`http://localhost:8082/value`

结果。

`dev/lyg`

这里有一个需要注意的是。文件名必须是`{spring.application.name}-{profile}.yml`或者`/dev/{spring.application.name}.yml`

> spring配置文件的优先级。

命令行 > system > 外部特定(dev)配置文件 > 外部配置文件 > @ConfigurationSource或者@ConfigurationProperties的配置文件>默认配置。

> 配置规则详解。

下面我们来看一看Config Client从Config Server中获取配置数据的流程:

* Config Client启动时，根据bootstrap.properties中配置的应用名称(application)、环境名(profile)和分支名(label)，向Config Server请求获取配置数据;
* Config Server根据Config Client的请求及配置，从Git仓库(这里以Git为例)中查找符合的配置文件;
* Config Server将匹配到的Git仓库拉取到本地，并建立本地缓存;
* Config Server创建Spring的ApplicationContext实例，并根据拉取的配置文件，填充配置信息，然后将该配置信息返回给Config Client;
* Config Client获取到Config Server返回的配置数据后，将这些配置数据加载到自己的上下文中。同时，因为这些配置数据的优先级高于本地Jar包中的配置，因此将不再加载本地的配置。


> 而Config-Client的bootstrap.properties配置对应如下:

* spring.application.name <==> application;
* spring.cloud.config.profile <==> profile;
* spring.cloud.config.label <==> label.

>  Git仓库配置

Config Server默认使用的就是Git，所以配置也非常简单，如上面的配置

```
spring.cloud.config.server.git.uri=http://
spring.cloud.config.server.git.username=username
spring.cloud.config.server.git.password=password
```

使用占位符

在服务端配置中我们也可以使用{application}、{profile} 和 {label}占位符，如下:

```
spring.cloud.config.server.git.uri=http://github.com/cd826/{application}
spring.cloud.config.server.git.username=username
spring.cloud.config.server.git.password=password
```
这样，我们就可以为每一个应用客户端创建一个单独的仓库。


使用模式匹配

我们也可以使用{application}/{profile}进行模式匹配，以便获取到相应的配置文件。配置示例如下:

```
pring.cloud.config.server.git.uri=https://github.com/spring-cloud-samples/config-repo

spring.cloud.config.server.git.repos.simple=https://github.com/simple/config-repo

spring.cloud.config.server.git.repos.special.pattern=special*/dev*,*special*/dev*
spring.cloud.config.server.git.repos.special.uri=https://github.com/special/config-repo

spring.cloud.config.server.git.repos.local.pattern=local*
spring.cloud.config.server.git.repos.local.uri=file:/home/configsvc/config-repo
```
如果模式中需要配置多个值，那么可以使用逗号分隔。

如果{application}/{profile}没有匹配到任何资源，则使用spring.cloud.config.server.git.uri配置的默认URI。

当我们使用yml类型的文件进行配置时，如果模式属性是一个YAML数组，也可以使用YAML数组格式来定义。这样可以设置成多个配个配置文件，如:

```
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
          repos:
            development:
              pattern:
                - */development
                - */staging
              uri: https://github.com/development/config-repo
            staging:
              pattern:
                - */qa
                - */production
              uri: https://github.com/staging/config-repo
```

搜索目录

当我们把配置文件存放在Git仓库中子目录中时，可以通过设置search-path来指定该目录。同样，search-path也支持上面的占位符。示例如下:

```
spring.cloud.config.server.git.uri=https://github.com/spring-cloud-samples/config-repo
spring.cloud.config.server.git.searchPaths=foo,bar*
```
这样系统就会自动搜索foo的子目录，以及以bar开头的文件夹中的子目录。

本地缓存

当Config-Server从Git(或SVN)中获取了配置信息后，将会在本地的文件系统中存储一份。默认将存储在系统临时目录下，并且以config-repo-作为开头，在Linux系统中默认存储的目录为/tmp/config-repo-<randomid>。Config-Server将配置信息存储在本地可以有效的防止当Git仓库出现故障而无法访问的问题，当Config-Server无法访问到Git仓库时就会读取之前存储在本地文件中的配置，然后将这些配置信息返回给Config-Client。比如，当我们断开网络进行测试，当我们启动Config-Server时会在控制台中看到以下输出：

```
17:18:01 285 [http-nio-8280-exec-1] WARN  o.s.c.c.s.e.MultipleJGitEnvironmentRepository - Could not fetch remote for master remote: https://github.com/cd826/SpringcloudSamplesConfig
17:18:01 660 [http-nio-8280-exec-1] INFO  o.s.c.a.AnnotationConfigApplicationContext - Refreshing 
17:18:01 710 [http-nio-8280-exec-1] INFO  o.s.b.f.a.AutowiredAnnotationBeanPostProcessor - JSR-330 'javax.inject.Inject' annotation found and supported for autowiring
17:18:01 782 [http-nio-8280-exec-1] INFO  o.s.c.c.s.e.NativeEnvironmentRepository - Adding property source: file:/Users/cd826/MyWork/springcloud-sample-projects/config/config-server/tmp/mallWeb-dev.properties
17:18:01 782 [http-nio-8280-exec-1] INFO  o.s.c.c.s.e.NativeEnvironmentRepository - Adding property source: file:/Users/cd826/MyWork/springcloud-sample-projects/config/config-server/tmp/mallWeb.properties
```

Spring Cloud 官方文档建议我们在Config-Server中指定本地文件路径，以避免出现不可预知的错误。可以使用下面的属性配置来指定本地文件路径:

```
## Git仓库
spring.cloud.config.server.git.basedir=tmp/

## SVN仓库
spring.cloud.config.server.svn.basedir=tmp/
```

> 安全保护

对于我们存储在配置中心的一些配置内容，总会有一些是敏感信息，比如数据库连接的用户名和密码，你总不能直接裸奔吧，所以我们还是需要对Config-Server做一些安全控制。当然，对于Config-Server的安全控制有很多种，比如：物理网络限制、OAuth2授权等。但是，在这里因为我们使用的是SpringBoot，所以使用SpringSecurity会更容易也更简单。这时候，我们只需要在Config-Server中增加如下依赖:

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
此时，当我们启动Config-Server时，SpringSecurity会默认为我们生产一个访问密码，这种方式常常不是我们需要的，所以一般我们还需要在配置文件中配置用户名和密码，比如：

```
security.user.name=cd826
security.user.password=pwd
```
这样，当我们需要访问Config-Server时就会弹出用户认证对话框。此时，对于Config-Client我们需要在配置文件中增加用户和访问口令的配置，如下：

```
spring.cloud.config.username=cd826
spring.cloud.config.password=pwd
```

加密与解密

访问安全是对整体的控制，多数情况下我们还需要对敏感内容加密后存储，比如之前所说的数据库访问的用户名称和登录口令。很幸运，Spring Cloud Config为我们提供相应的支持。

Spring Cloud Config提供了两种加解密方式: 1)对称加密; 2)非对称加密。在描述如何使用之前，我们先看看一些使用前提。

安装JCE(Java Cryptography Extension)

Spring Cloud Config所提供的加解密依赖JCE，因为，JDK中没有默认提供，所以我们需要先安装JCE。安装方法也比较简单，就是下载相应的Jar包，然后把这些包替换$JDK_HOME/jar/lib/security相应的文件，对于JDK8下载地址为:[JCE下载](https://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)

加解密端点

另外，Spring Cloud Config还提供了两个端点进行加密和解密，如下：

/encrypt: 加密端点，使用格式如下: curl $CONFIG_SERVER/encrypt -d 所要加密的内容
/decrypt: 解密端点，使用格式如下: curl $CONFIG_SERVER/decrypt -d 所要解密的内容

```
注意：当你测试中所加解密中包含特殊字符时，需要进行URL编码，这时候你需要使用--data-urlencode而不是-d.
```

对称加密

对称加解密的配置非常简单。我们只需要在配置文件中增加加解密所使用的密钥即可，如:

`encrypt.key=cd826_key`

配置好之后，你可以启动Config-Server，并使用上面所说的端点进行加解密测试。

对于，配置文件我们需要为加密的内容增加一个{cipher}前导符。如：

spring.datasource.username=dbuser
spring.datasource.password={cipher}FKSAJDFGYOS8F7GLHAKERGFHLSAJ

但是，如果你使用的配置文件是yml格式，那么需要使用单引号把加密内容引起来，如下：

```
spring:
    datasource:
        username:dbuser
        password:'{cipher}FKSAJDFGYOS8F7GLHAKERGFHLSAJ'
```

非对称加密

非对称加密相对于对称加密来说复杂了一些，首先我们需要借助Java的keytool生成密钥对，然后创建Key Store并复制到服务器目录下。对于keytool的使用可以参考这里:

使用keytool生成Key Store，命令如下:

```
$ keytool -genkeypair -alias tsfjckey -keyalg RSA \
  -dname "CN=Mall Web,OU=TwoStepsFromJava,O=Organization,L=city,S=province,C=china" \ 
  -keypass javatwostepsfrom -keystore server.jks -storepass twostepsfromjava
```

将所生成server.jks拷贝到项目的resources目录下(Config-Server)。

修改配置文件:

```
encrypt.key-store.location=server.jks
encrypt.key-store.alias=tsfjckey
encrypt.key-store.password=twostepsfromjava
encrypt.key-store.secret=javatwostepsfrom
```
非对称加密相对于对称加密来说配置也复杂，但安全性也会高很多。

使用多个加密Key

也许，我们需要对不同的敏感信息使用不同的加密key，这时候我们的配置文件只需要按如下进行编写:

`oo.bar={cipher}{key:testkey}...`

Config-Server在解密的时候就会尝试从配置文件中获取testkey的做为密钥。

> 高可用配置

整合Eureka

看到这里，可能有些童鞋已经发现，我们在Config-Client中配置config.uri时使用的具体的地址，那么是否可以使用之前的Eureka呢？答案是肯定，我们可以把Config-Server和其它微服务一样作为一个服务基本单元。我们只需要进行如下修改即可。


```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

在配置文件中配置我们服务的名称，及之前我们所编写Eureka服务器的地址

```
spring.application.name=config-server
eureka.client.service-url.defaultZone=http://localhost:8260/eureka
```

启动类:

```
@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

Config-Client改造

在pom.xml中增加如下依赖:

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

配置文件修改，注意这里的配置文件为:bootstrap.properties:

```
spring.application.name=mallWeb
spring.cloud.config.profile=dev

eureka.client.service-url.defaultZone=http://localhost:8260/eureka

spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.service-id=config-server
```

修改启动类:

```
@EnableDiscoveryClient
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

这样就完成了从Eureka中获取Config-Server的服务信息。

这里最重要的就是在配置中增加: `spring.cloud.config.discovery.enabled=true`，并将原来所配置的`spring.cloud.config.uri`，修改为`spring.cloud.config.discovery.service-id`。

> 快速失败与响应

开启Config-Server启动加载

默认情况下，只有当客户端请求时服务端才会从配置的Git仓库中进行加载，我们可以通过设置`clone-on-start`，让服务端在启动时就会加载。

```
spring.cloud.config.server.git.uri=https://git/common/config-repo.git

spring.cloud.config.server.git.repos.team-a.pattern=team-a-*
spring.cloud.config.server.git.repos.team-a.clone-on-start=true
spring.cloud.config.server.git.repos.team-a.uri=http://git/team-a/config-repo.git

spring.cloud.config.server.git.repos.team-b.pattern=team-b-*
spring.cloud.config.server.git.repos.team-b.clone-on-start=false
spring.cloud.config.server.git.repos.team-b.uri=http://git/team-b/config-repo.git

spring.cloud.config.server.git.repos.team-c.pattern=team-c-*
spring.cloud.config.server.git.repos.team-c.uri=http://git/team-a/config-repo.git
```

上面的配置，对于team-a的则在Config-Server启动时就会加载相应的配置，而对于其它则不会。当然，我们可以通过设置spring.cloud.config.server.git.clone-on-start的值来进行全局配置。

开启Config-Client快速失败

在一些情况下，我们希望启动一个服务时无法连接到服务端能够快速返回失败，那么可以通过下面的配置来设置:

`spring.cloud.config.fail-fast=true`

设置Config-Client重试

如果在启动时Config-Server碰巧不可以使用，你还想后面再进行重试，那么我们开始开启Config-Client的重试机制。首先，我们需要配置：

`spring.cloud.config.fail-fast=true`

然后，我们需要在我们的的依赖中增加：

```
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
这样，我们就可以为Config-Client开启了重试机制，当启动连接Config-Server失败时，Config-Client会继续尝试连接Config-Server，默认会尝试连接6次，时间间隔初始为1000毫秒，后面每次尝试连接会按照1.1倍数增加尝试连接时间的间隔，如果最后还不能够连接到Config-Server才会返回错误。我们可以通过在配置文件中复写`spring.cloud.config.retry.*`来进行相关配置。

如果你想全权控制重试机制，可以实现一个类型为:RetryOperationsInterceptor的类，并把bean的id设置为:`configServerRetryInterceptor`。

动态刷新配置

Config-Client中提供了一个refresh端点来实现配置文件的刷新。要想使用该功能，我们需要在Config-Client的pom.xml文件中增加以下依赖:

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

这样，当修改配置文件并提交到Git仓库后，就可以使用:http://localhost:8080/refresh刷新本地的配置数据。

> 但是，最好的方式还是和Spring Cloud Bus进行整合，这样才能实现配置的自动分发，而不是需要手工去刷新配置。




