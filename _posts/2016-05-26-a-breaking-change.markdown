---
published: true
title: A Breaking Change
layout: post
tags: [java]
---
We have a monolith. It's not the biggest monolith, but it does contain a lot of Java libraries, managed with Maven. This week an upgrade to one package caused a distant side effect.

Our monolith is a web application, and uses an embedded Jetty server. It also uses Jasper reports, compiling `jrxml` files on demand. This week, our builds stopped compiling reports, with an exception that Jasper could not find `javac`. The first thought was that somebody had changed the server install from a JDK to a JRE, leaving the compiler unavailable. But after installing the latest JDK, and checking the system path, the error changed:

```
net.sf.jasperreports.engine.JRException: Errors were encountered when compiling report expressions class file:
C:\Users\adrian\Workspaces\Customer\App\App-Web\j491845presale_seller_1464295746680_612367.java:227: error: cannot find symbol
                value = DATEFORMAT(((java.sql.Timestamp)field_sale_start.getValue()), "dd/MM/YYYY"); //$JR_EXPR_ID=2$
                        ^
  symbol:   method DATEFORMAT(Timestamp,String)
  location: class j491845presale_seller_1464295746680_612367
```

As `jasperreports-functions-5.6.1.jar` was in the classpath, there was clearly another problem.  Why was Jasper searching for `javac` at all, when Eclipse JDT was available?

{% highlight xml %}
 <dependency>
            <groupId>org.eclipse.jdt.core.compiler</groupId>
            <artifactId>ecj</artifactId>
            <version>4.4</version>
</dependency>
{%endhighlight %}

The next step was to go through our commits to find the point where things went wrong. It didn't take long to discover the breakage was caused by an upgrade from Jetty 9.0 (2013) to Jetty 9.3 (2016). But how had this broken Jasper?

Stepping through Jasper initialisation reveals that Jasper attempts to load `org.eclipse.jdt.internal.compiler.env.AccessRestriction` in order to discover which version of JDT is present. This attempt fails, even though the class is present on the classpath.

This turned my debugging attention to the classloader. Jetty includes a `WebAppClassloader`, and setting a breakpoint in its `loadClass` method revealed that  the JDT class was identified as a [server class](http://www.eclipse.org/jetty/documentation/current/jetty-classloading.html). Knowing this cause, it didn't take long to find the [commit that broke Jasper on Github](https://github.com/eclipse/jetty.project/commit/d0b70ee2591ad9de49d8710b187f91c0a1be2754).

As our application uses Jetty as an embedded web server, I disabled the `WebAppClassloader` like this, and Jasper reports worked again:

{% highlight java %}
WebAppContext wac = new WebAppContext();
// As we use Jasper Reports dynamic compiling, ensure Jetty uses the standard system classloader
// so that Jasper Reports can load ECJ
wac.setClassLoader(ClassLoader.getSystemClassLoader());
{% endhighlight %}

Originally published by [Adrian Cox](https://twitter.com/_a_d_r_1_a_n_) at 
[https://adrianathumboldt.github.io/](https://twitter.com/_a_d_r_1_a_n_).
