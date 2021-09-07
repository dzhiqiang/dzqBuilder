# DzqBuilder
通过Spring Initializr生成基于Spring Boot项目

源码地址：[https://github.com/spring-io/initializr](https://github.com/spring-io/initializr)

## 如何使用Spring Initializr

### 1. 先介绍Spring Initializr子项目：

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
   // 基本信息更新策略
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

#### 4.1 配置基本设置

大多数选择功能都是通过一个简单的基于列表的结构配置的，其中每个属性都有一个id、一个name以及该属性是否为默认属性。如果没有提供name，则使用id。

在src/main/resources下增加配置文件`application.yml`，添加languages 和 JVM 生成配置，内容如下

```yaml
# 属性可以参考ProjectDescription
initializr:
  # java 版本
  javaVersions:
    - id: 11
      default: false
    - id: 1.8
      # 默认
      default: true
  # 支持的语言
  languages:
    - name: Java
      id: java
      # 默认
      default: true
    - name: Kotlin
      id: kotlin
      default: false
```

重启服务，输入http://localhost:8080/，在jvm和languages 则会有选项，还有对应的默认值

:warning:对于languages 配置项必须有对应的实现，目前默认支持java，kotlin，groovy

可以通过下面的方式配置packagings 

```yaml
initializr:
  packagings:
    - name: Jar
      id: jar
      default: true
    - name: War
      id: war
      default: false
```

:warning:打包方式jar，war已经实现，对于其他打包格式，您需要实现`packaging`抽象并提供与之相对应的`PackagingFactory`。  

#### 4.2 配置文本设置

纯文本配置包括`groupId`, `artifactId`, `name`, `description`, `version` and `packageName`。 如果没有配置，每个配置都有一个默认值。 默认值也可以设置，如下所示:

```yaml
initializr:
  group-id:
    # group显示的默认值
    value: com.dzq
  artifact-id:
    value: demo
```

#### 4.3 配置多个选项的平台版本

可以像配置jvm和languages 一样配置平台版本

```yaml
initializr:
  bootVersions:
    - id: 2.4.0-SNAPSHOT
      name: 2.4.0 (SNAPSHOT)
      default: false
    - id: 2.3.3.BUILD-SNAPSHOT
      name: 2.3.3 (SNAPSHOT)
      default: false
    - id: 2.3.2.RELEASE
      name: 2.3.2
      default: true
```

上面的配置提供了三个版本，默认使用2.3.2。 但是在实践中，您可能希望升级可用的平台版本，而不必每次都重新部署应用程序。 实现您自己的`InitializrMetadataUpdateStrategy`bean允许您在运行时更新基本数据。`SaganInitializrMetadataUpdateStrategy`是该策略的一个实现，它获取最新的Spring Boot版本并更新基本数据，以确保使用这些配置的实例总是获得最新可用版本。

:warning: 如果使用`SaganInitializrMetadataUpdateStrategy` 请使用configure caching 防止过多请求。

#### 4.4 配置可用的项目类型

可用的项目类型主要定义生成的项目及其构建系统的结构。 一旦选择了项目类型，就会调用相关的操作来生成项目。  

默认情况下，Spring Initializr公开以下资源(所有通过HTTP GET访问):  

* `/pom.xml`：生成 Maven `pom.xml`
* `/build.gradle` 生成 Gradle build
* `/starter.zip` 生成完整项目结构的zip压缩文件
* `/starter.tgz`生成完整项目结构的tgz压缩文件

构建系统必须定义一个“build”标签，提供要使用的`BuildSystem`的名称(例如: `maven`,`gradle`)。  

可以配置额外的标签来进一步配置属性。 除了必填的`build`标签外，`format`标签也可用来定义项目的格式(`project`表示构建完整的项目，`build`只是生成构建文件)。 默认情况下，HTML UI过滤所有可用的类型，只显示`format`值为`project`的type。  

配置maven和gradle项目

```yaml
initializr:
  types:
    - name: Maven Project
      id: maven-project
      description: Generate a Maven based project archive
      tags:
        build: maven
        format: project
      default: true
      action: /starter.zip
    - name: Gradle Project
      id: gradle-project
      description: Generate a Gradle based project archive
      tags:
        build: gradle
        format: project
      default: false
      action: /starter.zip
```

#### 4.5 Configuring dependencies 配置依赖

最基本的`dependency`由以下几个部分组成:

* 客户端根据`id` 引用依赖
* 依赖的完整坐标`groupId` 和`artifactId`
* 用于显示的`name`,在UI中用于搜索
* `description`可以提供此依赖的一些其他信息

Spring Initializr自动认为没有maven坐标的依赖会被定义成Spring Boot启动器。 在这种情况下，id用于推断artifactId。

```yaml
initializr:
  dependencies:
    - name: Web
      content:
        # 没有groupId 和artifactId 默认artifactId为spring-boot-starter-web,格式为spring-boot-starter-{id}
        - name: Web
          id: web
          description: Full-stack web development with Tomcat and Spring MVC
```

在上面的spring-boot-start -web示例中，我们假设依赖是由平台管理(bootVersion)的，因此不需要为它提供版本属性。 您肯定需要定义平台没有提供的额外依赖项，我们强烈建议您使用配置表(Bill Of Materials，或BOM)下面会讲。

如果不适用BOM，你可以直接制定版本

```yaml
initializr:
  dependencies:
    - name: Tech
      content:
        - name: Acme
          # 支持的依赖
          id: acme
          groupId: com.example.acme
          artifactId: acme
          version: 1.2.0.RELEASE
          description: A solid description for this dependency
```

如果您添加了这个配置并搜索“acme”(或“solid”)，您将发现这个额外的配置; 用它生成一个maven项目应该将以下内容添加到pom中:  

```xml
<dependency>
    <groupId>com.example.acme</groupId>
    <artifactId>acme</artifactId>
    <version>1.2.0.RELEASE</version>
</dependency>
```

其他配置选项

##### 5.5.1 兼容范围

引入的依赖不同版本支持不同的Spring Boot版本，为了保证选择的Spring Boot版本对应的依赖的版本都是可用的，所以增加了依赖对Spring Boot版本的限制，通过`compatibilityRange`属性规定依赖使用的有效范围。规定的版本并不是代表依赖的版本，而是代表不同的Spring Boot版本，选择不同的Spring Boot版本用于过滤或者修改依赖。

一个版本由四部分组成:主要修订、次要修订、补丁修订和可选限定符。 Spring Initializr支持两种版本格式:  

`V1`是原始格式，其中限定符与版本用点分隔（2.3.4.RELEASE）。 它主要针对定义比较规范的快照(BUILD-SNAPSHOT)和(RELEASE)。  

`V2`是符合SemVer（2.3.4）的改进格式，因此使用破折号（-）分隔限定符。GAs没有限定符（没懂）。

限定符的排序如下：由小到大

* `M`代表里程碑(例如2.0.0.M1是即将到来的2.0.0发行版的第一个里程碑):可以被视为“beta”发行版。
* `RC`候选版本(例如2.0.0-rc2是即将发布的2.0.0版本的第二个候选版本)
* `BUILD-SNAPSHOT` 开发构建的（`2.1.0.BUILD-SNAPSHOT`表示即将发布的2.1.0版本的最新可用开发版本)。对于`V2`格式，它简单地称为`SNAPSHOT`，即`2.1.0-SNAPSHOT`。
* `RELEASE` 可用版本 (例如 `2.0.0.RELEASE` 就是 2.0.0版本)

