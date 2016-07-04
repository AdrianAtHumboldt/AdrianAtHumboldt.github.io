---
published: true
title: Spring 4 and Quartz 2 Integration with Custom Annotations
layout: post
tags: [java]
---
I'm integrating Quartz scheduling into an application, and was looking for an annotation based approach to configuration.  I quickly found [the SivaLabs implementation](http://sivalabs.in/2011/10/spring-and-quartz-integration-using-custom-annotation/), which works for Quartz 1.8. I've made a few changes to use this with Spring 4.1.4 and Quartz 2.2.3.

The custom annotation remains the same:

{% highlight java %}
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
@Scope("prototype")
public @interface QuartzJob {
    String name();
    String group() default "DEFAULT_GROUP";
    String cronExp();
}
{% endhighlight %}

The major change is that I have replaced the `ApplicationListener` class with a bean that performs a single scan of a selected package. I chose this approach because my configuration does not change once loaded, and I wanted to throw a fatal exception on any error in Quartz configuration. I've refactored to add each job to the scheduler as it is discovered, rather than building an intermediate list. 

{% highlight java %}
public class QuartzJobScanner
{
    @Autowired
    private Scheduler scheduler;

    private static final Logger log = LoggerFactory.getLogger(QuartzJobScanner.class);

    private final String scanPackage;

    public QuartzJobScanner(String scanPackage) {
         this.scanPackage = scanPackage;
    }

    @PostConstruct
    public void scheduleJobs() throws Exception
    {
        ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(false);
        // Filter to include only classes that have a particular annotation.
        provider.addIncludeFilter(new AnnotationTypeFilter(QuartzJob.class));
        // Find classes in the given package (or subpackages)
        Set<BeanDefinition> beans = provider.findCandidateComponents(scanPackage);

        for (BeanDefinition bd: beans)
            scheduleJob(bd);
    }
{% endhighlight %}

The `CronTriggerBean` and `JobDetailBean` from Spring 3 are gone in Spring 4.  The new code builds the job using the Quartz 2 fluent builders, rather than Spring's factory beans:

{% highlight java %}
    private void scheduleJob(BeanDefinition bd) throws Exception
    {
        Class<?> beanClass = Class.forName(bd.getBeanClassName());
        QuartzJob quartzJob = beanClass.getAnnotation(QuartzJob.class);

        // Sanity check
        if(Job.class.isAssignableFrom(beanClass) && quartzJob != null)
        {
            @SuppressWarnings("unchecked") Class<? extends Job> jobClass = (Class<? extends Job>)(beanClass);

            log.info("Scheduling quartz job: " + quartzJob.name());
            JobDetail job = JobBuilder.newJob(jobClass)
                    .withIdentity(quartzJob.name(), quartzJob.group())
                    .build();
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withSchedule(CronScheduleBuilder.cronSchedule(quartzJob.cronExp()))
                    .withIdentity(quartzJob.name() + "_trigger", quartzJob.group())
                    .forJob(job)
                    .build();

            scheduler.scheduleJob(job, trigger);
        }
    }
}
{% endhighlight %}

The job factory is unchanged from the original article, and the XML configuration looks like this:

{% highlight xml %}
<context:component-scan base-package="uk.co.humboldt.Application.Services" />
<bean class="uk.co.humboldt.Application.Services.QuartzJobScanner">
        <constructor-arg value="uk.co.humboldt.Application.Services"/>
</bean>
<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
        <property name="jobFactory">
            <bean class="uk.co.humboldt.Application.Services.QuartzJobFactory"/>
        </property>
</bean>
{% endhighlight %}

At this point jobs are automatically configured as long they:

* implement the `Job` interface
* have the `QuartzJob` annotation
* are in a package scanned by Spring, and by the `QuartzJobScanner`

[Full code here.](https://gist.github.com/AdrianAtHumboldt/0885a39be05ceea8501b6661bbe8335d)
