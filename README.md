# Log Aggregation Solution for Java Application running in Containers

Log aggregation, literally speaking, refers to "gathering log files and sending them into one place". However, it is especially important for helping developers and operation engineers to understand and debug their applications. Debugging an application without log messages which are rich enough can be such a headache that takes longer time to resolve than usual. Moreover, for enterprise line-of-business applications, logs aggregation is not only required by development team but also has become an ask from the security team. Being able to generate and egress logs in real-time has been a commmon needs from applications used in different industry. Besides that, being able to send out the log and directly send it to the storage for either hot or cold retention is also necessary. Hot-tier storage can be used for debugging and visualizing purpose, used by developers and operation engineers. Meanwhile, cold-tier storage is usualy set up for longer rentention purpose to fulfill the needs like regulation from security prospective. 

With that being said, desiging a log aggregation solution is essential to a modern application. This sometimes becomes a "after-thought" in application development process since the development team would first concentrate on solving the core business user cases. In addition, log aggregating has become a common need hence various different existing solutions has been propsed already. You might want to avoid the scenarios of re-inventing the wheels. For example, if your application is hosted on Azure there will be activity logs and diagnostic logs (if enabled) collected. You can choose a log ingesting tool such as Azure Event Hub to dump the logs to a storage account for longer retention, or integerate with other SIEM tool such as QRadar or Splunk. Howeber, if you want to generate customized logs which contains the information you need, you might want to use other logging services. In this article, an example Java application that utilized Apache Log4j as its logging tool and two different solutions both consist of log egress and ingress process will be demonstrated. 

## Solution 1: Apache Log4j + Appender to Azure Application Insights + Azure Storage Account
### Set up Application Insights resource in Azure platform
Application Insights is a Application Performance Management (APM) service for web developers on multiple platforms. It can be used to monitor your live web application, detect performance anomalies, visulize issues and usabilities. 

How to provision a new Application Insights resource: https://docs.microsoft.com/en-us/azure/azure-monitor/app/website-monitoring 

### Configure and run Application in containers using Tomcat
Log4j is a fail-stop logging tool which helps the developer to output log statements a variety of output targets. Apache Log4j 2 is an upgrade to Log4j that provides significant improvements over its predecessor, Log4j 1.x, and provides many of the improvements available in Logback while fixing some inherent problems in Logback’s architecture (https://logging.apache.org/). To showcase the demo, we build an example java application which coontains Log4j integeration to generate audit logs. 

The source code of this project can be found here: https://github.com/xuhongl/Log-Egress-AI 

In this solution, we use log4j application insights appender to collect trace logs and send them automatically to the Azure Application Insights resource where the logs can be displayed and explored. 

Use the Application Insights Java agent in AI-Agent.xml file:
```
<ApplicationInsightsAgent>
   <Instrumentation>
      <BuiltIn enabled="true">
         <Logging enabled="true" />
      </BuiltIn>
   </Instrumentation>
   <AgentLogger />
</ApplicationInsightsAgent>
```

Add logging libraries to the project to pom.xml:
```
    	<dependency>
			<groupId>com.microsoft.azure</groupId>
			<artifactId>applicationinsights-web-auto</artifactId>
			<version>2.5.0</version>
		</dependency>
       <dependency>
          <groupId>com.microsoft.azure</groupId>
          <artifactId>applicationinsights-logging-log4j1_2</artifactId>
          <version>[2.0,)</version>
       </dependency>
	   	<dependency>
			<groupId>com.microsoft.azure</groupId>
			<artifactId>applicationinsights-spring-boot-starter</artifactId>
			<version>1.1.1</version>
		</dependency>
```

Add the appender to log4f.properties
```
# Global logging configuration
log4j.rootLogger=INFO, stdout, R

log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout

# Pattern to output the caller's file name and line number.
log4j.appender.stdout.layout.ConversionPattern=%5p [%t] (%F:%L) - %m%n

log4j.appender.R=com.microsoft.applicationinsights.log4j.v1_2.ApplicationInsightsAppender
log4j.appender.R.instrumentationKey=ed8a0337-d85a-46d6-9448-4c9f67165808
```

Build and run the application
```
mvn clean package
mvn tomcat7:run
```

### Export log data from Application Insights to Storage Account Blob Storage
Set up log ingress to storage account by going to selected Application Insights resources -> "Configuration" -> "Continuous Export"


## Solution 2: Log4j + Appender to Azure Event Hub + Azure Storage Account + Azure Function

### Provision Event Hub and Storage Account in Azure platform
Create an event hub namespace and an event hub in Azure portal: https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-create

After creating an event hub namespace and during the process of creating a new event hub, enable "capture" to set up log exporting from event hub to the specified storage account for longer retention. Select "Don't emit empty files when no events occur during the Capture time window" to pervent empty log files with metadata from being reserved. 



