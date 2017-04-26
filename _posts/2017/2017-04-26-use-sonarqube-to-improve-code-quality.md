---
title: 使用 SonarQube 持续提升代码质量
date: '2017-04-26 23:45'
categories: devops
tags: SonarQube Sonar 代码质量 Jacoco 代码覆盖率
---

软件开发中, 除了编写业务代码外, 我们往往还需要写单元测试,集成测试,压力测试. 这些测试的目的是测量或者保证软件代码在`运行时的质量`, 这些测试往往是站在黑盒的角度来度量软件质量. 但是`运行时的质量`只是软件质量的一个方面, 我们还需要度量构成软件的代码的质量. 代码质量可以通过以下几点来度量: 可读性, 扩展性, 可维护性, 复杂度等等. `code review`是检查代码质量的一个手段, 但是`code review`是人工的, 效率较低, 不能看的很细, 也就不能发现很多潜在问题. 而且`code review`的效果严重依赖进行 review 的人的水平. 有没有软件自动检查代码质量, 可以代替人工`code review`, 或者至少部分代替? 静态代码检测工具就是用来干这个的. 目前有很多开源的静态代码监测工具, 最著名的要数 `SonarQube`. SonarQube有社区版和商业版. 社区版支持 JAVA, js, sql, xml 等语言, 集成了 PMD, FindBugs, Checkstyle, 可以说是分析 JAVA 源码的最佳工具.他可以找出 JAVA 代码中的格式问题, 性能问题, OWASP TOP 10等安全漏洞.

## 安装SonarQube
### quick start
使用 docker 启动:
```shell
$ docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube
```

启动后, 可以在docker 宿主机上的9000端口访问 SonarQube服务: http://${host}:9000

### 生产环境
上述配置只适合测试用, 一旦 docker 容器销毁后, 数据会丢失. 在生产环境中, 需要一个数据库, 并挂载相应的 docker 数据卷. 下面是一个生产环境的 docker-compose.yml 文件配置.

```yml
sonarqube:
  image: sonarqube
  environment:
    - reschedule:on-node-failure
    - SONARQUBE_JDBC_URL=jdbc:mysql://mysql-host:3306/sonar?useUnicode=true&characterEncoding=utf8
    - SONARQUBE_JDBC_USERNAME=sonar
    - SONARQUBE_JDBC_PASSWORD=sonar
  volumes:
    - /data/sonarqube/conf:/opt/sonarqube/conf
    - /data/sonarqube/data:/opt/sonarqube/data
    - /data/sonarqube/extensions:/opt/sonarqube/extensions
    - /data/sonarqube/bundled-plugins:/opt/sonarqube/lib/bundled-plugins
  restart: always
```

## 配置SonarQube服务器
### plugins
SonarQube通过 plugin来扩展其功能, 通过安装plugin, 可以支持分析 java,js,c#,objective c 等等各种语言源代码. 有些插件是收费的, JAVA,js,xml 等是默认免费支持的. 除了语言 plugin 外, 还有分析器 plugin. 比如 JAVA 界非常流行的Checkstyle, FindBugs, PMD 都默认集成了.

### 规则库
Checkstyle, FindBugs等工具是基于规则分析代码存在的问题和漏洞, SonarQube也一样. SonarQube默认集成了大量的规则, 仅 JAVA 就有近两千条规则.
这些 JAVA 规则以六个规则库的形式组织起来, 我们也可以创建自定义的规则库, 也可以修改现有的规则库. 如果一个语言对应有多个规则库, 那么需要指定一个默认规则库, SonarQube会默认使用默认规则库分析相应语言的源码.
![JAVA规则库](../img/sonar-java-rules.png)

规则分为三类: 坏味道, 漏洞, bug. 漏洞为安全漏洞, 包括了OWASP TOP 10漏洞模式. 漏洞和bug 是比较严重的问题, 需要修正. 坏味道往往是一些代码格式或者不鼓励的用法, 严重性相对较低, 可以根据自己公司的状况决定是否修改.

## maven配置
配置好 SonarQube服务器后, 如何如何让他来分析你的JAVA 代码呢?
首先, 需要配置 maven, 把以下代码加入~/.m2/settings.xml 中
```xml
<settings>
  <pluginGroups>
    <pluginGroup>org.sonarsource.scanner.maven</pluginGroup>
  </pluginGroups>
  <profiles>
    <profile>
        <id>sonar</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <sonar.host.url>
              http://your-sonar-server
            </sonar.host.url>
        </properties>
    </profile>
  </profiles>
</settings>
```

然后运行
```shell
mvn package sonar:sonar
```

运行完毕后, 访问 SonarQube服务器, 将会看到sonar 生成的详细代码分析报告. 下图为一个例子
![sonar project summary](../img/sonar-project-summary.png)

## 单元测试覆盖率 Jacoco
上图中, 有一项为覆盖率, 这是 Jacoco 生成的单元测试覆盖率. 单元测试覆盖率是评价单元测试是否足够的一个很重要的指标. 下面介绍如何让 SonarQube集成 Jacoco. 把下面代码加入 ~/.m2/settings.xml 中.

```xml
<settings>
  <profiles>
    <profile>
      <id>coverage-per-test</id>
      <build>
        <plugins>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <!-- Minimal supported version is 2.4 -->
            <version>2.20</version>
            <configuration>
              <properties>
                <property>
                  <name>listener</name>
                  <value>org.sonar.java.jacoco.JUnitListener</value>
                </property>
              </properties>
            </configuration>
          </plugin>
        </plugins>
      </build>

      <dependencies>
        <dependency>
          <groupId>org.sonarsource.java</groupId>
          <artifactId>sonar-jacoco-listeners</artifactId>
          <version>4.8.0.9441</version>
          <scope>test</scope>
        </dependency>
      </dependencies>
    </profile>
  </profiles>
</settings>
```

然后运行:
```shell
mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent package -Pcoverage-per-test sonar:sonar
```

运行完毕后, 可以在 SonarQube服务器上看到详细的 Jacoco 单元测试覆盖率报告.

## 跟Gitlab CI 持续集成
把下面代码加入 gitlab runner 机器的~/.m2/settings.xml 中.

```xml
<settings>
  <pluginGroups>
    <pluginGroup>org.sonarsource.scanner.maven</pluginGroup>
  </pluginGroups>
  <profiles>
    <profile>
        <id>sonar</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <sonar.host.url>
              http://your-sonar-server
            </sonar.host.url>
        </properties>
    </profile>
    <profile>
      <id>coverage-per-test</id>
      <build>
        <plugins>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <!-- Minimal supported version is 2.4 -->
            <version>2.20</version>
            <configuration>
              <properties>
                <property>
                  <name>listener</name>
                  <value>org.sonar.java.jacoco.JUnitListener</value>
                </property>
              </properties>
            </configuration>
          </plugin>
        </plugins>
      </build>

      <dependencies>
        <dependency>
          <groupId>org.sonarsource.java</groupId>
          <artifactId>sonar-jacoco-listeners</artifactId>
          <version>4.8.0.9441</version>
          <scope>test</scope>
        </dependency>
      </dependencies>
    </profile>
  </profiles>
</settings>
```

在 gitlab 项目的根目录下增加.gitlab-ci.yml 文件(参考[使用gitlab和阿里云容器服务进行持续部署]({{ site.url }}/ci-with-gitlab-and-aliyun-docker)), 并添加如下job:
```yml
build:
  stage: build
  script:
    - mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent package -Pcoverage-per-test sonar:sonar
  only:
    - master
  tags:
    - shell
```
