---
layout:     post
title:      Dubbo+Zookeeper脚手架搭建
subtitle:   Dubbo+Zookeeper脚手架搭建
date:       2019-10-29
author:     yyconstantine
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 微服务
---

# Dubbo+Zookeeper脚手架

---

近期新项目更改为使用dubbo+zookeeper搭建分布式服务，搭建完后进行记录。

---

#### 1.本地使用zookeeper搭建单机注册中心
- 下载：[zookeeper下载传送门👉](https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/)
- 下载并解压完成后，进行bin目录双击使用```zkServer.cmd```（Windows下）或```zkServer.sh```（Mac/Linux下），出现如下界面则证明启动成功：
    - ![zkServer-start.PNG](https://i.loli.net/2019/10/29/BbhtGEHF4UjQRXr.png)
    - 若启动失败/闪退，可在```zkServer.cmd```结尾添加pause，查看失败原因
- 到此，我们就已经在本地启动了一个zookeeper注册中心

---

#### 2.搭建一个简单的dubbo多服务项目
- 引入必要依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.0.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>me.sxl</groupId>
    <artifactId>dubbo-template</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>dubbo-template</name>
    <packaging>pom</packaging>

    <modules>
        <module>common</module>
        <module>consumer</module>
        <module>provider</module>
    </modules>

    <properties>
        <java.version>1.8</java.version>
        <dubbo.version>2.7.3</dubbo.version>
        <swagger.version>2.9.2</swagger.version>
        <zookeeper.version>0.11</zookeeper.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-zookeeper</artifactId>
            <version>${dubbo.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
                <exclusion>
                    <artifactId>log4j</artifactId>
                    <groupId>log4j</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>slf4j-api</artifactId>
                    <groupId>org.slf4j</groupId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
            </exclusions>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-dependencies-bom</artifactId>
            <version>${dubbo.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>${dubbo.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>javax.servlet</groupId>
                    <artifactId>servlet-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
                <exclusion>
                    <artifactId>spring-context</artifactId>
                    <groupId>org.springframework</groupId>
                </exclusion>
                <exclusion>
                    <groupId>org.yaml</groupId>
                    <artifactId>snakeyaml</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>${dubbo.version}</version>
        </dependency>

        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>${zookeeper.version}</version>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>${swagger.version}</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>${swagger.version}</version>
        </dependency>
        
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

- 创建服务提供方
    - 这里主要发布两个服务的接口，一个支付接口，一个支付流水接口
    
    ```java
    public interface PayService {
    
        /**
         * 发起支付
         * @return pay success
         */
        String pay();
    
    }
    ```
    ```java
    public interface PrintService {
    
        /**
         * 打印支付流水
         */
        void print();
    
    }
    ```
    
- 创建服务提供者
    - 配置文件如下：
   
   ```yaml
    server:
      port: 8081
    
    spring:
      application:
        name: provider-pay
    
    dubbo:
      scan:
        base-packages: me.sxl.providerpay.service
      application:
        name: ${spring.application.name}
      protocol:
        # 指定协议名称为dubbo
        name: dubbo
        # 随机使用自增的端口号,一般从20880开始
        port: -1
      registry:
        # 注册到本地的zookeeper,可指向自定义的ip和端口
        address: zookeeper://127.0.0.1:2181
    ```
    
    - 创建接口对应的实现类，注意这里的```@Service```不是Spring的注解
    
    ```java
    import me.sxl.common.pay.PayService;
    import org.apache.dubbo.config.annotation.Service;

    @Service
    public class PayServiceImpl implements PayService {
    
        @Override
        public String pay() {
            return "pay success";
        }
    
    }
    ```
    
    - 以上，我们完成了支付服务的dubbo接口，打印流水服务同理，只需要实现对应的service接口，并更改其tomcat端口号、项目名及dubbo的扫描路径
    
- 创建服务消费方
    - 对服务提供方的接口，我们分别在消费方中进行调用，注意通过```@Reference```的方式引入service
    
    ```java
    import me.sxl.common.pay.PayService;
    import org.apache.dubbo.config.annotation.Reference;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    public class PayController {
    
        /**
         * 在这里使用
         * @see org.apache.dubbo.config.annotation.Reference
         * 注解注入dubbo生产者提供的服务
         */
        @Reference
        private PayService payService;
    
        @GetMapping("/pay")
        public String pay() {
            return this.payService.pay();
        }
    
    }
    ```
    
    - 配置文件如下：
    
    ```yaml
    server:
      port: 8901
    
    spring:
      application:
        name: consume-pay
    
    dubbo:
      scan:
        base-packages: me.sxl.consumerpay.controller
      application:
        name: ${spring.application.name}
      registry:
        address: zookeeper://127.0.0.1:2181
    ```
    
    - 支付流水同理，在这里不再演示
    
- 至此，dubbo接口发布和调用都准备完毕，我们按照provider => consumer的顺序启动服务，并通过curl/postman的方式请求consumer发布的http接口即可验证正确性

---

#### 3.搭建dubbo-admin
按照上述步骤，我们已经可以对dubbo接口进行开发和调试了，但我们可以通过引入dubbo-admin便捷地管理dubbo服务
- 首先下载一个dubbo-admin的war包：[这里是传送门👉](https://pan.baidu.com/s/1X0QjtA7nwLn95p9cE-eiFw)
- 将war包放入tomcat/webapps目录下，在tomcat/bin下启动tomcat：```startup.bat/startup.sh```
- tomcat启动正常后，我们访问[http://localhost:8080/dubbo-admin-2.8.4](http://localhost:8080/dubbo-admin-2.8.4)，即可进入dubbo-admin界面，对我们的dubbo服务进行查看和管理

---

项目源码地址：[https://github.com/yyconstantine/Dubbo-Zookeeper-Demo](https://github.com/yyconstantine/Dubbo-Zookeeper-Demo)
