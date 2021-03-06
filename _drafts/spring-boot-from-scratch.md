---
layout: post-md
title: 'Spring Boot from Scratch'
author: 'Julien Sobczak'
date: '2016-03-11 09:15:00.000+01:00'
categories: inspect
tags:
- java
- framework
---


Implementing the framework using a Problem-Solution approach
------------------------------------------------------------


<blockquote>
  <p>"Spring Boot is just the kind of framework the Java community has been seeking for over a decade" -
  <em>Andrew Glover, Manager, Delivery Engineering at Netflix</em></p>
</blockquote>


<div class="tip">
  <p class="title">What you will learn?</p>
  <ul>
    <li>The internals of Spring Boot to help you fix bean definition problems that arise in practice.</li>
    <li>How Spring Boot use the <code>@Conditional</code> annotation to support auto-configuration.</li>
    <li>Advanced Java techniques like custom <code>ClassLoader</code> writing.</li>
     <li>Inspect a .class file using the ASM library.</li>
    <li>Create an uber-jar without using any Maven plugin.</li>
  </ul>
</div>



<div class="licence">
  <p>
    <a href="https://github.com/spring-projects/spring-boot/blob/v1.3.3.RELEASE/">Spring Boot</a> is published
    under the <a href="https://opensource.org/licenses/Apache-2.0">licence Apache, Version 2.0</a>.
    The code presented in this post was simplified (robustness, performance, validation, etc) and should not be used outside of this context. This post is based on the version 1.3.3.RELEASE of Spring Boot.
  </p>
</div>



### Prerequisites

We will follow the [official Getting Started Guide "Building an Application with Spring Boot"](https://spring.io/guides/gs/spring-boot/) and explain the inner working of the framework by recoding its main features. The resulting code has no vocation to be use outside of this post. Let's go!

Note: if you are just beginning with Spring Boot, take the time (around 15 minutes) to try the Getting Starting Guide before continuing.

The tutorial is composed of only 3 files: a `pom.xml` to reference Spring Boot dependencies (starters, Maven Plugin, Parent POM, ...), a basic `HelloController` to exercise the Web Starter and an `Application` class containing the main method to bootstrap our application. Here is the code source of these files.

{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>fr.imlovinit</groupId>
    <artifactId>spring-boot-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.3.3.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
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
{% endhighlight %}

{% highlight java %}
package hello;

import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMapping;

@RestController
public class HelloController {

    @RequestMapping("/")
    public String index() {
        return "Greetings from Spring Boot!";
    }

}
{% endhighlight %}

{% highlight java %}
package hello;

import java.util.Arrays;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(Application.class, args);

        System.out.println("Let's inspect the beans provided by Spring Boot:");

        String[] beanNames = ctx.getBeanDefinitionNames();
        Arrays.sort(beanNames);
        for (String beanName : beanNames) {
            System.out.println(beanName);
        }
    }

}
{% endhighlight %}

To launch the application, simply run the command:

{% highlight console %}
$ mvn spring-boot:run
{% endhighlight %}

We now have a fully-functional web application using Spring Boot. We will rewrite it progressively to remove the Spring Boot modules while keeping this example operational. Let's go!



### Spring Boot Essentials


Spring Boot brings a great deal of magic to Spring application development by performing:

+ **Automatic configuration** — Spring Boot can automatically provide configuration for application functionality common to many Spring applications.
+ **Starter dependencies** — You tell Spring Boot what kind of functionality you need, and it will ensure that the libraries needed are added to the build.


Note: Spring Boot brings other interesting features like the **Actuator** (gives you insight into what’s going on inside of the application) and the **command-line interface** (lets you write complete applications with no need for a traditional project build). These features are not described in this post but relied heavily on the two main points.


Before delving into the framework, let's take a high-level view of Spring Boot and its main components. The following diagram presents the main dependencies in the context of our pom.xml:



[TODO includes diagram]



This diagram highlights the three parts that composed this article :

- Creating your Spring Boot Application, describes why our code compiles though we only have added one dependency. This part inspect the different POM files.

- Running your Spring Boot Application, presents the core logic behind the magic of Spring Boot: how beans are auto-loaded, how a Tomcat instance is automatically bootstrapped.

- Packaging your Spring Boot Application, describes how the Spring Boot Maven plugin works under the hood to create the executable uber-jar.


Depending your interest, you could safely jump from one part to another.


## Part 1: Creating your Spring Boot Application


