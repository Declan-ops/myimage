# SpringBoot

## 原理初探

自动装配

@RestController

pom.xml

* spring-boot-dependencies :核心依赖在父工程中!

* 我们在写或者引入一些SPringboot依赖的时候，不需要指定版本，就因为有这些版本仓库



```java
List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
```

获取候选配置文件

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
   List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
         getBeanClassLoader());
   Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
         + "are using a custom packaging, make sure that file is correct.");
   return configurations;
}
```

自动配置的核心文件

```
META-INF/spring.factories
```

![image-20210815202711469](C:\Users\Declan\AppData\Roaming\Typora\typora-user-images\image-20210815202711469.png)





结论:` springboot`所有自动配置都是在启动的时候扫描并加载:` spring.factories`所有的自动配置类都在这里面，但是不一定生效，要

判断条件是否成立，只要导入了对应的start，就有对应的启动器了，有了启动器，我们自动装配就会生效，然后就配置成功!

1.`springboot`在启动的时候，从类路径下`/META-INF/ spring.factories`获取指定的值;

⒉将这些自动配置的类导入容器，自动配置就会生效，帮我进行自动配置!

3.以前我们需要自动配置的东西，现在`springboot`帮我们做了!

4.整合`javaEE`，解决方案和自动配置的东西都在`spring-boot-autoconfigure-2.2.0.RELEASE.jar`这个包下

5.它会把所有需要导入的组件，以类名的方式返回，这些组件就会被添加到容器;

6.容器中也会存在非常多的`xxxAutoConfiguration`的文件(@Bean)，就是这些类给容器中导入了这个场景需要的所有组件;并自动配置，

@Configuration , `JavaConfig`!

7.有了自动配置类，免去了我们手动编写配置文件的工作!

**启动器**

```xml
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
</dependency>
```

* 启动器:说白了就是Springboot的启动场景;

* 比如spring-boot-starter-web，他就会帮我们自动导入web环境所有的依赖!

* springboot会将所有的功能场景，都变成一个个的启动器

* 我们要使用什么功能，就只需要找到对应的启动器就可以了`starter`





## 添加静态资源

```java
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
    } else {
        this.addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
        this.addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
            registration.addResourceLocations(this.resourceProperties.getStaticLocations());
            if (this.servletContext != null) {
                ServletContextResource resource = new ServletContextResource(this.servletContext, "/");
                registration.addResourceLocations(new Resource[]{resource});
            }

        });
    }
}
```