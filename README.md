# Log Aggregation Solution for Java Application running in Containers

Log aggregation, literally speaking, refers to "gathering log files and sending them into one place". However, it is especially important for helping developers and operation engineers to understand and debug their applications. Debugging an application without log messages which are rich enough can be such a headache that takes a longer time to resolve than usual. Moreover, for enterprise line-of-business applications, logs aggregation is not only required by the development team but also has become an ask from the security team. Being able to generate and egress logs in real-time has been a common need from applications used in different industries. Besides that, being able to send out the log and directly send it to the storage for either hot or cold retention is also necessary. Hot-tier storage can be used for debugging and visualizing purpose, used by developers and operation engineers. Meanwhile, cold-tier storage is usually set up for longer retention purpose to fulfill the needs like regulation from a security perspective. 

With that being said, designing a log aggregation solution is essential to a modern application. This sometimes becomes an "after-thought" in the application development process since the development team would first concentrate on solving the core business use cases. In addition, log aggregating has become a common need hence various different existing solutions have been proposed already. You might want to avoid the scenarios of re-inventing the wheels. For example, if your application is hosted on Azure there will be activity logs and diagnostic logs (if enabled) collected. You can choose a log ingesting tool such as Azure Event Hub to dump the logs to a storage account for longer retention, or integrate with other SIEM tools such as QRadar or Splunk. However, if you want to generate customized logs that contain the information you need, you might want to use other logging services. In this article, an example Java application that utilized Apache Log4j as its logging tool and two different solutions both consist of log egress and ingress process will be demonstrated. 

## Solution 1: Apache Log4j + Appender to Azure Application Insights + Azure Storage Account
### Set up Application Insights resource in Azure platform
Application Insights is an Application Performance Management (APM) service for web developers on multiple platforms. It can be used to monitor your live web application, detect performance anomalies, visualize issues and usabilities. 

How to provision a new Application Insights resource: https://docs.microsoft.com/en-us/azure/azure-monitor/app/website-monitoring 

### Configure and run Application in containers using Tomcat
Log4j is a fail-stop logging tool that helps the developer to output log statements a variety of output targets. Apache Log4j 2 is an upgrade to Log4j that provides significant improvements over its predecessor, Log4j 1.x, and provides many of the improvements available in Logback while fixing some inherent problems in Logback’s architecture (https://logging.apache.org/). To showcase the demo, we build an example java application which ontains Log4j integration to generate audit logs. 

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

After creating an event hub namespace and during the process of creating a new event hub, enable "capture" to set up log exporting from event hub to the specified storage account for longer retention. Choose the time window and size window based on your use case. Select "Don't emit empty files when no events occur during the Capture time window" to prevent empty log files with metadata from being reserved.


### Integrate the application with log4j Event Hub Appender
download and compile the Azure Event Hub log4j Appender: https://github.com/rahoogan/log4j-eventhub-appender. To install the 3rd party .jar package into your project, using maven-install-plugin with the following method:

```
mvn install:install-file -Dfile=<path-to-file> -DpomFile=<path-to-pomfile>
```

Then add the dependency to pom.xml
```
<dependency>
<groupId>com.rahoogan.log4j</groupId>
<artifactId>log4j-eventhub-appender</artifactId>
<version>1.0-SNAPSHOT</version>
</dependency>
```
Rebuild the application and check whether the result target folder has that dependency .jar

Once succeed, run the application using the command shown above, open http://localhost:9090/ and view different categories on the website to generate sample log data. Once the data is generated, it will be automatically sent out to event hub, and then reached the storage account blob storage based on your configuration. 

sample source code: https://github.com/xuhongl/Log-Egress-EH 


### Use Azure Function to Convert the Format and Match the Entries
Azure Event Hubs enables you to automatically capture the streaming data in Event Hubs in an Azure Blob storage or Azure Data Lake Storage account of your choice. Captured data is written in Apache Avro format: a compact, fast, binary format that provides rich data structures with inline schema. This format is widely used in the Hadoop ecosystem, Stream Analytics, and Azure Data Factory. You can view these files in any tool such as Azure Storage Explorer. You can also download the files locally to work on them.

To convert the format from Avro to Json, and to match the entries for specific user cases, a Azure Function is used to implement this step. Azure Functions is a serverless compute service that lets you run event-triggered code without having to explicitly provision or manage infrastructure. It is a perfect fit for this use case since the function can be configured to be triggered only when there is a new log file saved into the blob storage. 

To create an Azure Function with blob storage trigger, one of the feasible ways is creating a new function project in VS code with existing templates. Make sure Azure extension has been installed in your VS code. First open command palette by Ctrl+Shift+P, choose "Azure Function: create a new function", select the folder that will contain your function project. Then select the language for your function project (here we choose C#). After that select a template for your project's first function, in this case you can choose either Blob Trigger or Event Hub trigger (here we choose Blob Trigger). Then provide a function name and namespace, create a new local app setting, select the storage account where the connection will be built. The next step is to specify the path inside that storage account which the trigger will monitor. Finally select the storage account that will be used for debugging and create the function project. 

The function project consists of four files: YourFunctionName.csproj, YourTriggerName.cs, host.json and local.settings.json. In which host.json specify the version of your function app. local.settings.json includes the webjob connection, storage account connection string and function runtime. To implement the converter, first go to YourFunctionName.csproj and include the dependencies as following:
```
<PackageReference Include="Apache.Avro" Version="1.7.7.2" />
<PackageReference Include="Microsoft.Hadoop.Avro-Core" Version="1.1.19" />
```

Then change the YourFunctionName.cs according to the github repo provided above. 


sample source code: https://github.com/xuhongl/Format-Converter-Blob 