We all have lost a considerable time trying to understand why a dependency class could not be found, or trying to make two unrelated projects work together although they need two incompatible versions of the same library. Using Maven exclusions help but it is not a satisfactory solution. These dependency issues are known under the term [Dependency Hell](https://en.wikipedia.org/wiki/Dependency_hell) to reflect the frustration of many software developers.


Even the most simple Java program needs many dependencies. Here is the list of dependencies for our Getting Started example:

{% highlight console %}
fr.imlovinit:spring-boot-demo:jar:0.0.1-SNAPSHOT
 +- ch.qos.logback:logback-classic:jar:1.1.5
 |   +- ch.qos.logback:logback-core:jar:1.1.5
 |   \- org.slf4j:slf4j-api:jar:1.7.16
 |       +- org.slf4j:jcl-over-slf4j:jar:1.7.16
 |       +- org.slf4j:jul-to-slf4j:jar:1.7.16
 |       \- org.slf4j:log4j-over-slf4j:jar:1.7.16
 +- org.springframework:spring-core:jar:4.2.5.RELEASE
 |   \- org.yaml:snakeyaml:jar:1.16
 +- org.apache.tomcat.embed:tomcat-embed-core:jar:8.0.32
 +- org.apache.tomcat.embed:tomcat-embed-el:jar:8.0.32
 +- org.apache.tomcat.embed:tomcat-embed-logging-juli:jar:8.0.32
 |   \- org.apache.tomcat.embed:tomcat-embed-websocket:jar:8.0.32
 +- org.hibernate:hibernate-validator:jar:5.2.4.Final
 |   +- javax.validation:validation-api:jar:1.1.0.Final
 |   +- org.jboss.logging:jboss-logging:jar:3.3.0.Final
 |   \- com.fasterxml:classmate:jar:1.1.0
 +- com.fasterxml.jackson.core:jackson-databind:jar:2.6.5
 |   +- com.fasterxml.jackson.core:jackson-annotations:jar:2.6.5
 |   \- com.fasterxml.jackson.core:jackson-core:jar:2.6.5
 +- org.springframework:spring-web:jar:4.2.5.RELEASE
 |   +- org.springframework:spring-aop:jar:4.2.5.RELEASE
 |   |   \- aopalliance:aopalliance:jar:1.0
 |   +- org.springframework:spring-beans:jar:4.2.5.RELEASE
 |   \- org.springframework:spring-context:jar:4.2.5.RELEASE
 \- org.springframework:spring-webmvc:jar:4.2.5.RELEASE
     \- org.springframework:spring-expression:jar:4.2.5.RELEASE
{% endhighlight %}

That's a lot of dependencies for such a simple project.


### Problem: I don't want to specify all of my dependencies!

Spring Boot solution is a two-part solution:

- Starters are a set of convenient dependency descriptors that you can include in your application. You get a one-stop-shop for all the Spring and related technology that you need without having to hunt through sample code and copy paste loads of dependency descriptors. For example, if you want to get started using Spring and JPA for database access just include the `spring-boot-starter-data-jpa` dependency in your project, and you are good to go.

- Provide a curated list of dependencies. In practice, you do not need to provide a version for any of these dependencies in your build configuration as Spring Boot is managing that for you. When you upgrade Spring Boot itself, these dependencies will be upgraded as well in a consistent way. The dependencies versions are known to works together in a seamless way.


Let's inspect the dependencies on the Getting Started example by running the command `mvn dependency:tree`. We get the following output:

{% highlight console %}
fr.imlovinit:spring-boot-demo:jar:0.0.1-SNAPSHOT
 \- org.springframework.boot:spring-boot-starter-web:jar:1.3.3.RELEASE
    +- org.springframework.boot:spring-boot-starter:jar:1.3.3.RELEASE
    |  +- org.springframework.boot:spring-boot:jar:1.3.3.RELEASE
    |  +- org.springframework.boot:spring-boot-autoconfigure:jar:1.3.3.RELEASE
    |  +- org.springframework.boot:spring-boot-starter-logging:jar:1.3.3.RELEASE
    |  |  +- ch.qos.logback:logback-classic:jar:1.1.5
    |  |  |  +- ch.qos.logback:logback-core:jar:1.1.5
    |  |  |  \- org.slf4j:slf4j-api:jar:1.7.16
    |  |  +- org.slf4j:jcl-over-slf4j:jar:1.7.16
    |  |  +- org.slf4j:jul-to-slf4j:jar:1.7.16
    |  |  \- org.slf4j:log4j-over-slf4j:jar:1.7.16
    |  +- org.springframework:spring-core:jar:4.2.5.RELEASE
    |  \- org.yaml:snakeyaml:jar:1.16
    +- org.springframework.boot:spring-boot-starter-tomcat:jar:1.3.3.RELEASE
    |  +- org.apache.tomcat.embed:tomcat-embed-core:jar:8.0.32
    |  +- org.apache.tomcat.embed:tomcat-embed-el:jar:8.0.32
    |  +- org.apache.tomcat.embed:tomcat-embed-logging-juli:jar:8.0.32
    |  \- org.apache.tomcat.embed:tomcat-embed-websocket:jar:8.0.32
    +- org.springframework.boot:spring-boot-starter-validation:jar:1.3.3.RELEASE
    |  \- org.hibernate:hibernate-validator:jar:5.2.4.Final
    |     +- javax.validation:validation-api:jar:1.1.0.Final
    |     +- org.jboss.logging:jboss-logging:jar:3.3.0.Final
    |     \- com.fasterxml:classmate:jar:1.1.0
    +- com.fasterxml.jackson.core:jackson-databind:jar:2.6.5
    |  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.6.5
    |  \- com.fasterxml.jackson.core:jackson-core:jar:2.6.5
    +- org.springframework:spring-web:jar:4.2.5.RELEASE
    |  +- org.springframework:spring-aop:jar:4.2.5.RELEASE
    |  |  \- aopalliance:aopalliance:jar:1.0
    |  +- org.springframework:spring-beans:jar:4.2.5.RELEASE
    |  \- org.springframework:spring-context:jar:4.2.5.RELEASE
    \- org.springframework:spring-webmvc:jar:4.2.5.RELEASE
       \- org.springframework:spring-expression:jar:4.2.5.RELEASE
{% endhighlight %}

All of these dependencies are inherited from the dependency org.springframework.boot:spring-boot-starter-web present in our pom.xml:

{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">`

    <modelVersion>4.0.0</modelVersion>
    <groupId>fr.imlovinit</groupId>
    <artifactId>spring-boot-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.3.3.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
{% endhighlight %}

The POM is concise but we could notice we have 3 dependencies on Spring Boot :
- We inherit from a parent pom `spring-boot-starter-parent`
- We depend on the starter `spring-boot-starter-web`
- We configure a Maven plugin `spring-boot-maven-plugin`

We should analyze each of these components to understand what they are doing exactly.


#### The parent POM.

{% highlight xml %}
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.3.3.RELEASE</version>
</parent>
{% endhighlight %}

This [project](https://github.com/spring-projects/spring-boot/tree/v1.3.3.RELEASE/spring-boot-starters/spring-boot-starter-parent) is a Maven project of type pom. The only file present is the pom.xml. Here is a simplified version:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>1.3.3.RELEASE</version>
    </parent>
    <artifactId>spring-boot-starter-parent</artifactId>
    <packaging>pom</packaging>
    <name>Spring Boot Starter Parent</name>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-core</artifactId>
                <version>${spring.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                            </goals>
                        </execution>
                    </executions>
                    <configuration>
                        <mainClass>${start-class}</mainClass>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
{% endhighlight %}

The full source includes other Maven plugins to package a WAR (useful to deploy on a standalone server), to run the project directly from Maven ({maven-exec-plugin}), to include information about the last Git commit ({git-commit-id-plugin}), or an alternative to the {spring-boot-maven-plugin} based on the {maven-shade-plugin}.

This pom is mainly used to declare the version of the Spring Framework and the version of the Spring Boot Maven Plugin (the subject of this post's second part). We notice another parent pom but be assured, this is the last pom of the hierarchy.

{% highlight xml %}
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>1.3.3.RELEASE</version>
</parent>
{% endhighlight %}

The project `spring-boot-dependencies` is also a project of type pom. This project aims to centralize all of the [dependency versions that are provided by Spring Boot](http://docs.spring.io/spring-boot/docs/1.3.3.RELEASE/reference/html/appendix-dependency-versions.html). This represents more than 400 dependencies (more than 2000 lines of code!). Here is a shortened version, adapted for our Hello World project:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>1.3.3.RELEASE</version>
    <packaging>pom</packaging>
    <name>Spring Boot Dependencies</name>
    <properties>
        <!-- Inherited from spring-boot-dependencies -->
        <spring-boot.version>1.3.3.RELEASE</spring-boot.version>
        <jersey.version>2.22.2</jersey.version>
        <logback.version>1.1.5</logback.version>
        <slf4j.version>1.7.16</slf4j.version>
        <spring.version>4.2.5.RELEASE</spring.version>
        <!-- And 400 others dependencies... -->
    </properties>
    <prerequisites>
        <maven>3.2.1</maven>
    </prerequisites>
    <dependencyManagement>
        <dependencies>
            <!-- Spring Boot -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot</artifactId>
                <version>1.3.3.RELEASE</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter</artifactId>
                <version>1.3.3.RELEASE</version>
                <exclusions>
                    <exclusion>
                        <groupId>commons-logging</groupId>
                        <artifactId>commons-logging</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
                <version>1.3.3.RELEASE</version>
            </dependency>
            <!-- Third Party -->
            <dependency>
                <groupId>ch.qos.logback</groupId>
                <artifactId>logback-classic</artifactId>
                <version>${logback.version}</version>
            </dependency>
            <!-- Spring Framework -->
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-core</artifactId>
                <version>${spring.version}</version>
            </dependency>
            <!-- 400 others dependencies... -->
        </dependencies>
    </dependencyManagement>
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.1</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
{% endhighlight %}

These dependencies are declared in the <dependencyManagement> and <pluginManagement> sections, so no dependency is directly inherited when using this parent POM. This only save you from specifying the version declaring a dependency in a <dependencies> section. But this is more than just a one-line saver. All the versions defined by Spring Boot are known to work seamlessly together! This solves the issue of [dependency hell][dependency-hell] where two libraries needs two incompatible versions of the same dependency.


Now that we know how the version of dependencies is determined, we should find how who is behind of all our dependencies.

Given the output of the previous Maven dependency tree, we know the starters are responsible from these dependencies. But what is a exactly a Starter?



#### The Starters


According the official documentation, Spring Boot Starters are a set of convenient dependency descriptors that you can include in your application. You get a one-stop-shop for all the Spring and related technology that you need without having to hunt through sample code and copy paste loads of dependency descriptors.

In our sample, we want to create a web application, so, we just include the {spring-boot-starter-web} dependency in our project.

{% highlight xml %}
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
{% endhighlight %}

The structure of a starter respects the following organization:

{% highlight console %}
spring-boot-starter-web
 +- src/main/resources/META-INF
 |  \+ spring.provides
 \+ pom.xml
{% endhighlight %}

That's all. Just one pom.xml (the file {spring.provides} could be ignored). So, let's see what contains this pom.xml:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starters</artifactId>
        <version>1.3.3.RELEASE</version>
    </parent>
    <artifactId>spring-boot-starter-web</artifactId>
    <name>Spring Boot Web Starter</name>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
        </dependency>
    </dependencies>
</project>
{% endhighlight %}

We finally find where our dependencies comes from. A starter exploit the `<dependencyManagement>` section defined previously to omit the version of the artifacts. We also see that a starter could reference other starters to inherit its dependencies transitively. For example, the starter Web depends on the starter Tomcat. The dependency `org.springframework.boot:spring-boot-starter` is important. All starters depends on it because this is the dependency who load the core Spring Boot dependency (presented in the last part of this article), containing in particular the annotation `@SpringBootApplication` we used on our main class `Application`.



So, the combination of the Parent POM and the Starters give us all the dependencies we need to write and compiles your application.
But what happens when I run my application.


## Part 2: Running your Spring Boot Application


In its most basic form, a Spring Application is a simple class annotated with the `@SpringBootApplication` annotation and declaring a simple main method like this:

{% highlight java %}
package hello;

import org.springframework.boot.*;
import org.springframework.context.ApplicationContext;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
{% endhighlight %}

Behind this simple API lies the most complex module of the framework, the Spring Boot core module, where the true magic happens: the Auto-configuration feature!


### Problem: I don't want to declare boilerplate beans like infrastructure components!

Let's take the example of Spring MVC. Before writing our controllers, some required beans should be defined first:

{% highlight java %}
package hello;

import org.springframework.context.annotation.*;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.view.InternalResourceViewResolver;
import org.springframework.web.servlet.view.JstlView;

@Configuration
@EnableWebMvc
@ComponentScan
public class HelloWorldConfiguration {

    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setViewClass(JstlView.class);
        viewResolver.setPrefix("/WEB-INF/views/");
        viewResolver.setSuffix(".jsp");
        return viewResolver;
    }

}
{% endhighlight %}

The code is concise thanks to the `@EnableWebMvc` annotation, who defines a instance of `RequestMappingHandlerMapping`, a list of `HttpMessageConverter` converters and many other beans.
Before Spring Boot, this annotation was a great time-saver. The last thing we need to to is register the Spring MVC servlet:

{% highlight java %}
package hello;

import javax.servlet.*;

import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;

public class HelloWorldInitializer implements WebApplicationInitializer {

    public void onStartup(ServletContext container) throws ServletException {

        AnnotationConfigWebApplicationContext ctx =
            new AnnotationConfigWebApplicationContext();
        ctx.register(HelloWorldConfiguration.class);
        ctx.setServletContext(container);

        ServletRegistration.Dynamic servlet =
            container.addServlet("dispatcher", new DispatcherServlet(ctx));

        servlet.setLoadOnStartup(1);
        servlet.addMapping("/");
    }

}
{% endhighlight %}

`AnnotationConfigWebApplicationContext` is created. It's `WebApplicationContext` implementation that looks for Spring configuration in classes annotated with `@Configuration` annotation. We help it by providing our configuration class directly with the method `register` before setting the `ServletContext` to link the `WebApplicationContext` to the lifecycle of `ServletContext`.
`DispatcherServlet` is then created, initialized, mapped to `"/*"` and set to eagerly load on application startup. This configuration looks what we do when using the web.xml file instead.


This basic example illustrates perfectly what we have to do each time we add a new Spring module or a new library to our classpath.


How could we do to reduce this configuration to nothing?

The first step is to define reusable `Configuration` classes that could be easily imported inside our application.
If we look inside the source of the module [Spring Boot Autoconfigure](https://github.com/spring-projects/spring-boot/tree/v1.3.3.RELEASE/spring-boot-autoconfigure), we find one packages for each Spring modules (batch, mvc, integration, ...) or supported technology (flyway, hazelcast, cassandra, ...) containing dozens of these `Configuration` classes. The content is very similar to the class we previously write:

{% highlight java %}
package org.springframework.boot.autoconfigure.web;

import org.springframework.context.annotation.*;
import org.springframework.web.servlet.config.annotation.DelegatingWebMvcConfiguration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.view.InternalResourceViewResolver;

@Configuration
@Import(EnableWebMvcConfiguration.class)
public class WebMvcAutoConfiguration extends WebMvcConfigurerAdapter {

    @Bean
    public InternalResourceViewResolver defaultViewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/");
        resolver.setSuffix(".jsp");
        return resolver;
    }

    /**
     * Configuration equivalent to {@code @EnableWebMvc}.
     */
    @Configuration
    public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {
    }

}
{% endhighlight %}

To use these classes, we just need to add the import declaration like this:


{% highlight java %}
package hello;

import org.springframework.boot.*;
import org.springframework.context.ApplicationContext;

@Import(WebMvcAutoConfiguration.class)
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
{% endhighlight %}

### Problem: I don't want to `@Import` Spring specific `Configuration` classes. I even do not know about them!

The obvious approach is to automatically load all the classes. This is what Spring Boot does internally but don't panic, not all beans definitions will be loaded (we will talk about conditional beans in the next section).

To avoid having a hundred of `@Import` declarations, Spring Boot exploits a particularity of this annotating. `@Import` allows as value an implementation of `ImportSelector`, in addition to the classic `@Configuration` class. The declaration of the `ImportSelector` interface follows:

{% highlight java %}
package org.springframework.context.annotation;

public interface ImportSelector {

    /**
     * Select and return the names of which class(es) should be imported based on
     * the {@link AnnotationMetadata} of the importing @{@link Configuration} class.
     */
    String[] selectImports(AnnotationMetadata importingClassMetadata);

}
{% endhighlight %}

With this interface, we could easily load a set of `@Configuration` classes but we still don't know how to find them.

The solution adopted by Spring Boot is to reuse a Spring Core properties files named spring.factories present under the META-INF folder. Spring Boot add a new property `org.springframework.boot.autoconfigure.EnableAutoConfiguration`. Here is an example:

{% highlight java %}
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration, \
...
{% endhighlight %}

The file lists the configuration classes under the EnableAutoConfiguration key. All jar present in the classpath are checked by Spring Boot at startup even if Spring Boot provide a unique file containing for its own use.


Tip: We still have not finished to cover Spring Boot but you already know how create your [own starter](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-auto-configuration.html#boot-features-custom-starter). Just create a `Configuration` class and provide a spring.factories file to make your class a candidate to autoloading. That's all!


The class `EnableAutoConfigurationImportSelector` load theses files and register the classes in the application context. This class implements `ImportSelector` to be used with the `@Import`.

{% highlight java %}
package org.springframework.boot.autoconfigure;

import java.io.*;
import java.util.*;
import org.springframework.beans.factory.BeanClassLoaderAware;
import org.springframework.context.annotation.ImportSelector;
import org.springframework.core.io.support.SpringFactoriesLoader;
import org.springframework.core.type.AnnotationMetadata;

public class EnableAutoConfigurationImportSelector
        implements ImportSelector, BeanClassLoaderAware {

    private ClassLoader beanClassLoader;

    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        try {
            List<String> configurations = getCandidateConfigurations();
            return configurations.toArray(new String[configurations.size()]);
        }
        catch (IOException ex) {
            throw new IllegalStateException(ex);
        }
    }

    private List<String> getCandidateConfigurations() {
        return SpringFactoriesLoader.loadFactoryNames(
                EnableAutoConfiguration.class, getBeanClassLoader());
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.beanClassLoader = classLoader;
    }

}
{% endhighlight %}

The class `SpringFactoriesLoader` is a wrapper around the spring.factories file and provide a method to extract a property value for a Class object. The fully qualified name of the given class is used as the property name. Note: This class is provided by Spring Core and is not Spring Boot specific.


But, does it means taht all the beans (Flyway, Spring Integration, ...) will be loaded even If I do not use the related projects?

Let's reconsider the Spring MVC autoconfiguration class:


{% highlight java %}
package org.springframework.boot.autoconfigure.web;

import org.springframework.context.annotation.*;
import org.springframework.web.servlet.config.annotation.DelegatingWebMvcConfiguration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.view.InternalResourceViewResolver;

@Configuration
@Import(EnableWebMvcConfiguration.class)
public class WebMvcAutoConfiguration extends WebMvcConfigurerAdapter {

    @Bean
    public InternalResourceViewResolver defaultViewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/");
        resolver.setSuffix(".jsp");
        return resolver;
    }

    /**
     * Configuration equivalent to {@code @EnableWebMvc}.
     */
    @Configuration
    public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {
    }

}
{% endhighlight %}

The class `WebMvcConfigurerAdapter` is directly referenced by our configuration class. This class is defined by the Spring MVC project and as we have seen in the first part of this article, this dependency is inherited when using the Spring Boot Starter Web.

Imagine for a moment that you are writing a command-line application. We do not have the starter web in your dependencies. At startup, if Spring Boot load the `WebMvcAutoConfiguration` class (and we just saw it does), the JVM will crash with a `NoClassDefFoundError` because the superclass `WebMvcConfigurerAdapter` is not on our classpath. So, how does Spring Boot to avoid this error?

Here is the solution:

{% highlight java %}
package org.springframework.boot.autoconfigure.web;

import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
...

@Configuration
@ConditionalOnClass({ DispatcherServlet.class, WebMvcConfigurerAdapter.class })
public class WebMvcAutoConfiguration {

    @Configuration
    @Import(EnableWebMvcConfiguration.class)
    public static class WebMvcAutoConfigurationAdapter extends WebMvcConfigurerAdapter {

        @Bean
        public InternalResourceViewResolver defaultViewResolver() {
            InternalResourceViewResolver resolver = new InternalResourceViewResolver();
            resolver.setPrefix("/WEB-INF/");
            resolver.setSuffix(".jsp");
            return resolver;
        }

    }

    @Configuration
    public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {
    }

}
{% endhighlight %}

Did you spot the difference with the previous definition of this class? The solution is a two-part solution.

First, we arrange to not have any external dependencies on the outer class definition. As a configuration class could contain other inner configuration classes, the dependency on Spring MVC could be easily pushed down in an inner class. For now, this is a noop operation.

Second, we use a new feature in Spring 4, the `@Conditional` annotation. This annotation indicates that a component is only eligible for registration when all specified conditions match. Otherwise, the component will not be registered and inner classes will never be loaded by the class loader. In this way, no `NoClassDefFoundError` will be thrown when Spring MVC is not on the classpath.

Note: The inner class technique is only required when extending a third-party library. When defining beans, the `@Conditional` is enough:

{% highlight java %}
package org.springframework.boot.autoconfigure.cassandra;

import com.datastax.driver.core.Cluster;

@Configuration
@ConditionalOnClass({ Cluster.class })
public class CassandraAutoConfiguration {

    @Bean
    public Cluster cluster() {
        return ...:
    }

}
{% endhighlight %}

While the method `cluster` is not called, no error is thrown by the JVM. This is the role of the annotation `@ConditionalOnClass` to prevent the method from being called. This annotation is provided by Spring Boot but rely heavily on the mechanism provided by Spring Core. Here is its declaration:

{% highlight java %}
package org.springframework.boot.autoconfigure.condition;

@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {

    Class<?>[] value() default {};
}
{% endhighlight %}


The `ConditionalOnClass` is annotated with the Spring Core `Conditional` annotation whose only required attribute is a reference to an implementation of the `org.springframework.context.annotation.Condition` class. Here is the implementation of the class `OnClassCondition`:

{% highlight java %}
package org.springframework.boot.autoconfigure.condition;

import java.util.*;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
import org.springframework.util.ClassUtils;
import org.springframework.util.MultiValueMap;

class OnClassCondition implements Condition {

    @Override
    public final boolean matches(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        MultiValueMap<String, Object> onClasses = getAttributes(metadata,
                ConditionalOnClass.class);
        return matches(onClasses, context);
    }

    private MultiValueMap<String, Object> getAttributes(AnnotatedTypeMetadata metadata,
            Class<?> annotationType) {
        return metadata.getAllAnnotationAttributes(annotationType.getName(), true);
    }

    private boolean matches(MultiValueMap<String, Object> attributes,
            ConditionContext context) {
        for (Object classObject : attributes.get("value")) {
            String className = (String) classObject;
            if (!ClassUtils.isPresent(className, context.getClassLoader())) {
                return false;
            }
        }
        return true;
    }

}
{% endhighlight %}

A new instance of this class will be created for each class annotated with `@ConditionalOnClass`. Internally, we use the class `AnnotatedTypeMetadata` provided by Spring Core to access to the annotations of a specific type in a form that does not necessarily require the class-loading. This behavior explains why extending a missing class is problematic but referencing a missing class inside this annotation is not:

{% highlight java %}
@Configuration
@ConditionalOnClass({ WebMvcConfigurerAdapter.class }) // OK.
public class WebMvcAutoConfiguration extends WebMvcConfigurerAdapter { // KO
    ...
}
{% endhighlight %}

Before going to the next problem, let's add a little syntactic sugar with the `@EnableAutoConfiguration` annotation:

{% highlight java %}
package org.springframework.boot.autoconfigure;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

}
{% endhighlight %}

With this annotation, users don't need to import the `EnableAutoConfigurationImportSelector` class to load all the configuration classes. We can go further and create the `@SpringBootApplication` annotation whose definition follows:


{% highlight java %}
package org.springframework.boot.autoconfigure;

import java.lang.annotation.*;
import org.springframework.context.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication {

}
{% endhighlight %}

`@SpringBootApplication` is an example of composed annotation, an annotation whose role is to aggregate multiple annotations to provide a more convenient API for the user. Here, we combine the auto-configuration annotation we just defined before, the `@ComponentScan` to load user bean definitions and the `@Configuration` to allow the user to define beans directly in the class annotated by `@SpringBootApplication`.



### Problem: But sometimes I need to override auto-configured bean definitions?

Spring Boot try to apply common defaults when defining beans but rapidly you need to tune this configuration. The good news is it is really easy to do: just defining a bean of the same type (or sometimes of the same name) in your configuration is enough for Spring Boot to detect it and not register its own bean.

Does the @Conditional annotation seems the right choice to you? Of course, it is! Spring Boot module makes heavy use of `@Conditional` annotation. We already saw the `@ConditionalOnClass` annotation. The other most used annotation is the `@ConditionalOnMissingBean` annotation, that only matches when the specified bean classes and/or names are not already contained in the `BeanFactory`. Rather than see the implementation of this annotation that is very close to the implementation of the `OnClassCondition` class, let's look at its use in the `WebMvcAutoConfiguration` class again:

{% highlight java %}
@Configuration
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class,
        WebMvcConfigurerAdapter.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
public class WebMvcAutoConfiguration {

    @Configuration
    @Import(EnableWebMvcConfiguration.class)
    public static class WebMvcAutoConfigurationAdapter extends WebMvcConfigurerAdapter {

        @Bean
        @ConditionalOnMissingBean
        public InternalResourceViewResolver defaultViewResolver() {
            // same as before
        }
    }

}
{% endhighlight %}

With this class declaration, we have two possibilities if the default configuration does not fit your needs. We could simply define a bean of type `InternalResourceViewResolver`, for example in our main classs

{% highlight java %}
package hello;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    public InternalResourceViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/myfolder/");
        resolver.setSuffix(".view");
        return resolver;
    }

}
{% endhighlight %}

The second solution is to completely override the Spring MVC definition by extending from `WebMvcConfigurationSupport` (see the `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class` on the class declaration).


However, one thing still needs to be resolved. How to be sure our configuration class will be considered before the condition on the missing bean is executed. Indeed, if the condition on the `defaultViewResolver` method is executed before our class is loaded, there will be not existing bean and the instance of `ViewResolver` will be created. That will result in two beans of the same type in the final application context.

Once again, we will use a feature of Spring Core to help us. When we created the `EnableAutoConfigurationImportSelector` class, the class responsible to load all auto-configuration classes, we inherited from the `ImportSelector` interface. Spring Core provides a variation that runs after all `@Configuration` beans have been processed. This type of selector is particularly useful when imports are `@Conditional` as with Spring Boot. This interface is `DeferredImportSelector`:


{% highlight java %}
public class EnableAutoConfigurationImportSelector implements DeferredImportSelector,
        BeanClassLoaderAware{
    ...
}
{% endhighlight %}

That's all there is to change! We now have finished implementing the auto-configuration, probably the most powerful feature of Spring Boot, but also the most obscure one.



### Problem: I want to start my web application from a simple Main method!

Popular Web containers like Tomcat and Jetty all supports an embedded mode. This mode let us start a web application without having to install a standalone server before. This is really great for testing and Spring Boot decides to go further by deploying your application along an embedded server, a must for your microservices, and a good alternative to concurrent product like DropWizard.


Spring Boot supports running your application in an embedded server. This functionality is organized around the `EmbeddedServletContainer` abstraction:


{% highlight java %}
package org.springframework.boot.context.embedded;

public interface EmbeddedServletContainer {

    void start();

    void stop();

}
{% endhighlight %}

To support multiple servers, Spring Boot creates another abstraction with the `EmbeddedServletContainerFactory` class. Its role is to instantiate an `EmbeddedServletContainer` (there exists an implementation for each supported server). Spring Boot uses the auto-configuration mechanism to retrieve the instance to use at startup. For example, if the tomcat jar is present in the classpath, the `TomcatEmbeddedServletContainerFactory` will be used to create the server instance. Here is the class responsible to instantiate the factory:



{% highlight java %}
package org.springframework.boot.autoconfigure.web;

import org.apache.catalina.startup.Tomcat;
import org.eclipse.jetty.server.Server;
import io.undertow.Undertow;

import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.context.embedded.tomcat.*;
import org.springframework.boot.context.embedded.jetty.*;
import org.springframework.boot.context.embedded.undertow.*;
import org.springframework.context.annotation.*;

@Configuration
public class EmbeddedServletContainerAutoConfiguration {

    /**
     * Nested configuration for if Tomcat is being used.
     */
    @Configuration
    @ConditionalOnClass({ Tomcat.class })
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class)
    public static class EmbeddedTomcat {

        @Bean
        public EmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
            return new TomcatEmbeddedServletContainerFactory();
        }

    }

    /**
     * Nested configuration if Jetty is being used.
     */
    @Configuration
    @ConditionalOnClass({ Server.class })
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class)
    public static class EmbeddedJetty {

        @Bean
        public EmbeddedServletContainerFactory jettyEmbeddedServletContainerFactory() {
            return new JettyEmbeddedServletContainerFactory();
        }

    }

    /**
     * Nested configuration if Undertow is being used.
     */
    @Configuration
    @ConditionalOnClass({ Undertow.class })
    @ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class)
    public static class EmbeddedUndertow {

        @Bean
        public EmbeddedServletContainerFactory undertowEmbeddedServletContainerFactory() {
            return new UndertowEmbeddedServletContainerFactory();
        }

    }

}
{% endhighlight %}

