# MAVEN

Apache 下的一个纯 Java 开发的开源项目。基于项目对象模型（缩写：POM）概念，Maven利用一个中央信息片断能管理一个项目的构建、报告和文档等步骤。

Maven 是一个项目管理工具，可以对 Java 项目进行构建、依赖管理。

Maven 也可被用于构建和管理各种项目，例如 C#，Ruby，Scala 和其他语言编写的项目。Maven 曾是 Jakarta 项目的子项目，现为由 Apache 软件基金会主持的独立 Apache 项目。

Maven 的主要功能

- 依赖管理系统
- 多模块构建
- 一致的项目结构
- 一致的构建模型和插件机制

## Maven 规约
Maven 要负责项目的自动化构建，以编译为例，Maven 要想自动进行编译，那么它必须知道 Java 的源文件保存在哪里，这样约定之后，不用我们手动指定位置，Maven 能知道位置，从而帮我们完成自动编译。

- /src/main/java/ ：Java 源码。
- /src/main/resource ：Java 配置文件，资源文件。
- /src/test/java/ ：Java 测试代码。
- /src/test/resource ：Java 测试配置文件，资源文件。
- /target ：文件编译过程中生成的 .class 文件、jar、war 等等。
- pom.xml ：配置文件

## Maven 环境配置
Maven 是一个基于 Java 的工具，所以要做的第一件事情就是安装 JDK。

- Maven 3.3 要求 JDK 1.7 或以上
- Maven 3.2 要求 JDK 1.6 或以上
- Maven 3.0/3.1 要求 JDK 1.5 或以上