:warning: 快照在该方案中有点特殊，因为它们总是表示发布的“最新状态”。`M1`代表一个给定的主要、次要和补丁版本的最老的版本，因此当在这一行中提到“第一个”版本时，它可以安全地使用。

一个版本范围有最低和最大边界。使用`[`或者`]`表示包含，否则可是使用 `(` 或者`)`。比如`[1.1.6.RELEASE,1.3.0.M1)`表示所有高于等于`1.1.6.RELEASE`版本，低于`1.3.0.M1`版本（当然也不包括高于等于`1.3.0.M1`的版本）

版本范围也可以制定一个版本值，比如1.2.0.RELEASE，表示这个版本或者更高版本。

如果你想制定最高版本，你可以使用`x`而不是一个硬编码的版本。比如`1.4.x.BUILD-SNAPSHOT`表示1.4.x 最新的快照版本。如果您想从最低1.1.0和发布到1.3的最新稳定版本限制一个依赖，你可以使用`[1.1.0.RELEASE,1.3.x.RELEASE]`。

:warning: 记住在YAML配置文件中引用版本范围的值(使用双引号"")。

##### 5.5.2 仓库

如果依赖在Maven Central （或者在后端配置的默认的仓库）没有可用的，你也可以添加一个仓库的引用。仓库的申明在顶层（env标签下），并通过配置中的关键字给出一个id：

```yaml
initializr:
  env:
    repositories:
      my-api-repo-1:
        name: repo1
        url: https://example.com/repo1
```

一旦定义，就可以在依赖中引用存储库

```yaml
initializr:
  dependencies:
    - name: Other
      content:
        - name: Foo
          groupId: org.acme
          artifactId: foo
          version: 1.3.5
          repository: my-api-repo-1
```