For this post, we will ignore these abstractions and create directly an instance of `TomcatEmbeddedServletContainer`. Internally, this class uses the Tomcat API to start a new server, register web components like servlets or filters. We will not go into detail about the different properties. Please refer to the Tomcat documentation for more information.

Here is the implementation:

{% highlight java %}
package org.springframework.boot.context.embedded.tomcat;

import java.io.File;
import java.io.IOException;
import java.util.Collections;

import javax.servlet.ServletContainerInitializer;

import org.apache.catalina.*;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.loader.WebappLoader;
import org.apache.catalina.startup.Tomcat;
import org.apache.catalina.startup.Tomcat.FixContextListener;
import org.springframework.boot.context.embedded.EmbeddedServletContainer;
import org.springframework.util.ClassUtils;

public class TomcatEmbeddedServletContainer implements EmbeddedServletContainer {

    private static String CONTEXT_PATH = "";
    private static int PORT = 8080;

    private final Tomcat tomcat;

    public TomcatEmbeddedServletContainer(ServletContainerInitializer ... initializers) {
        this.tomcat = createTomcat(initializers);
        initialize();
    }

    private Tomcat createTomcat(ServletContainerInitializer ... initializers) {
        Tomcat tomcat = new Tomcat();
        File baseDir = createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setPort(PORT);
        connector.setURIEncoding("UTF-8");
        tomcat.getService().addConnector(connector);
        tomcat.setConnector(connector);
        tomcat.getHost().setAutoDeploy(false);
        tomcat.getEngine().setBackgroundProcessorDelay(-1);
        prepareContext(tomcat.getHost(), initializers);
        return tomcat;
    }

