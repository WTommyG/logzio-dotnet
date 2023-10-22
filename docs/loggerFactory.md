# Logz.io loggerFactory Appender

- [Configuration](#configuration)
    - [XML](#xml)
    - [Code](#code)
- [Custom Fields](#custom-fields)
- [Extensibility](#extensibility)
- [Trace Context](#trace-context)
- [Code Sample](#code-sample)
    - [ASP.NET Core](#aspnet-core)
    - [.NET Core Desktop Application](#net-core-desktop-application)


Install the log4net appender from the Package Manager Console:

    Install-Package Logzio.DotNet.Log4net

Install the log4net as Microsoft Extensions Logging handler from the Package Manager Console:

    Install-Package Microsoft.Extensions.Logging.Log4Net.AspNetCore

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
        <!-- Enable the appender's internal debug logger (sent to the console output and trace log) -->
        <debug>false</debug>
        <!-- If internal debug logger is enabled, write debug logs to file. Absolute path to file,
        a GUID will be added to the file name. If no file was given or file path does not exist,
        creates debug.txt file in the directory where the application is running from. -->
        <debugLogFile>my_absolute_path_to_file</debugLogFile>
        <!-- Set to true if you want json keys in Logz.io to be in camel case. The default is false. -->
        <jsonKeysCamelCase>false</jsonKeysCamelCase>
        <!-- Add trace context (traceId and spanId) to each log. The default is false -->
        <addTraceContext>false</addTraceContext>
        <!-- Use the same static HTTP/s client for sending logs. The default is false -->
        <useStaticHttpClient>false</useStaticHttpClient>        
    </appender>
    
    <root>
    	<level value="INFO" />
    	<appender-ref ref="LogzioAppender" />
    </root>
</log4net>
```
### Code
To add the Logz.io appender via code, add the following lines:

```C#
var hierarchy = (Hierarchy)LogManager.GetRepository();
var logzioAppender = new LogzioAppender();
logzioAppender.AddToken("<<LOG-SHIPPING-TOKEN>>");
logzioAppender.AddListenerUrl("<<LISTENER-HOST>>");
// Uncomment and edit this line to enable proxy routing: 
// logzioAppender.AddProxyAddress("http://your.proxy.com:port");
// Uncomment these lines to enable gzip compression 
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

## Code Sample

### ASP.NET Core

Update Startup.cs file in Configure method to include the Log4Net middleware as below.

```C#
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILoggerFactory loggerFactory)
{
    ...
        
    loggerFactory.AddLog4Net();
        
    ...
}    
```

In the Controller add Data Member and Constructor as below.

```C#
private readonly ILoggerFactory _loggerFactory;
    
public ExampleController(ILoggerFactory loggerFactory, ...)
{
    _loggerFactory = loggerFactory;
            
    ...
}
```

In the Controller methods:

```C#
[Route("<PUT_HERE_YOUR_ROUTE>")]
public ActionResult ExampleMethod()
{
    var logger = _loggerFactory.CreateLogger<ExampleController>();
    var logRepository = LogManager.GetRepository(Assembly.GetEntryAssembly());
            
    // Replace "App.config" with the config file that holds your log4net configuration
    XmlConfigurator.Configure(logRepository, new FileInfo("log4net.config"));
            
    logger.LogInformation("Hello");
    logger.LogInformation("Is it me you looking for?");
            
    LogManager.Shutdown();
            
    return Ok();
}
```

### .NET Core Desktop Application

```C#
using System.IO;
using System.Reflection;
using log4net;
using log4net.Config;
using Microsoft.Extensions.Logging;

namespace LoggerFactoryAppender
{
    class Program
    {
        static void Main(string[] args)
        {
            ILoggerFactory loggerFactory = new LoggerFactory();
	        loggerFactory.AddLog4Net();

            var logger = loggerFactory.CreateLogger<Program>();
	        var logRepository = LogManager.GetRepository(Assembly.GetEntryAssembly());

            // Replace "App.config" with the config file that holds your log4net configuration
            XmlConfigurator.Configure(logRepository, new FileInfo("log4net.config"));

	        logger.LogInformation("Hello");
            logger.LogInformation("Is it me you looking for?");
                
            LogManager.Shutdown();
        }
    }
}
```

### Serverless platforms
If you’re using a serverless function, you’ll need to call the appender's flush method at the end of the function run to make sure the logs are sent before the function finishes its execution. You’ll also need to create a static appender in the Startup.cs file so each invocation will use the same appender. The appender should have the `UseStaticHttpClient` flag set to `true`.


###### Azure serverless function code sample

*Startup.cs*

```csharp
using Microsoft.Azure.Functions.Extensions.DependencyInjection;
using Logzio.DotNet.NLog;
using NLog;
using NLog.Config;
using System;

[assembly: FunctionsStartup(typeof(LogzioNLogSampleApplication.Startup))]

namespace LogzioNLogSampleApplication
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            var config = new LoggingConfiguration();

            // Replace these parameters with your configuration
            var logzioTarget = new LogzioTarget
            {
                Name = "Logzio",
                Token = "<<LOG-SHIPPING-TOKEN>>",
                LogzioType = "nlog",
                ListenerUrl = "https://<<LISTENER-HOST>>:8071",
                BufferSize = 100,
                BufferTimeout = TimeSpan.Parse("00:00:05"),
                RetriesMaxAttempts = 3,
                RetriesInterval = TimeSpan.Parse("00:00:02"),
                Debug = false,
                JsonKeysCamelCase = false,
                AddTraceContext = false,
                UseStaticHttpClient = true,
                // ParseJsonMessage = true,
                // ProxyAddress = "http://your.proxy.com:port"
            };

            config.AddRule(NLog.LogLevel.Debug, NLog.LogLevel.Fatal, logzioTarget);
            LogManager.Configuration = config;
        }
    }
}
```

*FunctionApp.cs*

```csharp
using System;
using Microsoft.Azure.WebJobs;
using NLog;
using Microsoft.Extensions.Logging;
using MicrosoftLogger = Microsoft.Extensions.Logging.ILogger;

namespace LogzioNLogSampleApplication
{
    public class TimerTriggerCSharpNLog
    {
        private static readonly Logger nLog = LogManager.GetCurrentClassLogger();

        [FunctionName("TimerTriggerCSharpNLog")]
        public void Run([TimerTrigger("*/30 * * * * *")]TimerInfo myTimer, MicrosoftLogger msLog)
        {
            msLog.LogInformation($"NLogzio C# Timer trigger function executed at: {DateTime.Now}");

            nLog.WithProperty("iCanBe", "your long lost pal")
                .WithProperty("iCanCallYou", "Betty, and Betty when you call me")
                .WithProperty("youCanCallMe", "Al")
                .Info("If you'll be my bodyguard");
            // Call Flush method before function trigger finishes
            LogManager.Flush(5000);
        }
    }
}
```