通常最好为每个依赖都有一个BOM，然后将仓库配置附加到BOM。

#### 4.6 配置列表

配置列表（BOM）是一种特殊的`pom.xml`,部署到Maven仓库。用来控制管理相关依赖的集合。在Spring Boot生态系统中，我们通常在BOM的id上使用`dependencies`。在其他项目中，我们看到`-bom`,建议将所有的依赖配置在BOM中，因为它提供了更高级的特性。同样重要的是，在一个项目中使用的两个bom不包含相同依赖项的冲突版本，因此最佳实践是在添加新bom之前查看Spring Initializr中的现有bom，并确保您没有添加冲突。

在Spring Initializr BOM定义在env级别，通过key的配置设定id的值，比如

```yaml
initializr:
  env:
    boms:
      #id的值
      my-api-bom:
        groupId: org.acme
        artifactId: my-api-dependencies
        version: 1.0.0.RELEASE
        repositories: my-api-repo-1
```

如果BOM需要一个特殊的、非默认的存储库，那么可以在这里引用它，而不必为每个依赖再次显式列出仓库。依赖或依赖组可以通过引用id声明它需要使用一个或多个BOM:

```yaml
initializr:
  dependencies:
    - name: Other
      content:
        - name: My API
          id : my-api
          groupId: org.acme
          artifactId: my-api
          bom: my-api-bom
```

##### 4.6.1. 通过平台版本定位坐标

处理依赖的配置和BOM结合使用外，您还可以使用版本映射配置版本关系在更细粒度的级别上。依赖或者BOM有一个映射列表，每一个映射列表由一个版本范围和一组覆盖平台版本的一个或者多个依赖组成。你可以使用映射来切换一个依赖版本，或者BOM,或者改变实例依赖id。

例如：

```yaml
initializr:
  env:
    boms:
      cloud-bom:
        groupId: com.example.foo
        artifactId: acme-foo-dependencies
        mappings:
          - compatibilityRange: "[1.2.3.RELEASE,1.3.0.RELEASE)"
            groupId: com.example.bar
            artifactId: acme-foo-bom
            version: Arcturus.SR6
          - compatibilityRange: "[1.3.0.RELEASE,1.4.0.RELEASE)"
            version: Botein.SR7
          - compatibilityRange: "[1.4.0.RELEASE,1.5.x.RELEASE)"
            version: Castor.SR6
          - compatibilityRange: "[1.5.0.RELEASE,1.5.x.BUILD-SNAPSHOT)"
            version: Diadem.RC1
            repositories: spring-milestones
          - compatibilityRange: "1.5.x.BUILD-SNAPSHOT"
            version: Diadem.BUILD-SNAPSHOT
            repositories: spring-snapshots,spring-milestones
```

这里主要的用法是平台版本映射到Foo项目的默认或者支持的版本，也可以看到对于里程碑和快照BOM声明了额外的仓库，因为这些版本可能不在当前仓库中，最初BOM被标识为`com.example.bar:acme-foo-bom`然后被重命名为`com.example.foo:acme-foo-dependencies`

##### 4.6.2 Aliases 别名

比如有一个"web-services"的依赖，可以通过别名定义一个新的名称

```yaml
initializr:
  dependencies:
    - name: Other
      content:
        - name: Web Services
          id: web-services
          aliases:
            - ws
```

生成项目时依赖的请求可以是`dependencies=ws`或者`dependencies=web-services`

##### 4.6.3 Facets 切面

facet是依赖上的标签，用于驱动生成项目中的代码修改。比如，`initializr-generator-spring`

检查如果是`war`类型打包的话是否存在`web`相关的依赖，如果没有`web`相关的依赖则驱动增加id为web的依赖

facets的值是一个List列表。

##### 4.6.4 Links 链接

Links(链接)可用于提供描述性和超链接数据，以指导用户如何更多地了解依赖关系。 依赖有一个"links"属性，它是一个Link列表。 每个链接都有一个rel标签来标识它，一个href和一个可选的(但推荐的)描述。

以下是目前官方支持的rel值:  

* `guide`：链接指向一个描述如何使用相关依赖项的指南。 它可以是一个教程，一个how-to或者通常是spring.io/guides上的一个指南 。
* `reference`：该链接通常指向开发人员指南的某一节或任何说明如何使用依赖项的页面  

如果url的实际值可以根据环境变化，则可以对其进行模板化。 URL参数是用花括号指定的，比如example.com/doc/{bootVersion}/section定义了bootVersion参数。  

目前支持一下属性：

`bootVersion`: 当前平台的版本。

下面的例子是针对`acme`依赖：