    private void prepareContext(Host host, ServletContainerInitializer[] initializers) {
        StandardContext context = new StandardContext();
        context.setName(CONTEXT_PATH);
        context.setPath(CONTEXT_PATH);
        context.addLifecycleListener(new FixContextListener());
        context.setParentClassLoader(ClassUtils.getDefaultClassLoader());
        context.setUseRelativeRedirects(false);
        context.setMapperContextRootRedirectEnabled(true);
        context.setLoader(new WebappLoader(context.getParentClassLoader()));
        for (ServletContainerInitializer initializer : initializers) {
            context.addServletContainerInitializer(initializer, Collections.emptySet());
        }
        host.addChild(context);
    }

    private synchronized void initialize() {
        try {
            tomcat.start();
        } catch (LifecycleException e) {
            throw new IllegalStateException("Unable to start Tomcat", e);
        }
    }

    @Override
    public void start(){
        // already started by initialize method()
    }

    @Override
    public synchronized void stop() {
        try {
            this.tomcat.stop();
            this.tomcat.destroy();
        }
        catch (LifecycleException ex) {
            // swallow and continue
        }
    }

    private File createTempDir(String prefix) {
        try {
            File tempDir = File.createTempFile(prefix + ".", "." + PORT);
            tempDir.delete();
            tempDir.mkdir();
            tempDir.deleteOnExit();
            return tempDir;
        } catch (IOException ex) {
            throw new IllegalStateException("Unable to create tempDir", ex);
        }
    }

}
{% endhighlight %}

What's interesting is that the constructor accept a variable list of `ServletContainerInitializer` instance. This interface allows a library/runtime to be notified of a web application's startup phase and perform any required programmatic registration of servlets, filters, and listeners in response to it. This is exactly what we need to register the entry point of the Spring MVC framework, the `DispatcherServlet`. Let's create another auto-configuration file to declare the Spring MVC servlet:


