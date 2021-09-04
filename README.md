# DzqBuilder
通过Spring Initializr生成基于Spring Boot项目

源码地址：[https://github.com/spring-io/initializr](https://github.com/spring-io/initializr)

## 如何使用Spring Initializr

#### 1. 先介绍Spring Initializr子项目：

`initializr-actuator`：可选模块，提供额外信息和统计功能。

`initializr-bom`：提供一个所需配置属性的列表更容易构建项目。

`initializr-docs`：生成文档。

`initializr-generator`：项目生成的核心类。

`initializr-generator-spring`：用于生成Spring Boot项目的可选模块，能够重用或替换你现在的系统架构。

`initializr-generator-test`：用于生成项目的测试模块。

`initializr-metadata`：用于生成项目的基础信息模块。

`initializr-service-sample`：自定义实例的例子，可以直接运行此模块，就能够生成项目。

`initializr-version-resolver`：可选模块，用户从pom文件中提取版本号。

`initializr-web`：用于第三方的web端，引用此项目可以直接http访问项目。

### 2. Initializr Generator

> `initializr-generator` 可以生成项目比较底层基础部分

#### 2.1 Project Generator

> `ProjectGenerator`类是生成项目的主要入口。`ProjectGenerator`接受参数`ProjectDescription`：生成项目的描述信息和参数 `ProjectAssetGenerator`:可以根据项目描述生成项目。

```java
// ProjectGenerator通过传入ProjectDescription和ProjectAssetGenerator生成项目
public <T> T generate(ProjectDescription description, ProjectAssetGenerator<T> projectAssetGenerator)
      throws ProjectGenerationException {
   // 获取到context
   try (ProjectGenerationContext context = this.contextFactory.get()) {
      // description放入到context中
      registerProjectDescription(context, description);
      registerProjectContributors(context, description);
      this.contextConsumer.accept(context);
      context.refresh();
      try {
         // 生成
         return projectAssetGenerator.generate(context);
      }
      catch (IOException ex) {
         throw new ProjectGenerationException("Failed to generate project", ex);
      }
   }
}
```

一个`ProjectDescription`项目描述都有那些属性呢？

* 坐标：`groupId`, `artifactId`, `name`, `description`
* `BuildSystem`构建系统,是Maven还是Gradle
* `Packaging`打包类型，是jar还是war
* The JVM `Language`: 是java还是kotlin
* 需要的依赖
* 项目的版本号，名称，包名

如上述代码，项目的生成是发生在`ProjectGenerationContext` 的。`ProjectGenerationContext`生成组件是通过注解`@ProjectGenerationConfiguration`定义的。这个配置如果注册在META-INF/spring.factories中，那么就会自动引入。比如

```properties
io.spring.initializr.generator.project.ProjectGenerationConfiguration=\
com.example.acme.build.BuildProjectGenerationConfiguration,\
com.example.acme.code.SourceCodeProjectGenerationConfiguration
```

注入到`ProjectGenerationContext`的组件使用是可以添加条件的。添加条件防止暴露bean，这些bean必须检查他们要做的事情。比如：

```java
@Bean
// 如果系统是gradle构建
@ConditionalOnBuildSystem(GradleBuildSystem.ID)
// 打包方式为war
@ConditionalOnPackaging(WarPackaging.ID)
public BuildCustomizer<GradleBuild> warPluginContributor() {
    // 返回BuildCustomizer：GradleBuild
    return (build) -> build.plugins().add("war");
}
```

BuildCustomizer<GradleBuild>组件只有当使用Gradle构建war时才会被注册。继承 `ProjectGenerationCondition` 可以创建自定义的条件。

这个条件只能在`ProjectGenerationConfiguration`下才能使用，因为需要 `ProjectDescription`

项目生成也可能依赖一些不是特定项目的组件（公共的一些组件），那么可以把这些公共的设计为特殊Context的父类，也能防止每次请求都要构建一次。比如

```java
// 参数是公共组件内容
public ProjectGenerator createProjectGenerator(ApplicationContext appContext) {
    // context是每次都重新构建的内容
    return new ProjectGenerator((context) -> {
        // 每次生成ProjectGenerator时把新的context的父类都设置成appContext
        context.setParent(appContext);
        context.registerBean(SampleContributor.class, SampleContributor::new);
    });
}
```

创建了一个新的 `ProjectGenerator`，能够使用项目中的bean,也可以使用通过在 `META-INF/spring.factories` 注册的所有项目贡献类（`ProjectContributor`）。也可以额外的注册 `ProjectContributor`

`ProjectContributor`是构建项目最高级别的接口。上面注册的`SampleContributor` 在跟目录生成 `hello.txt` 

```java
public class SampleContributor implements ProjectContributor {
    @Override
    public void contribute(Path projectRoot) throws IOException {
        // 生成hello.txt
        Path file = Files.createFile(projectRoot.resolve("hello.txt"));
        try (PrintWriter writer = new PrintWriter(Files.newBufferedWriter(file))) {
            // 写入内容Test
            writer.println("Test");
        }
    }
}
```

#### 2.2 Project Generation 生命周期

当类 `ProjectGenerator` 生成一个项目时，指定的`ProjectDescription`能够指定使用可用的 `ProjectDescriptionCustomizer`beans 还能够使用Spring的 `Ordered`进行排序。`ProjectDescriptionDiff` bean 能够知道原始的 `ProjectDescription`属性是否被修改。

一旦描述`ProjectDescription`根据`ProjectDescriptionCustomizer`制定过了，生成器就使用`ProjectAssetGenerator`生成项目，`initializr-generator`模块提供提供`DefaultProjectAssetGenerator`默认的实现生成一个目录结构，通过使用可用的`ProjectContributor`。

虽然默认的`ProjectAssetGenerator`使用系统文件和调用一组制定的组件，他也可以使用相同的`ProjectGenerator`实例和一个完全专注于其他内容的自定义实现。

#### 2.3 Project Abstractions 项目抽象

这个模块`ProjectGenerator`还包含了项目各个方面的抽象，以及一些方便的实现:

使用Maven和Gradle构造系统的抽象实现。

编程语言Java,Kotlin ,Groovy 的实现，`SourceCodeWriter` 是所有编程语言的接口。

一个打包接口针对于jar和war。

为它们添加新的实现需要创建BuildSystemFactory、LanguageFactory和PackagingFactory，并将它们注册到META-INF/spring中。分别是`io.spring.initializr.generator.buildsystem.BuildSystemFactory`，`io.spring.initializr.generator.language.LanguageFactory`和`io.spring.initializr.generator.packaging.PackagingFactory`。

JVM项目通常包含构建配置。`ProjectGenerator`模块提供了基于Maven和Gradle的 `build`模型，这个模型可以根据配置进行构建，这个工具包提供了`MavenBuildWriter`和`GradleBuildWriter`可以将Build 模型转换为构建文件。

### 3. Conventions for Spring Boot

> 这是一个可选组件，可以构建Spring Boot项目

#### 3.1 Requirements 需求

项目的开发者希望一下模块能够在`ProjectGenerationContext`bean下使用

* `InitializrMetadata`实例可以使用
* 一个可选的`MetadataBuildItemResolver`，它可以解析各种构建项(例如基于基本数据中的id的依赖和bom)

如果你正在使用父上下文，建议在父上下文配置它们，因为你不应该在每次生成新项目时都注册它们:

* `IndentingWriterFactory` 表示要使用缩进策略.
* `MustacheTemplateRenderer`使用`classpath:/templates`作为根位置。考虑用缓存策略注册这样的bean，以避免每次都解析模板。

#### 3.2 支持

支持以下方面：

* web:用于驱动包含一个id的 `web`的依赖项(如果没有该方面的依赖项存在，默认为spring-boot-start -web)
* jpa:用于标记该项目使用JPA。当与Kotlin结合使用时，可以确保配置适当的插件
* json:用来标记这个项目基于Jackson。当与Kotlin结合使用时，可以确保添加特定于Kotlin的jackson模块，以获得更好的互操作性。

### 4 创建自己的实例

创建一个新项目maven引入如下配置

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.4.RELEASE</version>
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.spring.initializr</groupId>
        <artifactId>initializr-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.spring.initializr</groupId>
        <artifactId>initializr-generator-spring</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.spring.initializr</groupId>
            <artifactId>initializr-bom</artifactId>
            <version>0.11.0-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

加入启动类

```java
@SpringBootApplication
@EnableCaching
@EnableAsync
public class ServiceApplication {
   public static void main(String[] args) {
      SpringApplication.run(ServiceApplication.class, args);
   }
   @Bean
   SaganInitializrMetadataUpdateStrategy saganInitializrMetadataUpdateStrategy(RestTemplateBuilder restTemplateBuilder,
         ObjectMapper objectMapper) {
      return new SaganInitializrMetadataUpdateStrategy(restTemplateBuilder.build(), objectMapper);
   }

}
```

输入http://localhost:8080/能够得到空的json。

> :warning:Spring Initializr项目是没有页面的。

可以通过配置`application.yml`文件配置更多的选项。