[安装教程](https://www.runoob.com/maven/maven-setup.html)


## Maven 常用命令

- mvn archetype：create ：创建 Maven 项目。
- mvn compile ：编译源代码。
- mvn deploy ：发布项目。
- mvn test-compile ：编译测试源代码。
- mvn test ：运行应用程序中的单元测试。
- mvn site ：生成项目相关信息的网站。
- mvn clean ：清除项目目录中的生成结果。
- mvn package ：根据项目生成的 jar/war 等。
- mvn install ：在本地 Repository 中安装 jar 。
- mvn eclipse:eclipse ：生成 Eclipse 项目文件。
- mvn jetty:run 启动 Jetty 服务。
- mvn tomcat:run ：启动 Tomcat 服务。
- mvn clean package -Dmaven.test.skip=true ：清除以前的包后重新打包，跳过测试类。

## Maven 优点和缺点

- 1）优点
    - 简化了项目依赖管理。
    - 便于与持续集成工具(Jenkins)整合。
    - 便于项目升级，无论是项目本身升级还是项目使用的依赖升级。
    - 有助于多模块项目的开发，一个模块开发好后，发布到仓库，依赖该模块时可以直接从仓库更新，而不用自己去编译。
    - Maven 有很多插件，便于功能扩展，比如生产站点，自动发布版本等。

- 2）缺点
    - Maven 是一个庞大的构建系统，学习难度大。
    - Maven 采用约定优于配置的策略(convention over configuration)，虽然上手容易，但是一旦出了问题，难于调试。

    - 由于需要下载，导入等，导致加载缓慢，或者出现错误，和不稳定

## Maven 坐标的含义

Maven 给我们制定了一套规则 —— 使用坐标进行唯一标识。Maven 的坐标元素包括 groupId、artifactId、version、packaging、classfier 。

- `groupId` ：定义当前 Maven 项目隶属的实际项目。首先，Maven 项目和实际项目不一定是一对一的关系。比如 Spring FrameWork 这一实际项目，其对应的 Maven 项目会有很多，如 spring-core、spring-context 等。这是由于 Maven 中模块的概念，因此，一个实际项目往往会被划分成很多模块。其次，groupId 不应该对应项目隶属的组织或公司。原因很简单，一个组织下会有很多实际项目，如果 groupId 只定义到组织级别，而后面我们会看到，artifactId 只能定义 Maven 项目(模块)，那么实际项目这个层次将难以定义。最后，groupId 的表示方式与 Java 包名的表达方式类似，通常与域名反向一一对应。上例中，groupId 为 junit ，是不是感觉很特殊，这样也是可以的，因为全世界就这么个 junit ，它也没有很多分支。
- `artifactId `：该元素定义当前实际项目中的一个 Maven 项目(模块)。推荐的做法是使用实际项目名称作为 artifactId 的前缀。比如上例中的 junit ，junit 就是实际的项目名称，方便而且直观。在默认情况下，Maven 生成的构件，会以 artifactId 作为文件头。例如 junit-3.8.1.jar ，使用实际项目名称作为前缀，就能方便的从本地仓库找到某个项目的构件。
- `version` ：该元素定义了使用构件的版本。如上例中 junit 的版本是 4.13-BETA ，你也可以改为 4.1.2 表示使用 4.1.2 版本的 junit 。
- `packaging` ：定义 Maven 项目打包的方式，使用构件的什么包。打包方式通常与所生成构件的文件扩展名对应。如上例中没有 packaging ，则默认为 jar 包，最终的文件名为junit-4.13-BETA.jar 。当然，也可以打包成 war 等。
- `classifier` ：该元素用来帮助定义构建输出的一些附件。附属构件与主构件对应。如上例中的主构件为 junit-4.13-BETA.jar ，该项目可能还会通过一些插件生成如 junit-4.13-BETA-javadoc.jar、junit-4.13-BETA-sources.jar，这样附属构件也就拥有了自己唯一的坐标。


- 上述 5 个元素中：

    - groupId、artifactId、version 是必须定义的。
    - packaging 是可选的(默认为 jar )。
    - 而 classfier 是不能直接定义的，需要结合插件使用。


## Maven 依赖的解析机制

- 解析发布(RELEASE)版本：如果本地有，直接使用本地的，没有就向远程仓库请求。
- 解析快照(SNAPSHOT)版本：合并本地和远程仓库的元数据文件 groupId/artifactId/version/maven-metadata.xml ，这个文件存的版本都是带时间戳的，将最新的一个改名为不带时间戳的格式供本次编译使用。
- 解析版本为 LATEST 过于复杂，且解析的结果不稳定，不推荐在项目中使用，感兴趣的同学自己去研究，简而言之就是合并groupId/artifactId/maven-metadata.xml 找到对应的最新版本和包含快照的最新版本。

>LASTEST、RELEASE、SNAPSHOT 的区别

- LASTEST ：是指某个特定构件最新的发布版或者快照版(SNAPSHOT)，最近被部署到某个特定仓库的构件。

- RELEASE ：是指仓库中最后的一个非快照版本。

- SNAPSHOT ：泛指。如果不 SNAPSHOT ，如果名字不变，本地有了不会从远程拉。如果每次更新都改名字，其他用的人也都改名字，太蛋疼了。

>Maven 的 Snapshot 版本与 Release 版本

- 1、Snapshot 版本代表不稳定、尚处于开发中的版本。

- 2、Release 版本则代表稳定的版本。

- 3、什么情况下该用 SNAPSHOT?

协同开发时，如果 A 依赖构件 B，由于 B 会更新，B 应该使用 SNAPSHOT 来标识自己。这种做法的必要性可以反证如下：

- a. 如果 B 不用 SNAPSHOT，而是每次更新后都使用一个稳定的版本，那版本号就会升得太快，每天一升甚至每个小时一升，这就是对版本号的滥用。
- b.如果 B 不用 SNAPSHOT, 但一直使用一个单一的 Release 版本号，那当 B 更新后，A 可能并不会接受到更新。因为 A 所使用的 repository 一般不会频繁更新 release 版本的缓存（即本地 repository)，所以B以不换版本号的方式更新后，A在拿B时发现本地已有这个版本，就不会去远程Repository下载最新的 B

>不用 Release 版本，在所有地方都用 SNAPSHOT 版本行不行？     