{% highlight java %}
package org.springframework.boot.autoconfigure.web;

import javax.servlet.ServletContainerInitializer;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRegistration.Dynamic;

import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.DispatcherServlet;

@Configuration
@ConditionalOnClass(DispatcherServlet.class)
public class DispatcherServletAutoConfiguration {

    @Bean(name = "dispatcherServlet")
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean(name = "dispatcherServletRegistration")
    public ServletContainerInitializer dispatcherServletRegistration() {
        return new ServletContainerInitializer() {
            @Override
            public void onStartup(Set<Class<?>> c, ServletContext ctx)
                    throws ServletException {
                Dynamic added = ctx.addServlet("dispatcherServlet", dispatcherServlet());
                added.addMapping("/*");
            }
        };
    }

}
{% endhighlight %}

We define two beans: the `DispatcherServlet`, and an anonymous class implementation of `ServletContainerInitializer`. This implementation retrieve the servlet and register it in the `ServletContext` object passed in parameter in the `onStartup` method. We configure the servlet to intercept all URLs starting from the root path. This class will be injected into the constructor of the previous `TomcatEmbeddedServletContainer` class.



So, we have an instance of Tomcat, we know how to register a servlet, but there is still one thing that is missing but required for any Spring Application: an application context.

The implementation to go when using Java Config is `AnnotationConfigWebApplicationContext`. But for Spring Boot, we need something more. We need to start a web server along the application context. To do this, Spring Boot decides to create a custom application context, `EmbeddedWebApplicationContext`, based on the superclass `GenericWebApplicationContext`, a subclass of `GenericApplicationContext`, suitable for web environments.


{% highlight java %}
package org.springframework.boot.context.embedded;

import java.util.*;
import javax.servlet.*;
import org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedServletContainer;
import org.springframework.context.*;
import org.springframework.web.context.*;
import org.springframework.web.context.support.*;

public class EmbeddedWebApplicationContext extends GenericWebApplicationContext
        implements ServletContainerInitializer {

    private final AnnotatedBeanDefinitionReader reader;

    private Class<?> annotatedClass;

    private volatile EmbeddedServletContainer embeddedServletContainer;

    public EmbeddedWebApplicationContext(Class<?> annotatedClass) {
        setEnvironment(new StandardServletEnvironment());
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.annotatedClass = annotatedClass;
        refresh();
        this.reader.register(this.annotatedClass);
    }

    @Override
    protected void onRefresh() {
        super.onRefresh();
        if (embeddedServletContainer == null) {
            embeddedServletContainer = getEmbeddedServletContainer();
        }
    }

    @Override
    protected void finishRefresh() {
        super.finishRefresh();
        if (embeddedServletContainer != null) {
            embeddedServletContainer.start();
        }
    }

    @Override
    protected void onClose() {
        super.onClose();
        if (embeddedServletContainer != null) {
            embeddedServletContainer.stop();
            embeddedServletContainer = null;
        }
    }

    private EmbeddedServletContainer getEmbeddedServletContainer() {
        List<ServletContainerInitializer> initializers = new ArrayList<>();
        initializers.add(this);
        initializers.addAll(getBeanFactory().
                getBeansOfType(ServletContainerInitializer.class).values());

        return new TomcatEmbeddedServletContainer(
                initializers.toArray(new ServletContainerInitializer[] {}));
    }

    /* ServletContainerInitializer implementation */

    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx)
            throws ServletException {
        ctx.setAttribute(ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE,
                         EmbeddedWebApplicationContext.this);
        setServletContext(ctx);
    }

}
{% endhighlight %}

Most of the code simply overrides superclass methods to start and stop the web server at the right time. The most interesting code happens in the method of creation of the `EmbeddedServletContainer`. The application context registers itself as an instance of `ServletContainerInitializer`. This is useful to gain access to the `ServletContext` and register itself as the root web application context. This attribute is used by the utility class `WebApplicationContextUtils`:

