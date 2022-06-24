### 概述
Starter是Spring Boot的四大核心功能特性之一，除此之外，Spring Boot还有自动装配、Actuator监控等特性。  

Spring Boot里面的这些特性，都是为了让开发者在开发基于Spring生态下的企业级应用时，只需要关心业务逻辑，减少对配置和外部环境的依赖。其中，Starter是启动依赖，它的主要作用有几个。  

### 作用
1. Starter组件以功能为纬度，来维护对应的jar包的版本依赖，使得开发者可以不需要去关心这些版本冲突这种容易出错的细节。
2. Starter组件会把对应功能的所有jar包依赖全部导入进来，避免了开发者自己去引入依赖带来的麻烦。
3. Starter内部集成了自动装配的机制，也就说在程序中依赖对应的starter组件以后，
	这个组件自动会集成到Spring生态下，并且对于相关Bean的管理，也是基于自动装配机制来完成。
4. 依赖Starter组件后，这个组件对应的功能所需要维护的外部化配置，会自动集成到Spring Boot里面，
	我们只需要在application.properties文件里面进行维护就行了，比如Redis这个starter，只需要在application.properties
	文件里面添加redis的连接信息就可以直接使用了。

### 总结
Starter组件几乎完美的体现了Spring Boot里面约定优于配置的理念。另外，Spring Boot官方提供了很多的Starter组件，比如Redis、JPA、MongoDB等等。但是官方并不一定维护了所有中间件的Starter，所以对于不存在的Starter，第三方组件一般会自己去维护一个。
官方的starter和第三方的starter组件，最大的区别在于命名上。

1. 官方维护的starter的以spring-boot-starter开头的前缀。  
2. 第三方维护的starter是以spring-boot-starter结尾的后缀。   

这也是一种约定优于配置的体现。  