```yaml
initializr:
  dependencies:
    - name: Tech
      content:
        - name: Acme
          id: acme
          groupId: com.example.acme
          artifactId: acme
          version: 1.2.0.RELEASE
          description: A solid description for this dependency
          links:
            - rel: guide
              href: https://com.example/guides/acme/
              description: Getting started with Acme
            - rel: reference
              href: https://docs.example.com/acme/html
```

##### 4.6.5 提升搜索结果

每个依赖项都可以有一个`weight`权重(一个数字>=0)和关键字(字符串列表)，用于在web UI的搜索功能中对它们进行优先排序。 如果你在UI的“依赖项”框中输入一个关键字，这些依赖项将在下面列出，如果它们有一个(未加权的依赖项排在最后)，按照权重递减的顺序。 

### 5. 使用WEB生成项目

为了发现有哪些选项可以用于生成示例，假如有一个实例已经运行，可以使用curl 工具访问它。

```shell
$ curl http://localhost:8080
```

如果更喜欢`HTTPie`,可以使用一下命令

```shell
$ http http://localhost:8080
```

也可以直接在浏览器输入http://localhost:8080

返回值结果分为三个部分：

第一：可用[项目列表](#配置可用的项目类型)

第二：可选参数列表

第三：定义了依赖列表。 每个依赖提供了一个标识符，如果您想要选择依赖项、描述和兼容性范围(如果有的话)，则必须使用这个标识符。

假设您想要基于2.3.5版本生成一个“my-project.zip”项目。 平台的发布，使用web和devtools依赖项(记住，这两个id显示在服务的功能中):  

```shell
$ curl -G http://localhost:8080/starter.zip -d dependencies=web,devtools \
           -d bootVersion=2.3.5.RELEASE -o my-project.zip
```

如果你提取my-project.zip，你会注意到一些与web UI的区别:  

* 项目将在当前目录中被提取(web UI会自动添加一个与项目名称相同的基础目录)  

* 项目的名称不是my-project (-o参数对项目的名称没有影响)  

使用http命令也可以生成完全相同的项目:  

```shell
$ http https://localhost:8080/starter.zip dependencies==web,devtools \
           bootVersion==2.3.5.RELEASE -d
```

### 6. “如果做” 指引

本节提供了在配置Spring Initializr时经常出现的一些常见的“我如何做……”类型问题的答案。  

#### 6.1 添加一个新依赖

要添加一个新的依赖项，首先确定要添加的依赖项的Maven坐标(`groupId:artifactId:version`)，然后检查它使用的平台版本。 如果有多个版本可以使用不同的平台版本，那也没关系。 

* 如果有一个已发布的BOM管理您的依赖项的版本，那么首先将其添加到env部分(参见配置材料列表)。
* 然后配置依赖项，如果可以，将其放入现有组中，否则创建一个新组。
* 如果有BOM，则省略版本。
* 如果您需要这个依赖的兼容性版本范围(或最小或最大)，请将其作为关联到平台版本。

#### 6.2 覆盖依赖的版本

有时，管理依赖版本的BOM会与最新版本发生冲突。 或者，这可能只适用于一系列Spring Boot版本。 或者可能根本就没有BOM，或者不值得为一个依赖项创建一个BOM。 在这些情况下，您可以在顶级或版本映射中手动指定依赖项的版本。 在顶层它看起来是这样的(只是一个依赖项中的版本属性):  

```yaml
initializr:
  dependencies:
    - name: Tech
      content:
        - name: Acme
          id: acme
          groupId: com.example.acme
          artifactId: acme
          version: 1.2.0.RELEASE
          description: A solid description for this dependency
```

#### 6.3 将平台的一个版本关联到依赖的一个版本

如果你的依赖需要一个特定的平台版本，不同的平台版本需要你的依赖的不同版本，有一些机制可以配置它。
最简单的方法是在依赖声明中放入一个compatibilityRange。 这是平台的一系列版本，而不是您的依赖项。 例如:  

```yaml
initializr:
  dependencies:
    - name: Stuff
      content:
        - name: Foo
          id: foo
          ...
          # foo适用平台1.2.0版本
          compatibilityRange: 1.2.0.M1
        - name: Bar
          id: bar
          ...
          # bar适用平台1.2.0版本
          compatibilityRange: "[1.5.0.RC1,2.0.0.M1)"
```

如果你的依赖不同的版本对应不同的平台版本，那么您需要`mappings`属性，mapping是`compatibilityRange`属性和其他属性的集合，会覆盖被定义的值，比如

```yaml
initializr:
  dependencies:
    - name: Stuff
      content:
        - name: Foo
          id: foo
          groupId: org.acme.foo
          artifactId: foo-spring-boot-starter
          compatibilityRange: 1.3.0.RELEASE
          bom: cloud-task-bom
          mappings:
            - compatibilityRange: "[1.3.0.RELEASE,1.3.x.RELEASE]"
              artifactId: foo-starter
            - compatibilityRange: "1.4.0.RELEASE"
```

在这个例子中，当平台版本是1.3.0则`foo`的artifact是`foo-spring-boot-starter`,当1.3.x版本被选择，`foo`的artifact必须选择`foo-starter`

mapping也可以被应用在BOM声明:

```yaml
initializr:
  env:
    boms:
      my-api-bom:
        groupId: org.acme
        artifactId: my-api-bom
        additionalBoms: ['my-api-dependencies-bom']
        mappings:
          - compatibilityRange: "[1.0.0.RELEASE,1.1.6.RELEASE)"
            version: 1.0.0.RELEASE
            repositories: my-api-repo-1
          - compatibilityRange: "1.2.1.RELEASE"
            version: 2.0.0.RELEASE
            repositories: my-api-repo-2
```

在这个例子中，平台版本`1.1.6`选择依赖版本`1.0.0`，设置一个不同的仓库，平台版本`1.2.1`选择依赖版本`2.0.0`，在设置另外一个仓库。

#### 6.4 配置一个仓库

如果默认的仓库(通常是Maven Central)不包含所需要的依赖，那么依赖或BOM可能需要使用特定的仓库。 通常，声明它的最佳位置是在BOM配置中，但是如果没有BOM，那么您可以将它放在依赖配置中。 您还可以使用平台版本映射来覆盖依赖或BOM的默认仓库。这个上面的例子很多了！！

#### 6.5 配置一个自定义的父POM

如果是Maven项目，你可以配置一个自定义的POM

```yaml
initializr:
  env:
    maven:
      parent:
        groupId: com.example
        artifactId: my-parent
        version: 1.0.0
        includeSpringBootBom : true
```

`includeSpringBootBom` 默认值`false` .当是  `true`时,  `spring-boot-dependencies` 将与项目使用的Spring Boot版本一起添加到 `dependencyManagement` 。

#### 6.6 确保规律的依赖具有基本的启动器

如果一个依赖项不是独立的(特别是如果它不依赖于一个现有的Spring Boot启动器)，你可以将它标记为“non starter”:  

```yaml
initializr:
  dependencies:
    - name: Stuff
      content:
        - name: Lib
          id: lib
          groupId: com.acme
          artifactId: lib
          starter: false
```

当生成的项目只有这个标志的依赖时，也会添加基本的Spring Boot启动器。 

#### 6.7 设置组共享公共属性

依赖组是用户界面实现的一个提示，用于在用户选择依赖项时将事物分组在一起。 这也是在依赖之间共享设置的一种方便的方法，因为每个依赖项都继承所有的设置。 组中最常见的设置是groupId, compatibilityRange和bom:  

```yaml
initializr:
  dependencies:
    - name: Stuff
      bom: stuff-bom
      compatibilityRange: "[1.3.0.RELEASE,2.0.0.M1)"
      content:
...
```

默认情况下，这些依赖仅对1.3.0版本的平台版本可用到2.0.0.M1(不包括)，并使用`stuff-bom`BOM.

#### 6.8 配置平台版本格式

Spring Initializr支持两种格式:`V1`是2.1之前由基本数据定义的原始格式。 `V2`是在基本数据2.2中随`V1`一起提供的SemVer格式。 为了提供向后兼容的内容，应该配置每种格式的版本范围，以便能够匹配。 

```yaml
initializr:
  env:
    platform:
      compatibility-range: "2.0.0.RELEASE"
      v1-format-compatibility-range: "[2.0.0.RELEASE,2.4.0-M1)"
      v2-format-compatibility-range: "2.4.0-M1"
```

### 7. 更高级的配置

#### 7.1 缓存配置

如果您使用了服务，您会注意到日志中有许多包含从`spring.io/project_metadata/spring-boot`获取Spring Boot 基本信息的日志。 为了避免过于频繁地检查最新的Spring Boot版本，您应该在服务上启用缓存。 如果您愿意使用JCache (JSR-107)实现，Spring Initializr有一些自动配置来应用适当的缓存。  

### // TODO 文档未翻译完



## 源码解读

Spring Boot 自动配置

### initializr-web

#### InitializrAutoConfiguration

```properties
# 文件地址META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
io.spring.initializr.web.autoconfigure.InitializrAutoConfiguration
```

```java
// application.yml 文件initializr是root标签
@ConfigurationProperties(prefix = "initializr")
public class InitializrProperties extends InitializrConfiguration
    
@JsonIgnore // 序列化忽略此属性
/**
  * Available Spring Boot versions.
  * 可用的Spring Boot版本
*/
@JsonIgnore
private final List<DefaultMetadataElement> bootVersions = new ArrayList<>();
```

```java
// 讲解InitializrAutoConfiguration 提供的bean
@Configuration
@EnableConfigurationProperties(InitializrProperties.class) // 上面已经讲了
public class InitializrAutoConfiguration {
   // 用于生成临时文件，等待压缩
   // ProjectDirectoryFactory 参数ProjectDescription
   @Bean
   @ConditionalOnMissingBean // 同类型只能有一个
   public ProjectDirectoryFactory projectDirectoryFactory() {
      return (description) -> Files.createTempDirectory("project-");
   }
   // 生成输出字符流的工厂
   @Bean
   @ConditionalOnMissingBean
   public IndentingWriterFactory indentingWriterFactory() {
      return IndentingWriterFactory.create(new SimpleIndentStrategy("\t"));
   }

   @Bean
   @ConditionalOnMissingBean(TemplateRenderer.class)
   // 读取配置文件，进行模板转换 mustache 模板
   // 模板用法 https://www.cnblogs.com/df-fzh/p/5979093.html
   public MustacheTemplateRenderer templateRenderer(Environment environment,
         // 防止不存在CacheManager，没有不会报错，注入CacheManager
         // 可选配置
         ObjectProvider<CacheManager> cacheManager) {
      return new MustacheTemplateRenderer("classpath:/templates",
            determineCache(environment, cacheManager.getIfAvailable()));
   }
   
   private Cache determineCache(Environment environment, CacheManager cacheManager) {
      if (cacheManager != null) {
         // Binder更容易的绑定对象，更容易转换成对象
         // https://blog.csdn.net/u011357213/article/details/109360301
         Binder binder = Binder.get(environment);
         boolean cache = binder.bind("spring.mustache.cache", Boolean.class).orElse(true);
         if (cache) {
            return cacheManager.getCache("initializr.templates");
         }
      }
      return new NoOpCache("templates");
   }
   // 基础信息提供bean
   @Bean
   @ConditionalOnMissingBean(InitializrMetadataProvider.class)
   public InitializrMetadataProvider initializrMetadataProvider(InitializrProperties properties,
         // 可选配置
         // 可以选择自动从spring服务获取最新的springboot版本
         ObjectProvider<InitializrMetadataUpdateStrategy> initializrMetadataUpdateStrategy) {
      // 从properties转到InitializrMetadataProvider中
      InitializrMetadata metadata = InitializrMetadataBuilder.fromInitializrProperties(properties).build();
      return new DefaultInitializrMetadataProvider(metadata,
            initializrMetadataUpdateStrategy.getIfAvailable(() -> (current) -> current));
   }
   // 根据InitializrMetadata基本信息得到DependencyMetadata依赖的基本信息
   @Bean
   @ConditionalOnMissingBean
   public DependencyMetadataProvider dependencyMetadataProvider() {
      return new DefaultDependencyMetadataProvider();
   }

   /**
    * Initializr web configuration.
    * 初始化web配置
    */
   @Configuration
   @ConditionalOnWebApplication
   static class InitializrWebConfiguration {
	  // 配置了ContentType类型
      @Bean
      InitializrWebConfig initializrWebConfig() {
         return new InitializrWebConfig();
      }
	  // 项目生成
      @Bean
      @ConditionalOnMissingBean
      ProjectGenerationController<ProjectRequest> projectGenerationController(
            InitializrMetadataProvider metadataProvider, // 上文的基本信息提供者
            ObjectProvider<ProjectRequestPlatformVersionTransformer> platformVersionTransformer, // 可选的
            ApplicationContext applicationContext) {// bean上下文
         // 项目生成调用者
         ProjectGenerationInvoker<ProjectRequest> projectGenerationInvoker = new ProjectGenerationInvoker<>(
               // bean上下文和DefaultProjectRequestToDescriptionConverter:把请求转换为ProjectDescription
               // DefaultProjectRequestToDescriptionConverter 会进行转换，转换过程包含校验参数赋值，比较简单不在细讲
               applicationContext, new DefaultProjectRequestToDescriptionConverter(platformVersionTransformer
                     .getIfAvailable(DefaultProjectRequestPlatformVersionTransformer::new)));
         return new DefaultProjectGenerationController(metadataProvider, projectGenerationInvoker);
      }
	  // 后面的Controller暂时不涉及，先以ProjectGenerationController举例
      @Bean
      @ConditionalOnMissingBean
      ProjectMetadataController projectMetadataController(InitializrMetadataProvider metadataProvider,
            DependencyMetadataProvider dependencyMetadataProvider) {
         return new ProjectMetadataController(metadataProvider, dependencyMetadataProvider);
      }

      @Bean
      @ConditionalOnMissingBean
      CommandLineMetadataController commandLineMetadataController(InitializrMetadataProvider metadataProvider,
            TemplateRenderer templateRenderer) {
         return new CommandLineMetadataController(metadataProvider, templateRenderer);
      }

      @Bean
      @ConditionalOnMissingBean
      SpringCliDistributionController cliDistributionController(InitializrMetadataProvider metadataProvider) {
         return new SpringCliDistributionController(metadataProvider);
      }

      @Bean
      InitializrModule InitializrJacksonModule() {
         return new InitializrModule();
      }

   }

   /**
    * Initializr cache configuration.
    */
   @Configuration
   @ConditionalOnClass(javax.cache.CacheManager.class)
   static class InitializrCacheConfiguration {

      @Bean
      JCacheManagerCustomizer initializrCacheManagerCustomizer() {
         return new InitializrJCacheManagerCustomizer();
      }

   }

   @Order(0)
   private static class InitializrJCacheManagerCustomizer implements JCacheManagerCustomizer {

      @Override
      public void customize(javax.cache.CacheManager cacheManager) {
         createMissingCache(cacheManager, "initializr.metadata",
               () -> config().setExpiryPolicyFactory(CreatedExpiryPolicy.factoryOf(Duration.TEN_MINUTES)));
         createMissingCache(cacheManager, "initializr.dependency-metadata", this::config);
         createMissingCache(cacheManager, "initializr.project-resources", this::config);
         createMissingCache(cacheManager, "initializr.templates", this::config);
      }

      private void createMissingCache(javax.cache.CacheManager cacheManager, String cacheName,
            Supplier<MutableConfiguration<Object, Object>> config) {
         boolean cacheExist = StreamSupport.stream(cacheManager.getCacheNames().spliterator(), true)
               .anyMatch((name) -> name.equals(cacheName));
         if (!cacheExist) {
            cacheManager.createCache(cacheName, config.get());
         }
      }

      private MutableConfiguration<Object, Object> config() {
         return new MutableConfiguration<>().setStoreByValue(false).setManagementEnabled(true)
               .setStatisticsEnabled(true);
      }

   }

}
```

####  `/starter.zip`方法

```java
//ProjectGenerationController
@RequestMapping("/starter.zip")
public ResponseEntity<byte[]> springZip(R request) throws IOException {
   // 重点讲解文件生成过程
   // https://start.spring.io/#!type=maven-project&language=java&platformVersion=2.5.4&packaging=jar&jvmVersion=1.8&groupId=com.example&artifactId=demo&name=demo&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.demo&dependencies=lombok,web
   ProjectGenerationResult result = this.projectGenerationInvoker.invokeProjectStructureGeneration(request);
   // 生成zip文件
   Path archive = createArchive(result, "zip", ZipArchiveOutputStream::new, ZipArchiveEntry::new,
         ZipArchiveEntry::setUnixMode);
   return upload(archive, result.getRootDirectory(), generateFileName(request, "zip"), "application/zip");
}
```

生成文件

```java
public ProjectGenerationResult invokeProjectStructureGeneration(R request) {
   // 上文已经介绍
   InitializrMetadata metadata = this.parentApplicationContext.getBean(InitializrMetadataProvider.class).get();
   try {
      // 进行转换 
      ProjectDescription description = this.requestConverter.convert(request, metadata);
      // 项目生成器
      /**
      	1. 生成projectGenerationContext，相当于生成了本次请求的上下文，默认的无参构造器+context.setAllowBeanDefinitionOverriding(false);//不允许覆盖
      	2. 对projectGenerationContext进行处理
      	    // 把ApplicationContext父上下文，通用的bean
      		context.setParent(this.parentApplicationContext);
      		// 注册InitializrMetadata ,application.yml中属性
			context.registerBean(InitializrMetadata.class, () -> metadata);
			// 注册基本信息解析类，解析依赖，解析bom,解析仓库
			context.registerBean(BuildItemResolver.class, () -> new MetadataBuildItemResolver(metadata, context.getBean(ProjectDescription.class).getPlatformVersion()));
			// 继承：ProjectDescriptionCustomizer，主要是处理request中路径，包名不规范命名进行统一（转换）
			context.registerBean(MetadataProjectDescriptionCustomizer.class,
				() -> new MetadataProjectDescriptionCustomizer(metadata));
      */
      ProjectGenerator projectGenerator = new ProjectGenerator((
            // 处理projectGenerationContext过程
            projectGenerationContext) -> customizeProjectGenerationContext(projectGenerationContext, metadata));
      // 开始生成
      ProjectGenerationResult result = projectGenerator.generate(description,
            generateProject(description, request));
      addTempFile(result.getRootDirectory(), result.getRootDirectory());
      return result;
   }
   catch (ProjectGenerationException ex) {
      publishProjectFailedEvent(request, metadata, ex);
      throw ex;
   }
}
```

生成文件

```java
public <T> T generate(ProjectDescription description, ProjectAssetGenerator<T> projectAssetGenerator)
      throws ProjectGenerationException {
   // 上文已经介绍，默认的构造器
   try (ProjectGenerationContext context = this.contextFactory.get()) {
      // 把生成description的Supplier放入到上下文，并且注入ProjectDescriptionDiffFactory，并执行ProjectDescriptionCustomizer实现类-MetadataProjectDescriptionCustomizer，refresh时执行
      registerProjectDescription(context, description);
      // 所有META-INF/spring.factories文件中的ProjectGenerationConfiguration，进行注入BeanDefinition
      registerProjectContributors(context, description);
      // 处理context 上文中的2
      this.contextConsumer.accept(context);
      // 生成对应的bean ,执行了实现ProjectDescriptionCustomizer接口的customize方法
      context.refresh();
      try {
         // 生成文件
         return projectAssetGenerator.generate(context);
      }
      catch (IOException ex) {
         throw new ProjectGenerationException("Failed to generate project", ex);
      }
   }
}
```

生成文件

```java
// ProjectAssetGenerator 函数式接口
private ProjectAssetGenerator<ProjectGenerationResult> generateProject(ProjectDescription description, R request) {
   return (context) -> {
      // 生成项目目录 ，调用 DefaultProjectAssetGenerator的generate方法
      Path projectDir = getProjectAssetGenerator(description).generate(context);
      publishProjectGeneratedEvent(request, context);
      return new ProjectGenerationResult(context.getBean(ProjectDescription.class), projectDir);
   };
}
```

默认的生成方法

```java
@Override
public Path generate(ProjectGenerationContext context) throws IOException {
   // 得到请求参数
   ProjectDescription description = context.getBean(ProjectDescription.class);
   // 创建项目文件,空文件：C:\Users\duanzhiqiang1\AppData\Local\Temp\project-5900421886564624260
   Path projectRoot = resolveProjectDirectoryFactory(context).createProjectDirectory(description);
   // 创建项目目录，空文件：C:\Users\duanzhiqiang1\AppData\Local\Temp\project-5900421886564624260\demo
   Path projectDirectory = initializerProjectDirectory(projectRoot, description);
   // 排序得到所有的ProjectContributor：
   // 所有注册的ProjectContributor执行contribute，开始构建项目
   List<ProjectContributor> contributors = context.getBeanProvider(ProjectContributor.class).orderedStream()
         .collect(Collectors.toList());
   for (ProjectContributor contributor : contributors) {
      contributor.contribute(projectDirectory);
   }
   return projectRoot;
}
```

注入的例子：

```java
@ProjectGenerationConfiguration
public class ApplicationConfigurationProjectGenerationConfiguration {

   @Bean
   public ApplicationPropertiesContributor applicationPropertiesContributor() {
      return new ApplicationPropertiesContributor();
   }
   // web文件注入
   // 默认存在，如果依赖存在facet等于web则创建文件目录
   @Bean
   public WebFoldersContributor webFoldersContributor(Build build, InitializrMetadata metadata) {
      return new WebFoldersContributor(build, metadata);
   }

}
@Override
public void contribute(Path projectRoot) throws IOException {
    if (build){
        
    }
    
	if (this.buildMetadataResolver.hasFacet(this.build, "web")) {
		Files.createDirectories(projectRoot.resolve("src/main/resources/templates"));
		Files.createDirectories(projectRoot.resolve("src/main/resources/static"));
	}
}
```

其他Contributor

--MultipleResourcesProjectContributor：多文件复制，可以增加 可执行文件名称

MavenWrapperContributor继承MultipleResourcesProjectContributor:把目录classpath:maven/wrapper文件复制到项目目录

MavenBuildProjectContributor：创建pom文件，写入pom文件内容。

创建MavenBuild，并放入BuildCustomizer接口的bean,执行BuildCustomizer的customize，然后修改mavenbuild，也就是修改POM格式。

MainApplicationTypeCustomizer：生成main方法，主方法。

TestSourceCodeProjectContributor：测试文件生成。

继承：SingleResourceProjectContributor 单文件处理

ApplicationPropertiesContributor：属性文件复制

HelpDocumentProjectContributor：帮助文档

GitIgnoreContributor：.gitignore文档复制

多配置：