{% highlight java %}
public class WebApplicationContextUtils {
    ...
    public static WebApplicationContext getWebApplicationContext(ServletContext ctx) {
        return getWebApplicationContext(ctx,
            WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
    ...
}
{% endhighlight %}

In addition, we also register all others beans implementing the interface `EmbeddedServletContainer` present in the application context. If you remember, we declared only one bean of this type to register the `ServletDispatcher`.


The last thing to do is provide a facade to hide most of the Spring Boot internal code. This is the aim of the class `SpringApplication` that we used in our main method:

{% highlight java %}
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
{% endhighlight %}

Its implementation simply reuse the `EmbeddedWebApplicationContext`:


{% highlight java %}
package org.springframework.boot;

import org.springframework.context.ApplicationContext;
import org.springframework.context.ConfigurableApplicationContext;

public class SpringApplication {

    private final Object source;

    public SpringApplication(Object source) {
        this.source = source;
    }

    public ConfigurableApplicationContext run(String... args) {
        return new EmbeddedWebApplicationContext((Class<?>) source);
    }

    public static ConfigurableApplicationContext run(Object source, String... args) {
        return new SpringApplication(source).run(args);
    }

}
{% endhighlight %}

Cherry on the cake, let's add the Spring Banner, displayed in the console when the application starts:

{% highlight java %}
...
import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
...

public class SpringApplication {

    private static final Log logger = LogFactory.getLog(SpringApplication.class);

    ...

    public ConfigurableApplicationContext run(String... args) {
        logger.info(createStringFromBanner(new Banner()));
        return new EmbeddedWebApplicationContext((Class<?>) source);
    }

    private String createStringFromBanner(Banner banner) {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        banner.printBanner(new PrintStream(baos));
        return baos.toString();
    }
}
{% endhighlight %}

With the code for the `Banner` class:


{% highlight java %}
package org.springframework.boot;

import java.io.PrintStream;

class Banner {

    private static final String[] BANNER = { "",
            "  .   ____          _            __ _ _",
            " /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\",
            "( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\",
            " \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )",
            "  '  |____| .__|_| |_|_| |_\\__, | / / / /",
            " =========|_|==============|___/=/_/_/_/" };

    private static final String SPRING_BOOT = " :: Spring Boot :: ";

    private static final int STRAP_LINE_SIZE = 42;

    public void printBanner(PrintStream printStream) {
        for (String line : BANNER) {
            printStream.println(line);
        }
        String padding = "";
        while (padding.length() < STRAP_LINE_SIZE) {
            padding += " ";
        }

        printStream.println(SPRING_BOOT);
        printStream.println();
    }

}
{% endhighlight %}

Under Eclipse, we just have to Right Click + "Run as Java Application" to starts the application and see the banner appearing in the console. But we have much more to see about Spring Boot. To be able to run your application from a simple main class is really great when developing in your IDE but if we want to deploy your application outside of your local machine, we need to package it. Fortunately, Spring Boot supports many options when it comes to packaging...




## Part 3: Packaging your Spring Boot Application



### I want to package my web application as a single executable jar?

<div class="tip">
  <p class="title">What is an uber-jar?</p>
  <p>An uber-jar (or fat jar) is a jar containing the artifact to package, including its dependencies. The classic way to create an uber-jar is to use the Maven Shade Plugin. The plugin allows also to *shade* - i.e. rename - the packages of some of the dependencies.</p>
</div>



Here is an simple example (only two dependencies) using the Maven Shade Plugin :
`
{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>fr.imlovinit</groupId>
    <artifactId>maven-shade-plugin-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation=\
"org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>demo.Application</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <dependency>
            <groupId>commons-lang</groupId>
            <artifactId>commons-lang</artifactId>
            <version>2.6</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.7.3</version>
        </dependency>
    </dependencies>

</project>
{% endhighlight %}

This simple project contains only one class:

{% highlight java %}
package demo;

import java.util.*;
import java.util.Map;
import org.apache.commons.lang.StringUtils;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

public class Application {

    public static void main(String[] args) {
        try {
            System.out.println("I need Commons Lang & Jackson 2");
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
    }
}
{% endhighlight %}

When running the goal `package`, a new JAR is generated under the `target/` directory. Here is its directory hierarchy:

{% highlight console %}
maven-shade-plugin-demo-0.0.1-SNAPSHOT
 +- com
 |  \- fasterxml
 |     \- <jackson source>
 +- demo
 |  \- Application.class
 +- META-INF
 |  \- MANIFEST.MF <specify the main class to launch by default>
 \- org
    \- apache
       \- <commons lang source>
{% endhighlight %}

The Maven Shade Plugin expand all the dependency artefacts and repackage them together in a single archive. The META-INF files are merged together if necessary. The result looks like if we have coded all of these projects ourselves in our workspace.

Now, if we inspect a jar generated by Spring Boot, we observe Spring decide to use another approach.

{% highlight console %}
spring-boot-demo.jar
 +- hello
 |  +- Application.class
 |  \- HelloController.class
 +- lib
 |  +- spring-webmvc-4.2.5.RELEASE.jar
 |  +- tomcat-embed-core-8.0.32.jar
 |  \- ...
 +- META-INF
 |  \- MANIFEST.mf
 \- org/springframework/boot/loader
    |- JarLauncher.class
    \- ...
{% endhighlight %}

The application source code is separated from the libraries. We still have a manifest to specify the main class to launch but this class is not our `Application` class:

{% highlight properties %}
Manifest-Version: 1.0
Created-By: Apache Maven 3.3.9
Start-Class: hello.Application
Main-Class: org.springframework.boot.loader.JarLauncher
{% endhighlight %}

Note: the `Start-Class` is not a standard Manifest attribute but an extension of Spring Boot. We will use its value later in this section.


### How to create a well-organized JAR archive?

Unfortunately, there is no off-the-shelf solution. So, Spring Boot creates its own Maven Plugin (and a Gradle Plugin for non-Maven users). The plugin provides goals to package executable jar ou war archives or run the application "in-place":

{% highlight console %}
$ mvn help:describe -Dplugin=org.springframework.boot:spring-boot-maven-plugin

spring-boot:help
  Description: Display help information on spring-boot-maven-plugin.
    Call mvn spring-boot:help -Ddetail=true -Dgoal=<goal-name> to display
    parameter details.

spring-boot:repackage
  Description: Repackages existing JAR and WAR archives so that they can be
    executed from the command line using java -jar. With layout=NONE can also
    be used simply to package a JAR with nested dependencies (and no main
    class, so not executable).

spring-boot:run
  Description: Run an executable archive application.

spring-boot:start
  Description: Start a spring application. Contrary to the run goal, this
    does not block and allows other goal to operate on the application. This
    goal is typically used in integration test scenario where the application
    is started before a test suite and stopped after.

spring-boot:stop
  Description: Stop a spring application that has been started by the 'start'
    goal. Typically invoked once a test suite has completed.
{% endhighlight %}

The goal that interest us is the `repackage` goal. Let's begin by defining the structure of your mojo (Maven term to describe a plugin goal).


{% highlight java %}
package org.springframework.boot.maven;

import org.apache.maven.artifact.Artifact;
import org.apache.maven.plugin.*;
import org.apache.maven.plugins.annotations.*;
import org.apache.maven.project.MavenProject;
import org.springframework.boot.loader.tools.Library;
import org.springframework.boot.loader.tools.Repackager;

@Mojo(name = "repackage",
    defaultPhase = LifecyclePhase.PACKAGE,
    requiresDependencyResolution = ResolutionScope.COMPILE_PLUS_RUNTIME)
public class RepackageMojo extends AbstractMojo {

    @Parameter(defaultValue = "${project.build.directory}", required = true)
    private File outputDirectory;

    @Parameter(defaultValue = "${project.build.finalName}", required = true)
    private String finalName;

    @Parameter(defaultValue = "${project}", readonly = true, required = true)
    private MavenProject project;

    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {
        File source = this.project.getArtifact().getFile();
        File target = getTargetFile();
        Repackager repackager = new Repackager(source);

        try {
            repackager.repackage(target, getLibrairies());
        }
        catch (IOException ex) {
            throw new MojoExecutionException(ex.getMessage(), ex);
        }
    }

    private List<Library> getLibrairies() {
        List<Library> result = Lists.newArrayList();
        for (Artifact artifact : this.project.getArtifacts()) {
            String name = artifact.getFile().getName();
            result.add(new Library(name, artifact.getFile()));
        }
        return result;
    }

    private File getTargetFile() {
        if (!this.outputDirectory.exists()) {
            this.outputDirectory.mkdirs();
        }
        return new File(this.outputDirectory, this.finalName + "."
                + this.project.getArtifact().getArtifactHandler().getExtension());
    }

}
{% endhighlight %}

This mojo asks Maven to inject three dependencies:
- `outputDirectory`: The directory containing the generated archive.
- `finalName`: The name of the generated archive
- `project`: The instance of `MavenProject`. This object exposes a method to retrieve the list of dependencies of the project calculated by Maven (the same list as the effective pom).

The two first properties determine the archive's filepath to create. In practice, the value will often be the absolute filepath of the target folder and the nane of the <artifactId>-<version>.jar.
To be able to retrieve the list of dependencies through the `MavenProject` instance, we need to configure our Mojo using the attribute `requiresDependencyResolution` as follows:

{% highlight java %}
@Mojo(name = "repackage",
    defaultPhase = LifecyclePhase.PACKAGE,
    requiresDependencyResolution = ResolutionScope.COMPILE_PLUS_RUNTIME)
{% endhighlight %}

The `requiresDependencyResolution` flags this Mojo as requiring the dependencies in the specified class path to be resolved before it can execute. With this attribute, we can now use the `getArtifacts()` method on `MavenProject` to collect the list of dependencies to include in the lib folder. This is the role of the method `getLibrary()`. The `Library` class is a simple POJO defined by Spring Boot that contains a `File` and its name. Then, we create a `Repackager` by passing it the target file name and the dependencies list.

Before showing the code of this `Repackager` class, we should consider we have a wrapper `JarWriter` around the `java.util.jar` package. The aim of class present in the source of Spring Boot is to hide the low-level I/O manipulation code required to create a valid [Zip Archive](https://en.wikipedia.org/wiki/Zip_(file_format)). Here is the API of this class:


{% highlight java %}
package org.springframework.boot.loader.tools;

import java.io.*;
import java.net.*;
import java.util.*;
import java.util.jar.*;

/**
 * Writes JAR content, ensuring valid directory entries are always create.
 */
public class JarWriter {

    /**
     * Write the specified manifest.
     */
    void writeManifest(Manifest manifest) throws IOException;

    /**
     * Write all entries from the specified jar file.
     */
    void writeEntries(JarFile jarFile) throws IOException;

    /**
     * Writes an entry.
     * The {@code inputStream} is closed once the entry has been written
     */
    void writeEntry(String entryName, InputStream inputStream)
            throws IOException;

    /**
     * Write a nested library.
     */
    void writeNestedLibrary(String destination, Library library)
            throws IOException;

    /**
     * Close the writer.
     */
    void close() throws IOException;

}
{% endhighlight %}

If you are interested, have a look at the [source](https://github.com/spring-projects/spring-boot/blob/v1.3.3.RELEASE/spring-boot-tools/spring-boot-loader-tools/src/main/java/org/springframework/boot/loader/tools/JarWriter.java).

The missing piece is still the `Repackager class`, but with such an utility class, its implementation follows naturally. The repackager's constructor accept the standard JAR file created by Maven. By default, the jar produces by the repackager have the same file path, so we need to check first and rename the original file to be able to create the final jar archive. Once the file is created, we add the manifest by calling the method `writeManifest()` on `JarWriter`, then the original jar content with the method `writeEntries()` before iterating over the libraries returned by Maven and calling the method `writeNestedLibrary()` for each of them. Here is the implementation:

{% highlight java %}
package org.springframework.boot.loader.tools;

import java.io.*;
import java.util.*;
import java.util.jar.*;

public class Repackager {

    private final File source;

    public Repackager(File source) {
        this.source = source.getAbsoluteFile();
    }

    public void repackage(File destination, List<Library> libraries) throws IOException {
        destination = destination.getAbsoluteFile();
        File workingSource = this.source;
        if (this.source.equals(destination)) {
            workingSource = new File(this.source.getParentFile(),
                    this.source.getName() + ".original");
            workingSource.delete();
            this.source.renameTo(workingSource);
        }
        destination.delete();
        JarFile jarFileSource = new JarFile(workingSource);
        try {
            repackage(jarFileSource, destination, libraries);
        }
        finally {
            jarFileSource.close();
        }
    }

    private void repackage(JarFile sourceJar, File destination, List<Library> libraries)
            throws IOException {
        JarWriter writer = new JarWriter(destination);
        try {
            writer.writeManifest(buildManifest(sourceJar));
            writer.writeEntries(sourceJar);
            for (Library library : libraries) {
                writer.writeNestedLibrary("lib/", library);
            }
        }
        finally {
            try {
                writer.close();
            }
            catch (Exception ex) {
                // Ignore
            }
        }
    }

    private Manifest buildManifest(JarFile source) throws IOException {
        String mainClass = MainClassFinder.findSingleMainClass(source, "");
        Manifest manifest = new Manifest();
        manifest.getMainAttributes().putValue("Manifest-Version", "1.0");
        manifest.getMainAttributes().putValue("Main-Class", mainClass);
        return manifest;
    }

}
{% endhighlight %}

The building of the Manifest requires the name of the main class if we want to launch the jar archive without having to specify a fully-qualified name. To determine the main class, Spring Boot search in the original jar (contains only .class files) using ASM, the most popular Java bytecode manipulation library. Let's build a basic example to illustrate the use of ASM.

{% highlight java %}
package hello;

public class HelloWorld {

    private String message;

    public HelloWorld(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }

}
{% endhighlight %}

Before using ASM, we need to compile this class to generate the bytecode. ASM provides a class `asm.ClassReader` taking the compiled class as input. We could then register visitors (pattern [Visitor](https://en.wikipedia.org/wiki/Visitor_pattern)) that will be called during the class analysis. Visitors should extends the class `asm.ClassVisitor` and override the methods that interest them (ex: `visitMethod` to be notified on each method declaration found in the class). Here is an example using ASM to list the attributes and methods of a class:

{% highlight java %}
InputStream sourceCode = new FileInputStream("target/classes/hello/HelloWorld.class");
ClassReader reader = new ClassReader(sourceCode);
reader.accept(new ClassVisitor(Opcodes.ASM4) {

    @Override
    public FieldVisitor visitField(int access, String name,
                                   String desc, String signature, Object value) {
        System.out.println("Visit field " + name);
        return super.visitField(access, name, desc, signature, value);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name,
                                     String desc, String signature, String[] exceptions) {
        System.out.println("Visit method " + name);
        return super.visitMethod(access, name, desc, signature, exceptions);
    }

}, ClassReader.SKIP_CODE);
{% endhighlight %}

Here is the console output when running this code:

{% highlight console %}
Visit field message
Visit method <init>
Visit method getMessage
{% endhighlight %}

Now that we understand the working of the ASM library, let's see how Spring Boot does to find the main class. This is the role of the method `MainClassFinder.findSingleMainClass()`:

{% highlight java %}
package org.springframework.boot.loader.tools;

import java.io.*;
import java.util.*;
import java.util.jar.*;
import org.springframework.asm.ClassReader;
import org.springframework.asm.ClassVisitor;
import org.springframework.asm.MethodVisitor;
import org.springframework.asm.Opcodes;
import org.springframework.asm.Type;

public abstract class MainClassFinder {

    public static String findSingleMainClass(JarFile jarFile)
            throws IOException {
        for (JarEntry entry : getClassEntries(jarFile)) {
            InputStream inputStream = new BufferedInputStream(
                    jarFile.getInputStream(entry));
            try {
                if (isMainClass(inputStream)) {
                    return convertToClassName(entry.getName());
                }
            }
            finally {
                inputStream.close();
            }
        }
        return null;
    }

    private static List<JarEntry> getClassEntries(JarFile source) {
        Enumeration<JarEntry> sourceEntries = source.entries();
        List<JarEntry> classEntries = new ArrayList<JarEntry>();
        while (sourceEntries.hasMoreElements()) {
            JarEntry entry = sourceEntries.nextElement();
            if (entry.getName().endsWith(".class")) {
                classEntries.add(entry);
            }
        }
        return classEntries;
    }

    private static boolean isMainClass(InputStream inputStream) {
        try {
            ClassReader classReader = new ClassReader(inputStream);
            MainMethodFinder mainMethodFinder = new MainMethodFinder();
            classReader.accept(mainMethodFinder, ClassReader.SKIP_CODE);
            return mainMethodFinder.isFound();
        }
        catch (IOException ex) {
            return false;
        }
    }

    private static String convertToClassName(String name) {
        name = name.replace("/", ".");
        name = name.replace('\\', '.');
        name = name.substring(0, name.length() - ".class".length());
        return name;
    }

    private static class MainMethodFinder extends ClassVisitor {

        private boolean found;

        MainMethodFinder() {
            super(Opcodes.ASM4);
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String desc,
                String signature, String[] exceptions) {
            Type mainMethodType = Type.getMethodType(
                    Type.VOID_TYPE,
                    Type.getType(String[].class));
            if (isAccess(access, Opcodes.ACC_PUBLIC, Opcodes.ACC_STATIC)
                    && "main".equals(name)
                    && mainMethodType.getDescriptor().equals(desc)) {
                this.found = true;
            }
            return null;
        }

        private boolean isAccess(int access, int... requiredOpsCodes) {
            for (int requiredOpsCode : requiredOpsCodes) {
                if ((access & requiredOpsCode) == 0) {
                    return false;
                }
            }
            return true;
        }

        public boolean isFound() {
            return this.found;
        }

    }

}
{% endhighlight %}

We begin by searching all .class files and create an `InputStream` for each of them. Then, the method `isMainClass`. This method uses ASM like we did previously, registering its visitor represented by the inner class `MainMethodFinder`. The visitor need only to be notified when a new method is found, so it only overrides the method `visitMethod`. The remaining code is low-level ASM code to detect if it is the method main by checking the modifiers, the parameter types, and of course, the method name.


Our Maven plugin is now operational. We could update the pom of our project demo:

{% highlight java %}
<project>
    ...
    <build>
        <plugins>
            <plugin>
                <groupId>fr.imlovinit</groupId>
                <artifactId>spring-boot-lite-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    ...
</project>
{% endhighlight %}

And run the command:

{% highlight console %}
$ mvn clean spring-boot-demo:repackage
{% endhighlight %}

A new JAR is created inside the target directory named `spring-boot-demo-0.0.1-SNAPSHOT.jar`. But when launching the JAR, an error message is displayed in the console:

{% highlight java %}
$ java -jar spring-boot-demo-0.0.1-SNAPSHOT.jar
java.lang.NoClassDefFoundError: org/springframework/context/ApplicationContext
        at java.lang.Class.getDeclaredMethods0(Native Method)
        at java.lang.Class.privateGetDeclaredMethods(Unknown Source)
        at java.lang.Class.privateGetMethodRecursive(Unknown Source)
        at java.lang.Class.getMethod0(Unknown Source)
        at java.lang.Class.getMethod(Unknown Source)
        at sun.launcher.LauncherHelper.validateMainClass(Unknown Source)
        at sun.launcher.LauncherHelper.checkAndLoadMain(Unknown Source)
Caused by: java.lang.ClassNotFoundException: org.springframework.context.ApplicationContext
        at java.net.URLClassLoader.findClass(Unknown Source)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        at sun.misc.Launcher$AppClassLoader.loadClass(Unknown Source)
        at java.lang.ClassLoader.loadClass(Unknown Source)
        ... 7 more
Error: A JNI error has occurred, please check your installation and try again
Exception in thread "main"
{% endhighlight %}

We get an `NoClassDefFoundError` on `org.springframework.context.ApplicationContext`. Why? This class is present in the archive `spring-context-4.2.5.RELEASE.jar` under the lib folder. The problem is that the java command only load the files of type .class and do not search in the eventual jars included in the archive. This explains why the Maven Shade Plugin unzips the dependencies before creating the final archive. In this way, the java command find all the classes to load. So, how does Spring Boot do to load our dependencies? The answer is the module [Spring Boot Loader](https://github.com/spring-projects/spring-boot/tree/v1.3.3.RELEASE/spring-boot-tools/spring-boot-loader) and its class `JarLauncher` (The class that was marked as the Main-Class in the MANIFEST.mf).


### Spring Boot Loader

The goal of the Loader is to bootstrap the application with all the dependencies present in the classpath. To do so, the Loader needs to create a custom `ClassLoader` that will try to load classes from the lib/ folder and then delegate to the original parent ClassLoader if the class could not be found For example, a class like `java.lang.IllegalArgumentException` will not be found in the lib folder but the parent class loader could load any class of the JRE.

Let's decompose the class `JarLauncher`, probably the most complex piece of code of all this post.

{% highlight java %}
public class JarLauncher {


    public static void main(String[] args) {
        new JarLauncher().launch(args);
    }

    private final JarFileArchive archive;

    public JarLauncher() {
        try {
            this.archive = createArchive(); (1)
        }
        catch (Exception ex) {
            throw new IllegalStateException(ex);
        }
    }

    public void launch(String[] args) {
        try {
            ClassLoader classLoader = createClassLoader(); (2)
            startRunner(args, this.archive.getMainClass(), classLoader); (3)
        }
        catch (Exception ex) {
            ex.printStackTrace();
            System.exit(1);
        }
    }

    ...
}
{% endhighlight %}

The `JarLauncher` contains a `main` method that simply creates an instance of `JarLauncher` and call the method `launch`.
This class is divided in three sections:

- Read the content of the launched JAR to find the dependencies in lib/ folder
- Create the custom `ClassLoader` to check this folder first
- Run the main class using the previously created `ClassLoader`


#### The Archive

Spring Boot Loader create its own abstraction, extending {@link java.util.jar.JarFile} to offer additional functionalities like adding or retrieved a nested archive inside the JAR.

The implementation of the class `JarFileArchive` is pointless to understand Spring Boot. Here is its API:

{% highlight java %}
public class JarFileArchive {

    /** Search nested archived using a filter to exclude unwanted files. */
    List<JarFileArchive> getNestedArchives(EntryFilter filter) throws IOException;

    /** Return the content of the MANIFEST.MF file. */
    Manifest getManifest() throws IOException;

    /**
     * Return the value of the Start-Class attribute.
     * This attribute is Spring Boot specific and should not be confused
     * with the standard Main-Class attribute.
     */
    String getMainClass() throws Exception;
}
{% endhighlight %}

How to retrieve the path of the Jar passed to the java command?

TODO explain

{% highlight java %}
    private final JarFileArchive createArchive() throws Exception {
        return new JarFileArchive(
                getClass().
                getProtectionDomain().
                getCodeSource().
                getLocation().
                getPath());
    }
{% endhighlight %}


#### The ClassLoader

With the `JarFileArchive` object, we can now create a special `ClassLoader`. The class loader is represented by the class `LaunchedURLClassLoader`, probably the most complex piece of code presented in this post.

We will follow the code line by line:

{% highlight java %}
public class LaunchedURLClassLoader extends URLClassLoader {

    public LaunchedURLClassLoader(URL[] urls, ClassLoader parent) {
        super(urls, parent);
    }
{% endhighlight %}

Spring Boot extends the `java.net.URLClassLoader`, the standard implementation used to load classes from a search path of URLs referring to both JAR files and directory.  Any URL that ends with a '/' is assumed to refer to a directory. Otherwise, the URL is assumed to refer to a JAR file which will be opened as needed.

This class loader understand URLs like the following ones:

- `jar:file:./target/spring-boot-demo.jar`
Load the JAR located at this relative path

- `jar:file:./target/spring-boot-demo.jar!/lib/spring-webmvc-4.2.5.RELEASE.jar!/`
Load the root of the JAR `spring-webmvc-4.2.5.RELEASE.jar` included in the `spring-boot-demo.jar`.


The constructor expects two arguments:

- A list of URLs like the previous ones. Thanks to the `JarArchiveFile`, we could easily retrieved the URL of the main JAR and all of its dependencies.
- The parent class loader.


<div class="tip">
  <p class="title">How many class loaders?</p>
  <p>Even the most basic Java program has at least three class loaders:</p>
  <ul>
    <li>The <strong>bootstrap class loader</strong> loads the core Java libraries located in the $JAVA_HOME/jre/lib directory.</li>
    <li>The <strong>extensions class loader</strong> loads the code in the extensions directories ($JAVA_HOME/jre/lib/ext by default). It is implemented by the sun.misc.Launcher$ExtClassLoader class.</li>
    <li>The <strong>system/application class loader</strong> loads code found on java.class.path by default. It is implemented by the sun.misc.Launcher$AppClassLoader class.</li>
  </ul>
  <p>These class loaders follow a delegation hierarchy, where each one delegates to its parent before attempting to load a class itself. Our custom class loader will act similarly.</p>
</div>

Let's go back to the implementation of `LaunchedURLClassLoader`:

{% highlight java %}
public class LaunchedURLClassLoader extends URLClassLoader {

    ...

    @Override
    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            Class<?> loadedClass = findLoadedClass(name); (1)
            if (loadedClass == null) {
                loadedClass = doLoadClass(name);
            }
            if (resolve) {
                resolveClass(loadedClass); (2)
            }
            return loadedClass;
        }
    }

    private Class<?> doLoadClass(String name) throws ClassNotFoundException {
        // 1) Try to find locally
        try {
            findPackage(name);
            Class<?> cls = findClass(name);
            return cls;
        }
        catch (Exception ex) {
            // Ignore and continue
        }

        // 2) Use standard loading
        return super.loadClass(name, false);
    }

    ...

{% endhighlight %}

The code begins with a guard condition to detect if the class was already loaded previously. The `resolveClass` method is misleadingly named and is used by a class loader to link a class. All `ClassLoader` follow these two step. If the class was not loaded, we go into the method `doLoadClass`. This method attempts to load classes from the URLs before delegating to the parent loader. Let's continue:

{% highlight java %}
    ...

    private void findPackage(final String name) throws ClassNotFoundException {
        int lastDot = name.lastIndexOf('.');
        if (lastDot != -1) {
            String packageName = name.substring(0, lastDot);
            if (getPackage(packageName) == null) {
                try {
                    definePackageForFindClass(name, packageName);
                }
                catch (Exception ex) {
                    // Swallow and continue
                }
            }
        }
    }
{% endhighlight %}

We extract the package name from the given name to test if the package was already encountered (All classes in the same package share the same `java.lang.Package` object). If so, we could continue without loading it; the parent class loader will be able to find the target class. Otherwise, we have to register the new package before a `findClass` call is made. This is necessary to ensure that the appropriate manifest for nested JARs are associated with the package. This is implemented by the method `definePackageForFindClass`:

{% highlight java %}
    ...

    private void definePackageForFindClass(final String name, final String packageName) {
        try {
            AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                @Override
                public Object run() throws ClassNotFoundException {
                    String path = name.replace('.', '/').concat(".class");
                    for (URL url : getURLs()) {
                        try {
                            if (url.getContent() instanceof JarFile) {
                                JarFile jarFile = (JarFile) url.getContent();
                                if (jarFile.getJarEntryData(path) != null) {
                                    definePackage(packageName, jarFile.getManifest(),
                                            url);
                                    return null;
                                }

                            }
                        }
                        catch (IOException ex) {
                            // Ignore
                        }
                    }
                    return null;
                }
            }, AccessController.getContext());
        }
        catch (java.security.PrivilegedActionException ex) {
            // Ignore
        }
    }
{% endhighlight %}

We now have a valid class loader containing all your dependencies, ready to host our application code.


#### The Runner


The `Runner` is an utility class that used by the launcher to call your main method. The problem is our code could not run in the application class loader created by the JVM at startup because no dependency does not exist in this class loader. We have to start our code using the specific class loader `LauncherURLClassLoader`. This could be easily done by starting a new thread configured with the right class loader:

{% highlight java %}
package org.springframework.boot.loader;

import java.lang.reflect.Method;

public class MainMethodRunner implements Runnable {

    private final String mainClassName;

    private final String[] args;

    public MainMethodRunner(String mainClass, String[] args) {
        this.mainClassName = mainClass;
        this.args = args;
    }

    @Override
    public void run() {
        try {
            Class<?> mainClass = Thread.currentThread().getContextClassLoader()
                    .loadClass(this.mainClassName);
            Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
            mainMethod.invoke(null, new Object[] { this.args });
        }
        catch (Exception ex) {
            throw new RuntimeException(ex);
        }
    }

}
{% endhighlight %}

With the class loader and the runner, we can now completed the implementation of the class `JarLauncher`.

{% highlight java %}
public class JarLauncher {

    ...

    public void launch(String[] args) {
        try {
            JarFile.registerUrlProtocolHandler();
            ClassLoader classLoader = createClassLoader();
            startRunner(args, this.archive.getMainClass(), classLoader);
        }
        catch (Exception ex) {
            ex.printStackTrace();
            System.exit(1);
        }
    }

    // New code follows:

    private ClassLoader createClassLoader() throws Exception {
        Set<URL> urls = new LinkedHashSet<URL>();
        urls.addAll(getArtefactURL());
        urls.addAll(getLibURLs());
        return new LaunchedURLClassLoader(urls, getClass().getClassLoader());
    }

    private void startRunner(String[] args, String mainClass, ClassLoader classLoader)
            throws Exception {
        // Create a new thread to configure the created ClassLoader
        Runnable runner = createMainMethodRunner(mainClass, args, classLoader);
        Thread runnerThread = new Thread(runner);
        runnerThread.setContextClassLoader(classLoader);
        runnerThread.setName(Thread.currentThread().getName());
        runnerThread.start();
    }

    private Runnable createMainMethodRunner(String mainClass, String[] args,
            ClassLoader classLoader) throws Exception {
        return (Runnable) classLoader.
                loadClass("org.springframework.boot.loader.MainMethodRunner").
                getConstructor(String.class, String[].class).
                newInstance(mainClass, args);
    }

    ...

}
{% endhighlight %}

The `createClassLoader` retrieve the URLs (the URL of the JAR generated by Spring Boot and the URLs of each dependency) before instantiating the class loader. The code to retrieved the URLs simply uses the `JarFileArchive` API to traverse the nested archives and filter to keep all archives in the lib folder.

The method `startRunner` launch the application using the archive file's manifest (the Start-Class Attribute) to determine the main class, and using the fully configured class loader. We should ensure the runner is instantiated in the right class loader. This is the aim of the method `createMainMethodRunner` who use the reflection API to do the equivalent of `new MainMethodRunner(mainClass, args)`.


The implementation of the loader is now completed. If we run the Maven command on your demo application:

{% highlight console %}
$ mvn clean spring-boot-lite:repackage
{% endhighlight %}

The executable JAR is still generated but if we launch it again, the application starts correctly:

{% highlight console %}
$ java -jar spring-boot-demo-0.0.1-SNAPSHOT.jar
10:27:45.148 [main] INFO  o.s.boot.SpringApplication -
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::


Let's inspect the beans provided by Spring Boot:
application
beanNameHandlerMapping
defaultServletHandlerMapping
defaultViewResolver
dispatcherServlet
dispatcherServletRegistration
handlerExceptionResolver
helloController
httpRequestHandlerAdapter
mvcContentNegotiationManager
mvcConversionService
mvcPathMatcher
mvcResourceUrlProvider
mvcUriComponentsContributor
mvcUrlPathHelper
mvcValidator
mvcViewResolver
org.springframework..DispatcherServletAutoConfiguration
org.springframework..EmbeddedServletContainerAutoConfiguration
org.springframework..WebMvcAutoConfiguration
org.springframework..enhancedConfigurationProcessor
org.springframework..importAwareProcessor
org.springframework..internalAutowiredAnnotationProcessor
org.springframework..internalCommonAnnotationProcessor
org.springframework..internalConfigurationAnnotationProcessor
org.springframework..internalRequiredAnnotationProcessor
org.springframework..internalEventListenerFactory
org.springframework..internalEventListenerProcessor
requestMappingHandlerAdapter
requestMappingHandlerMapping
resourceHandlerMapping
simpleControllerHandlerAdapter
tomcatEmbeddedServletContainer
viewControllerHandlerMapping
viewResolver
{% endhighlight %}


We know why our compile compiles without having the declare all the dependencies, how Spring Boot Auto-configuration avoid us to declare boilerplace beans and how Spring Boot package our application as an executable jar archive.

We’ve come a long way, and have a (almost) usable implementation of Spring Boot. There is, of course, a lot more to cover than just this article. Why don't fork the project at your turn and see how the actuator works under the hood or how the command line tool does to create a full application with no need for a build file! 




<div class="remember">
  <p class="title">A retenir</p>
  <ul>
    <li>An uber-jar could be created using the Maven Shade Plugin. All jars are exploded and merged into a unique jar.</li>
    <li>Spring keep dependencies in its own folder lib/ at the root of the JAR hierarchy but need to code custom logic to create a  <code>ClassLoader</code> at application startup.</li>
    <li>Creating a new starter for Spring Boot is easy: one project containing only a pom.xml listing the required dependencies, and one project to contains the bean definitions inside a class annotated with <code>@AutoConfiguration</code>. (All in one single project is totally fine)</li>
    <li>Auto-configuration relies on a new feature of Spring Framework 4, the <code>@Conditional</code> annotation. Spring Boot extends this mechanism to add its own conditions to detect if a bean is already defined or if a class is present in the classpath.</li>
  </ul>
</div>


