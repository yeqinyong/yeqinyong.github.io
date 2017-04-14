---
title:  "使用gitlab和阿里云容器服务进行持续部署(二) -- spring应用多环境部署"
categories: devops
tags: gitlab CI CD 阿里云 容器服务 docker spring springboot
---
{% include toc title="目录" icon="file-text" %}

在[上篇文章]({{ site.url }}/ci-with-gitlab-and-aliyun-docker)中我们介绍了[使用gitlab和阿里云容器服务进行持续部署]({{ site.url }}/ci-with-gitlab-and-aliyun-docker). 在实际开发中, 我们往往有多种部署环境, 比如本地开发环境, 测试环境, 预发布环境, 生产环境. 本文结合技术讲讲如何使用gitlab和阿里云容器服务进行spring应用多环境持续部署.

## Spring Configuration
spring应用的配置可以写在不同格式的文件中, 比如 xml, json, properties, yml 等, 甚至可以写在[配置中心](http://cloud.spring.io/spring-cloud-static/Camden.SR5/#_spring_cloud_config), 获得动态改变配置的能力.

spring 配置还支持多环境, 我们可以很方便的为不同环境(开发,测试,生产等)准备不同的配置文件.

比如, 我们有三个spring 配置文件:
1. application.yml
```yaml
name: developer
```

2. application-test.yml
```yaml
name: tester
```

3. application-prod.yml
```yaml
name: operator
```

那么, 我们直接运行改程序, 看到的 name 为 `developer`. 如果把环境变量设为: `export SPRING_PROFILES_ACTIVE=test`, 再运行程序, 看到的 name 为`tester`. 如果把环境变量设为: `export SPRING_PROFILES_ACTIVE=prod`, 再运行程序, 看到的 name 为`operator`.

利用这一点, 我们可以只用 build 一次, 然后在不同的环境设置不同的`SPRING_PROFILES_ACTIVE`环境变量, 达到一次 build, 多次部署的目的.

## git 分支策略与工作流
为什么要讲 git分支策略呢? git 分支策略和团队开发协作与版本的发布关系很大, 直接影响工作流和不同版本在不同环境的部署.

目前比较常用的git 分支策略有:
- git flow
- github flow
- gitlab flow  

三者的具体介绍可以参考 http://www.15yan.com/topic/yi-dong-kai-fa-na-dian-shi/6yueHxcgD9Z/ . 我们这里简要介绍下.

#### git flow
git flow 是最复杂的, 他提倡一个master分支和一个develop分支，同时也包括了feature分支、release分支、hotfix分支。git flow 适合需要维护多个版本的软件开发. 比如传统软件, 类似 Windows, 需要同时维护多个Windows版本的开发与更新.

#### gihub flow
github flow 相对要简单很多, 它有一个master分支和若干 feature分支组成. 功能开发用 feature 分支, 完成后并入 master 分支, master 分支用于发布.

#### gitlab flow
gitlab flow 强调一个 master 分支, 用来存放最新代码, 若干个环境分支, 对应不同的部署环境. 比如你有四个部署环境: 本地开发环境, 测试环境, 预发布环境, 生产环境. 那么除了 master 分支, 只要额外两个环境分支: 预发布分支和生产分支. 为什么只需要额外两个环境分支? 本地开发环境和测试环境怎么办? 本地开发环境对应的为开发者本地 git repository 的 master 分支, 测试环境可以对应git 服务器上的 master 分支. 当然你也可以创建一个 test 分支来对应 test 环境.

个人认为gitlab flow 使用起来更为简单, 非常适合互联网服务.

#### 我们采用的分支策略和工作流
由于我们公司(亿保健康)是小产品团队加微服务架构, 而且我们只有三个部署环境: 本地开发环境, 测试环境(兼具测试和预发布功能), 和生产环境. 所以我们大多数的项目采用一种非常简单的, 只有两个分支的 gitlab flow: 一个 master 分支, 一个 release 分支. master分支作为持续集成分支, 每次代码提交都会触发 build, unit test, sonar 静态代码检查等. 每次发布时, 先发布到测试环境, 测试通过后, 再发布到正式环境. 测试环境的发布过程为: merge master branch 到 release branch, 并给 release branch 打一个名为 `{version}-test` 的 tag, 然后构建和部署. 正式环境的发布过程为: 给 release branch 打一个名为`{version}-prod`的 tag, 然后构建和部署.

## 项目配置
接下来我们谈谈, 一个具体项目的结构和配置.

### 结构

- .gitlab-ci.yml
- docker
- application.yml
本文同样以一个 hello world的 spring boot 应用为例介绍如何实现上述多环境持续部署流水线. 以下为该应用的目录结构.
```
├── .gitlab-ci.yml            # gitlab 持续集成脚本
├── docker
│   ├── Dockerfile            # 用来构建 docker 镜像
│   └── docker-compose.yml    # docker 服务编排文件, 用来把服务部署到阿里云容器集群上
├── pom.xml
├── scripts
│   ├── merge-to-release.sh     # 把代码从 master branch merge到 release branch
│   ├── prod-deploy.sh          # 部署到生产环境
│   └── test-deploy.sh          # 部署到测试环境
└── src
    ├── main
    │   └── java.ebao.hello
    │         └── BootApp.java  # 一个简单的 hello world 服务
    ├── resource                # spring应用配置文件
    │   ├── application.yml
    │   ├── application-dev.yml
    │   ├── application-prod.yml
    │   └── application-test.yml
    └── test
```

这个目录结构跟[上一篇]({{ site.url }}/ci-with-gitlab-and-aliyun-docker)中提到的结构有两点变化:
1. scripts 目录下多了 merge-to-release.sh 脚本, 和针对不同环境的部署脚本
2. src/main/resource 目录下多了不同环境的 spring 应用配置文件

下面逐一介绍各部分.
## 1. scripts目录
**merge-to-release.sh**, 把代码从 master branch merge到 release branch
```shell
set -e

# merge into release branch
git checkout -B release
git merge master
git add -A
git push
```

**test-deploy.sh**,  部署到测试环境(一个阿里云容器集群)
```shell
set -e

version=$1
if [[ -z $1 ]]; then
	echo "请输入版本号, 如: 1.0"
	exit 1
fi

git checkout release
git tag $version-test
git push --tags
git checkout master
```

**prod-deploy.sh**,  部署到正式环境(一个阿里云容器集群)
```shell
set -e

version=$1
if [[ -z $1 ]]; then
	echo "请输入版本号, 如: 1.0"
	exit 1
fi

git checkout release
git tag $version-prod
git push --tags
git checkout master
```

`test-deploy.sh`和`prod-deploy.sh`就是给 release branch 打个以`-test`或者`-prod`结尾的 tag, 然后再提交. 下面讲的`.gtilab-ci.yml`会根据tag运行不同环境的部署脚本.

## 2. .gitlab-ci.yml
```yaml
variables:
  PROJECT_NAME: hello-ci
  IMAGE_NAME: registry.xxx.com/$PROJECT_NAME
  JAR_FILE: ${PROJECT_NAME}-${CI_BUILD_TAG}.jar
  TEST_DOMAIN_SUFFIX: test.com
  PROD_DOMAIN_SUFFIX: prod.com

build:
  stage: build
  script:
    - mvn clean package
    - mvn sonar:sonar
  only:
    - master
  tags:
    - maven

test-deploy:
  variables:
    DOMAIN: ${PROJECT_NAME}.${TEST_DOMAIN_SUFFIX}
    PROFILE: test
    INSTANCES: '1'
  stage: deploy
  script:
    - mvn versions:set -DnewVersion=$CI_BUILD_TAG
    - mvn clean package
    - cp target/$JAR_FILE docker/app.jar
    - cd docker
    - docker build -t $IMAGE_NAME:$CI_BUILD_TAG . # build the docker image
    - docker push $IMAGE_NAME:$CI_BUILD_TAG
    - docker-compose pull
    - docker-compose -p $PROJECT_NAME up -d # start an application in 阿里云容器服务
  only:
    - /^.*-test$/
  except:
    - master
  tags:
    - test-docker

prod-deploy:
  stage: deploy
  variables:
    DOMAIN: ${PROJECT_NAME}.${PROD_DOMAIN_SUFFIX}
    PROFILE: prod
    INSTANCES: '2'
  script:
    - mvn versions:set -DnewVersion=$CI_BUILD_TAG
    - mvn clean package
    - cp target/$JAR_FILE docker/app.jar
    - cd docker
    - docker build -t $IMAGE_NAME:$CI_BUILD_TAG . # build the docker image
    - docker push $IMAGE_NAME:$CI_BUILD_TAG
    - docker tag $IMAGE_NAME:$CI_BUILD_TAG $IMAGE_NAME
    - docker push $IMAGE_NAME
    - docker-compose pull
    - docker-compose -p $PROJECT_NAME up -d # start a application in 阿里云容器服务
  only:
    - /^.*-prod$/
  except:
    - master
  tags:
    - prod-docker
```

文件中定义了三个 job: build, test-deploy, prod-deploy.
`build` 是一个持续集成job, 每次 master branch 的提交都胡触发该 job. 他会运行构建, unit test, sonar 代码分析.
`test-deploy`和`prod-deploy`分别部署测试和正式环境. 由于 gitlab-ci 语法目前不支持多个 only 条件的`AND`操作(only 下面的多个条件的关系是`OR`). 所以我们采用了下面的限制条件, 表示非 master branch 上的以`-prod`结尾的 tag 提交会触发`prod-deploy`job. 由于我们只有 release 和 master 两个 branch, 这就相当于只有 release branch 上的以`-prod`结尾的 tag 提交才会触发`prod-deploy`job.
```yaml
only:
  - /^.*-prod$/
except:
  - master
```

`test-deploy`和`prod-deploy`的定义几乎一样, 最大的区别是 `tags`字段, 分别为`test-docker`和`prod-docker`. 正如[上篇文章]({{ site.url }}/ci-with-gitlab-and-aliyun-docker)介绍的, 这分别对应两个 gitlab runner. 两个 runner 上分别配置了不同的环境变量, 连接不同的容器集群.

## 3. docker/Dockerfile
内容不变, 参考[上篇文章]({{ site.url }}/ci-with-gitlab-and-aliyun-docker)

## 4. docker/docker-compose.yml
```yaml
hello-ci:
  image: $IMAGE_NAME:$CI_BUILD_TAG
  labels:
    aliyun.scale: '$INSTANCES'
    aliyun.routing.port_8080: 'http://$DOMAIN'
  restart: always
  environment:
    - reschedule:on-node-failure
    - SPRING_PROFILES_ACTIVE=$PROFILE
```

这里`$INSTANCES`, `$DOMAIN`, `$PROFILE`都采用了变量, 如果你仔细看`.gitlab-ci.yml`中`test-deploy`和`prod-deploy`的定义, 会发现里面定义了这三个变量. 这样我们就能实现不同环境下起不同数量的容器实例, 使用不同的域名, 读取不同的 spring 配置文件.

## 5. gitlab-runner的配置
增加`test-docker`和`prod-docker`两个 runner 的配置, 具体可以参考参考[上篇文章]({{ site.url }}/ci-with-gitlab-and-aliyun-docker).

## 结束
至此, 基于gitlab 和阿里云容器服务的 spring 应用多环境持续部署就完成了.
基于这套流水线, 开发人员不需要登录具体的测试和生产环境进行部署, 大大提高了我们产品的部署和迭代速度.
