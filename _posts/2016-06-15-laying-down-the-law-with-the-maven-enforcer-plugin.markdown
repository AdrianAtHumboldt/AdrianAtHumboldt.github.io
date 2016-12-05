---
published: true
title: Laying down the law with the Maven Enforcer Plugin
layout: post
tags: [java]
---
We recently moved an application to a new server, with an updated installation of Java 8. It didn't take long to discover that Amazon S3 accesses were failing with an authentication error, and it didn't take much longer to discover that this problem was [already fixed in the version of the AWS Java SDK](https://github.com/aws/aws-sdk-java/issues/484) we were using.

```
com.amazonaws.services.s3.model.AmazonS3Exception: AWS authentication requires a valid Date or x-amz-date header
```

The real problem was that we had many dependencies on different versions of Joda Time, and AWS had lost the battle - we were actually running Joda Time 2.0. 

It was time to stop this happening again, with the [Maven Enforcer Plugin](http://maven.apache.org/enforcer/maven-enforcer-plugin/). It's possible to fail the build if we ever reference two versions of the same dependency:
<!--more-->
{% highlight xml%}
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>1.4.1</version>
    <executions>
        <execution>
            <id>enforce-same-version</id>
            <configuration>
                <rules>
                    <dependencyConvergence/>
                </rules>
            </configuration>
            <goals>
                <goal>enforce</goal>
            </goals>
        </execution>
    </executions>
</plugin>
{% endhighlight %}

Of course, this reveals many other problems. There are a lot of libraries which will fail enforcement all by themselves. For example, Jasper Reports includes two versions of Apache Commons Beanutils:

```
+-uk.co.humboldt:OurApplication:0.0.1-SNAPSHOT
  +-net.sf.jasperreports:jasperreports:6.2.0
    +-commons-beanutils:commons-beanutils:1.9.0
and
+-uk.co.humboldt:OurApplication:0.0.1-SNAPSHOT
  +-net.sf.jasperreports:jasperreports:6.2.0
    +-commons-digester:commons-digester:2.1
      +-commons-beanutils:commons-beanutils:1.8.3
```

In fact, Jasper Reports was by far the worst offender in our code base, due to the sheer number of other libraries pulled in. Jasper also disagreed with our version of Jackson and our version of Spring framework. I also killed Commons Logging, [as the application uses SLF4J](http://www.slf4j.org/legacy.html).  The tamed version of the Jasper dependency now looks like this:

{% highlight xml %}
        <dependency>
            <groupId>net.sf.jasperreports</groupId>
            <artifactId>jasperreports</artifactId>
            <version>6.2.0</version>
            <exclusions>
                <exclusion>
                    <artifactId>jackson-core</artifactId>
                    <groupId>com.fasterxml.jackson.core</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>jackson-databind</artifactId>
                    <groupId>com.fasterxml.jackson.core</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>jackson-annotations</artifactId>
                    <groupId>com.fasterxml.jackson.core</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>commons-collections</artifactId>
                    <groupId>commons-collections</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>spring-context</artifactId>
                    <groupId>org.springframework</groupId>
                </exclusion>
                <exclusion>
                    <groupId>eclipse</groupId>
                    <artifactId>jdtcore</artifactId>
                </exclusion>
                <exclusion>
                    <artifactId>olap4j</artifactId>
                    <groupId>org.olap4j</groupId>
                </exclusion>
                <exclusion>
                    <groupId>com.lowagie</groupId>
                    <artifactId>itext</artifactId>
                </exclusion>
                <exclusion>
                    <artifactId>commons-logging</artifactId>
                    <groupId>commons-logging</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>commons-beanutils</artifactId>
                    <groupId>commons-beanutils</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>commons-beanutils</groupId>
            <artifactId>commons-beanutils</artifactId>
            <version>1.9.2</version>
            <exclusions>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>net.sf.jasperreports</groupId>
            <artifactId>jasperreports-functions</artifactId>
            <version>6.2.0</version>
            <exclusions>
                <exclusion>
                    <groupId>joda-time</groupId>
                    <artifactId>joda-time</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- Import Standard itext - the Jaspersoft fixes are for transparent charts, and don't seem to affect us. -->
        <dependency>
            <groupId>com.lowagie</groupId>
            <artifactId>itext</artifactId>
            <version>2.1.7</version>
        </dependency>
{% endhighlight %}

Originally published by [Adrian Cox](https://twitter.com/_a_d_r_1_a_n_) at 
[https://adrianathumboldt.github.io/](https://twitter.com/_a_d_r_1_a_n_).
