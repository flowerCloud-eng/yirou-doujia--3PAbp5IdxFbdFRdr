[合集 \- Serilog文档翻译系列(5\)](https://github.com)[1\.Serilog文档翻译系列（一） \- 入门指南08\-28](https://github.com/hugogoos/p/18380656)[2\.Serilog文档翻译系列（二） \- 设置AspNetCore应用程序08\-29](https://github.com/hugogoos/p/18386366)[3\.Serilog文档翻译系列（三） \- 基础配置09\-01](https://github.com/hugogoos/p/18390965):[楚门加速器p](https://tianchuang88.com)[4\.Serilog文档翻译系列（四） \- 结构化数据09\-06](https://github.com/hugogoos/p/18398988)5\.Serilog文档翻译系列（五） \- 编写日志事件09\-24收起
![](https://img2024.cnblogs.com/blog/386841/202409/386841-20240924223722605-166713797.png)


日志事件通过 Log 静态类或 ILogger 接口上的方法写入接收器。下面的示例将使用 Log 以便语法简洁，但下面显示的方法同样可用于接口。


`Log.Warning("Disk quota {Quota} MB exceeded by {User}", quota, user);`


通过此日志方法创建的警告事件将具有两个相关属性，Quota 和 User。假设 quota 是一个整数，user 是一个字符串，呈现的消息可能如下所示。


`Disk quota 1024 MB exceeded by "nblumhardt"`


（Serilog 使用双引号渲染字符串值，以更清晰地指示其底层数据类型，并使属性值从周围的消息文本中脱颖而出。）


# 01、消息模板语法


上述字符串 “Disk quota {Quota} exceeded by {User}” 是一个 Serilog 消息模板。消息模板是标准 .NET 格式字符串的超集，因此任何适用于 string.Format() 的格式字符串也会被 Serilog 正确处理。


* 属性名称用 { 和 } 括起来书写。
* 属性名称必须是有效的 C\# 标识符，例如 FooBar，但不能是 Foo.Bar 或 Foo\-Bar。
* 括号可以通过重复书写来转义，例如 {{ 将被渲染为 {。
* 使用数字属性名称的格式，如 {0} 和 {1}，将通过将属性名称视为索引来与日志方法的参数匹配；这与 string.Format() 的行为相同。
* 如果任何属性名称是非数字，则所有属性名称将从左到右与日志方法的参数进行匹配。
* 属性名称可以带有可选的运算符前缀 @ 或 $，以控制属性的序列化方式。
* 属性名称可以带有可选的格式后缀，例如 :000，以控制属性的渲染方式；这些格式字符串的行为与 string.Format() 语法中的对应部分完全相同。


![](https://img2024.cnblogs.com/blog/386841/202409/386841-20240924223731314-1018963518.png)


# 02、消息模板推荐


流畅风格指南：好的 Serilog 事件使用属性名称作为消息内容，如上面的用户示例。这提高了可读性，并使事件更简洁。


句子与片段：日志事件消息是片段，而不是句子；为了与其他使用 Serilog 的库保持一致，尽量避免使用句末的句号。


模板与消息：Serilog 事件关联的是消息模板，而不是消息。内部，Serilog 会解析并缓存每个模板（直到固定大小限制）。将日志方法的字符串参数视为消息，如下例所示，会降低性能并消耗缓存内存。



```
//不推荐
Log.Information("The time is " + DateTime.Now);

```

相反，始终使用模板属性在消息中包含变量



```
// 推荐
Log.Information("The time is {Now}", DateTime.Now);

```

属性命名：属性名称应使用 PascalCase，以保持与 Serilog 生态系统中其他代码和库的一致性。


![](https://img2024.cnblogs.com/blog/386841/202409/386841-20240924223741505-1955188061.png)


# 03、日志事件级别


Serilog 使用级别作为分配日志事件重要性的主要手段。级别按重要性递增的顺序为：


* 详细信息（Verbose） \- 跟踪信息和调试细节；通常仅在特殊情况下开启。
* 调试（Debug） \- 内部控制流和诊断状态转储，以便于定位已知问题。
* 信息（Information） \- 对外部观察者有意义或相关的事件；默认启用的最低日志级别。
* 警告（Warning） \- 可能问题或服务/功能下降的指示。
* 错误（Error） \- 指示应用程序或连接系统中的失败。
* 致命（Fatal） \- 导致应用程序完全失败的严重错误。


## 1、信息级别的作用


信息级别与其他指定级别不同\-它没有特定的语义，许多方面上表达了其他级别的缺失。


由于 Serilog 允许对应用程序的事件流进行处理或分析，信息级别可以视为事件的同义词。也就是说，大多数有趣的应用程序事件数据应记录在此级别。


## 2、级别检测


在大多数情况下，应用程序应该在不检查当前日志级别的情况下记录事件。级别检查的开销非常小，而调用禁用的日志记录方法的开销也很低。


在少数性能敏感的情况下，推荐的级别检测模式是将级别检测的结果存储在一个字段中，例如：


`readonly bool _isDebug = Log.IsEnabled(LogEventLevel.Debug);`


可以在写入日志事件之前高效地检查 \_isDebug 字段：


`if (_isDebug) Log.Debug("Someone is stuck debugging...");`


## 3、动态级别


许多大型或分布式应用需要在相对限制的日志级别下运行，例如信息级（我更倾向于这种）或警告级，并仅在检测到问题时将日志级别提高到调试级或详细级，以便收集更多数据的开销是合理的。


如果应用需要动态切换日志级别，第一步是在配置日志记录器时创建一个 LoggingLevelSwitch 的实例：


`var levelSwitch = new LoggingLevelSwitch();`


该对象默认将当前最低级别设置为信息级，因此为了使日志记录更为严格，可以提前设置其最低级别：


`levelSwitch.MinimumLevel = LogEventLevel.Warning;`


在配置日志记录器时，使用 MinimumLevel.ControlledBy() 提供该开关：



```
var log = new LoggerConfiguration()
  .MinimumLevel.ControlledBy(levelSwitch)
  .WriteTo.ColoredConsole()
  .CreateLogger();

```

现在，写入日志记录器的事件将根据开关的 MinimumLevel 属性进行过滤。


要在运行时调整日志级别，例如响应通过网络发送的命令，可以更改该属性：



```
levelSwitch.MinimumLevel = LogEventLevel.Verbose;
log.Verbose("This will now be logged");

```

![](https://img2024.cnblogs.com/blog/386841/202409/386841-20240924223754135-1715061400.jpg)


# 04、源上下文


Serilog 和大多数 .NET 日志框架一样，允许事件带上其来源标签，通常是写入这些事件的类的名称：



```
var myLog = Log.ForContext<MyClass>();
myLog.Information("Hello!");

```

写入的事件将包含一个属性 SourceContext，其值为 "MyNamespace.MyClass"，可以用来过滤噪音事件或选择性地将其写入特定的接收器。


并非所有附加到事件的属性都需要在消息模板或输出格式中表示；所有属性都存储在底层 LogEvent 对象的字典中。


有关过滤器和日志记录拓扑的更多信息，请参阅《Serilog文档翻译系列（三） \- 基础配置》知识。


# 05、关联


正如 ForContext() 会将日志事件标记为写入它们的类，ForContext() 的其他重载允许日志事件用标识符进行标记，这些标识符随后可以支持与该标识符关联的事件的关联。



```
var job = GetNextJob();
var jobLog = Log.ForContext("JobId", job.Id);
jobLog.Information("Running a new job");
job.Run();
jobLog.Information("Finished");

```

在这里，两个日志事件都将携带 JobId 属性，其中包含作业标识符。


提示：当记录到使用文本格式的接收器时，例如 Serilog.Sinks.Console，可以在输出模板中包含 {Properties} 来打印出所有未包含的上下文属性。


注：相关源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Planner](https://github.com)


