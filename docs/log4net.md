# Logz.io log4net Appender

- [Configuration](#configuration)
	- [XML](#xml)
	- [Code](#code)
- [Custom Fields](#custom-fields)
- [Extensibility](#extensibility)
- [Trace Context](#trace-context)
- [Code Samples](#code-samples)


Install the log4net appender from the Package Manager Console:

    Install-Package Logzio.DotNet.Log4net

If you prefer to install the library manually, download the latest version from the releases page.

## Configuration
### XML
If you configure your logging in an XML file, simply add a reference to the Logz.io appender.

```xml
<log4net>
    <appender name="LogzioAppender" type="Logzio.DotNet.Log4net.LogzioAppender, Logzio.DotNet.Log4net">
    	<!-- 
		Required fields 
	-->
	<!-- Your Logz.io API token -->
	<token><<LOG-SHIPPING-TOKEN>></token>
			
	<!-- 
		Optional fields (with their default values) 
	-->
	<!-- The type field will be added to each log message, making it 
	easier for you to differ between different types of logs. -->
    	<type>log4net</type>
	<!-- The URL of the Lgz.io listener -->
    	<listenerUrl>https://<<LISTENER-HOST>>:8071</listenerUrl>
        <!--Optional proxy server address:
        proxyAddress = "http://your.proxy.com:port" -->
	<!-- The maximum number of log lines to send in each bulk -->
    	<bufferSize>100</bufferSize>
	<!-- The maximum time to wait for more log lines, in a hh:mm:ss.fff format -->
    	<bufferTimeout>00:00:05</bufferTimeout>
	<!-- If connection to Logz.io API fails, how many times to retry -->
    	<retriesMaxAttempts>3</retriesMaxAttempts>
    	<!-- Time to wait between retries, in a hh:mm:ss.fff format -->
	<retriesInterval>00:00:02</retriesInterval>
	<!-- Set the appender to compress the message before sending it -->
	<gzip>true</gzip>
	<!-- Uncomment this to enable sending logs in Json format -->
	<!--<parseJsonMessage>true</parseJsonMessage>-->
	<!-- Enable the appender's internal debug logger (sent to the console output and trace log) -->
	<debug>false</debug>
	<!-- If internal debug logger is enabled, write debug logs to file. Absolute path to file,
	a GUID will be added to the file name. If no file was given or file path does not exist,
	creates debug.txt file in the directory where the application is running from. -->
	<debugLogFile>my_absolute_path_to_file</debugLogFile>		
	<!-- If you have custom fields keys that start with capital letter and want to see the fields 
	with capital letter in Logz.io, set this field to true. The default is false 
	(first letter will be small letter). -->
	<jsonKeysCamelCase>false</jsonKeysCamelCase>
	<!-- Add trace context (traceId and spanId) to each log. The default is false -->
	<addTraceContext>false</addTraceContext>
    <!-- Use the same static HTTP/s client for sending logs. The default is false -->
	<UseStaticHttpClient>false</addTraceContext>

    </appender>
    
    <root>
    	<level value="INFO" />
    	<appender-ref ref="LogzioAppender" />
    </root>
</log4net>
```
Add a reference to the configuration file in your code, as shown in the example [here](https://github.com/logzio/logzio-dotnet/blob/master/sample-applications/LogzioLog4netSampleApplication/Program.cs).
### Code
To add the Logz.io appender via code, add the following lines:

```C#
var hierarchy = (Hierarchy)LogManager.GetRepository();
var logzioAppender = new LogzioAppender();
logzioAppender.AddToken("<<LOG-SHIPPING-TOKEN>>");
logzioAppender.AddListenerUrl("<<LISTENER-HOST>>");
// <-- Uncomment and edit this line to enable proxy routing: --> 
// logzioAppender.AddProxyAddress("http://your.proxy.com:port");
// <-- Uncomment this to enable sending logs in Json format -->  
// logzioAppender.ParseJsonMessage(true);
// <-- Uncomment these lines to enable gzip compression --> 
// logzioAppender.AddGzip(true);
// logzioAppender.ActivateOptions();
// logzioAppender.JsonKeysCamelCase(false);
// logzioAppender.AddTraceContext(false);
// logzioAppender.AddDebug(false);
// logzioAppender.AddDebugLogFile("my_absolute_path_to_file");
// logzioAppender.UseStaticHttpClient(false);
logzioAppender.ActivateOptions();
hierarchy.Root.AddAppender(logzioAppender);
hierarchy.Root.Level = Level.All;
hierarchy.Configured = true;
```

## Custom Fields

You can add static keys and values to be added to all log messages. For example:

```XML
<appender name="LogzioAppender" type="Logzio.DotNet.Log4net.LogzioAppender, Logzio.DotNet.Log4net">
    <customField>
	<key>Environment</key>
	<value>Production</value>
    </customField>
    <customField>
	<key>Location</key>
	<value>New Jerseay B1</value>
    </customField>
</appender>
```

## Extensibility 

If you want to change some of the fields or add some of your own, inherit the appender and override the `ExtendValues` method:

```C#
public class MyAppLogzioAppender : LogzioAppender
{
    protected override void ExtendValues(LoggingEvent loggingEvent, Dictionary<string, string> values)
    {
	values["logger"] = "MyPrefix." + values["logger"];
	values["myAppClientId"] = new ClientIdProvider().Get();
    }
}
```

You will then have to change your configuration in order to use your own appender.

## Trace Context

**WARNING**: Does not support .NET Standard 1.3

If you’re sending traces with OpenTelemetry instrumentation (auto or manual), you can correlate your logs with the trace context.
In this way, your logs will have traces data in it: span id and trace id.
To enable this feature, set `<addTraceContext>true</addTraceContext>` in your configuration or `logzioAppender.AddTraceContext(true);`
in your code (as shown in the previews sections).

## Code Samples

### XML

```C#
using System.IO;
using log4net;
using log4net.Config;
using System.Reflection;

namespace dotnet_log4net
{
    class Program
    {
        static void Main(string[] args)
        {
            var logger = LogManager.GetLogger(typeof(Program));
            var logRepository = LogManager.GetRepository(Assembly.GetEntryAssembly());

            // Replace "App.config" with the config file that holds your log4net configuration
            XmlConfigurator.Configure(logRepository, new FileInfo("App.config"));

            logger.Info("Now I don't blame him 'cause he run and hid");
            logger.Info("But the meanest thing he ever did");
            logger.Info("Before he left was he went and named me Sue");

            LogManager.Shutdown();
        }
    }
}
```

### Code

```C#
using log4net;
using log4net.Core;
using log4net.Repository.Hierarchy;
using Logzio.DotNet.Log4net;

namespace dotnet_log4net
{
    class Program
    {
        static void Main(string[] args)
        {
            var hierarchy = (Hierarchy)LogManager.GetRepository();
            var logger = LogManager.GetLogger(typeof(Program));
            var logzioAppender = new LogzioAppender();
            
            logzioAppender.AddToken("<<LOG-SHIPPING-TOKEN>>");
            logzioAppender.AddListenerUrl("https://<<LISTENER-HOST>>:8071");
            // <-- Uncomment and edit this line to enable proxy routing: --> 
            // logzioAppender.AddProxyAddress("http://your.proxy.com:port");
            // <-- Uncomment this to enable sending logs in Json format -->  
            // logzioAppender.ParseJsonMessage(true);
            // <-- Uncomment these lines to enable gzip compression --> 
            // logzioAppender.AddGzip(true);
            // logzioAppender.ActivateOptions();
            // logzioAppender.JsonKeysCamelCase(false)
            // logzioAppender.AddTraceContext(false);
            // logzioAppender.AddDebug(false);
            // logzioAppender.AddDebugLogFile("my_absolute_path_to_file");
            // logzioAppender.UseStaticHttpClient(false);
            logzioAppender.ActivateOptions();
            
            hierarchy.Root.AddAppender(logzioAppender);
            hierarchy.Configured = true;
            hierarchy.Root.Level = Level.All;

            logger.Info("Now I don't blame him 'cause he run and hid");
            logger.Info("But the meanest thing he ever did");
            logger.Info("Before he left was he went and named me Sue");

            LogManager.Shutdown();
        }
    }
}
```


## Serverless platforms
If you’re using a serverless function, you’ll need to call the appender's flush method at the end of the function run to make sure the logs are sent before the function finishes its execution. You’ll also need to create a static appender in the Startup.cs file so each invocation will use the same appender. The appender should have the `UseStaticHttpClient` flag set to `true`.

###### Azure serverless function code sample
*Startup.cs*
```csharp
using Microsoft.Azure.Functions.Extensions.DependencyInjection;
using log4net;
using log4net.Repository.Hierarchy;
using Logzio.DotNet.Log4net;

[assembly: FunctionsStartup(typeof(LogzioLog4NetSampleApplication.Startup))]

namespace LogzioLog4NetSampleApplication
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            var hierarchy = (Hierarchy)LogManager.GetRepository();
            var logzioAppender = new LogzioAppender();
            logzioAppender.AddToken("<<LOG-SHIPPING-TOKEN>>");
            logzioAppender.AddListenerUrl("https://<<LISTENER-HOST>>:8071");
            logzioAppender.ActivateOptions();
            logzioAppender.UseStaticHttpClient(true);
            hierarchy.Root.AddAppender(logzioAppender);
            hierarchy.Configured = true;
        }
    }
}

```

*FunctionApp.cs*
```csharp
using System;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using log4net;
using MicrosoftLogger = Microsoft.Extensions.Logging.ILogger;

namespace LogzioLog4NetSampleApplication
{
    public class TimerTriggerCSharpLog4Net
    {
        
        private static readonly ILog logger = LogManager.GetLogger(typeof(TimerTriggerCSharpLog4Net));

        [FunctionName("TimerTriggerCSharpLog4Net")]
        public void Run([TimerTrigger("*/30 * * * * *")]TimerInfo myTimer, MicrosoftLogger msLog)
        {
            msLog.LogInformation($"Log4Net C# Timer trigger function executed at: {DateTime.Now}");

            logger.Info("Now I don't blame him 'cause he run and hid");
            logger.Info("But the meanest thing he ever did");
            logger.Info("Before he left was he went and named me Sue");
            LogManager.Flush(5000);

            msLog.LogInformation($"Log4Net C# Timer trigger function finished at: {DateTime.Now}");
        }
    }
}
```