不行。正式环境中不得使用 snapshot 版本的库。 比如说，今天你依赖某个 snapshot 版本的第三方库成功构建了自己的应用，明天再构建时可能就会失败，因为今晚第三方可能已经更新了它的 snapshot 库。你再次构建时，Maven 会去远程 repository 下载 snapshot 的最新版本，你构建时用的库就是新的 jar 文件了，这时正确性就很难保证了。

## Maven常见的依赖范围

- compile:编译依赖，默认的依赖方式，在编译（编译项目和编译测试用例），运行测试用例，运行（项目实际运行）三个阶段都有效，典型地有spring-core等jar。
- test:测试依赖，只在编译测试用例和运行测试用例有效，典型地有JUnit。
- provided:对于编译和测试有效，不会打包进发布包中，典型的例子为servlet-api,一般的web工程运行时都使用容器的servlet-api。
- runtime:只在运行测试用例和实际运行时有效，典型地是jdbc驱动jar包。
- system: 不从maven仓库获取该jar,而是通过systemPath指定该jar的路径。
- import: 用于一个dependencyManagement对另一个dependencyManagement的继承。


>多模块项目管理项目依赖的版本

通过在父模块中声明dependencyManagement和pluginManagement， 然后让子模块通过`<parent>`元素指定父模块，这样子模块在定义依赖是就可以只定义groupId和artifactId，自动使用父模块的version,这样统一整个项目的依赖的版本。

## Maven 生命周期

Maven有三套相互独立的生命周期，分别是 Clean、Default 和 Site。每个生命周期包含一些阶段，阶段是有顺序的，后面的阶段依赖于前面的阶段。

1. Clean 生命周期：清理项目，包含三个 phase ：
- pre-clean：执行清理前需要完成的工作。
- clean：清理上一次构建生成的文件。
- post-clean：执行清理后需要完成的工作
2. Default 生命周期：构建项目，重要的 phase 如下：
- validate：验证工程是否正确，所有需要的资源是否可用。
- compile：编译项目的源代码。
- test：使用合适的单元测试框架来测试已编译的源代码。这些测试不需要已打包和布署。
- package：把已编译的代码打包成可发布的格式，比如 jar、war 等。
- integration-test：如有需要，将包处理和发布到一个能够进行集成测试的环境。
- verify：运行所有检查，验证包是否有效且达到质量标准。
- install：把包安装到maven本地仓库，可以被其他工程作为依赖来使用。
- deploy：在集成或者发布环境下执行，将最终版本的包拷贝到远程的repository，使得其他的开发者或者工程可以共享。
3. Site 生命周期：建立和发布项目站点，phase 如下：
- pre-site：生成项目站点之前需要完成的工作
- site：生成项目站点文档
- post-site：生成项目站点之后需要完成的工作
- site-deploy：将项目站点发布到服务器

各个生命周期相互独立，一个生命周期的阶段前后依赖。

`mvn clean` ：调用 Clean 生命周期的 clean 阶段，实际执行 pre-clean 和 clean 阶段

`mvn test` ：调用 Default 生命周期的 test 阶段，实际执行 test 以及之前所有阶段

`mvn clean install` ：调用 Clean 生命周期的 clean 阶段和 Default 生命周期 的 install 阶段，实际执行 pre-clean 和 clean ，install 以及之前所有阶段。

## Maven 仓库

Maven 的仓库只有两大类：

1. 本地仓库。
2. 远程仓库。在远程仓库中又分成了 3 种：
- 中央仓库。
- 私服。
- 其它公共库。

Maven 会先搜索本地仓库（repository），发现本地没有然后从远程仓库（中央仓库）获取。
- 但中央仓库只有一个，最好从其镜象处下载。国内可以用阿里云下的服务器。【其它公共库】
- 也有通过 Nexus 搭建的私服进行获取的。【私服】

> Maven私服的仓库类型

- （宿主仓库）hosted repository
- （代理仓库）proxy repository
- （仓库组）group repository