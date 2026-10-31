[English](README.md) | **简体中文**

# serilog-sinks-mariadb [![Build status](https://ci.appveyor.com/api/projects/status/x19dx2d21h6aow9x/branch/master?svg=true)](https://ci.appveyor.com/project/Mantas/serilog-sinks-mariadb/branch/master) ![Nuget](https://img.shields.io/nuget/v/Serilog.Sinks.MariaDB.svg) ![Nuget](https://img.shields.io/nuget/dt/Serilog.Sinks.MariaDB.svg)

一个将事件写入 **MariaDB/MySQL** 的 Serilog 接收器。日志事件写入数据表，重要属性也可写入独立列。默认情况下，属性以 JSON 格式写入 Text 列，也可提供自定义格式化器。本项目深受 [MSSqlServer 接收器](https://github.com/serilog/serilog-sinks-mssqlserver)启发。

该接收器继承自 [`PeriodicBatching` 接收器](https://github.com/serilog/serilog-sinks-periodicbatching)：事件既可批量插入以提高性能，也可逐条插入以提高可靠性。

**NuGet 包：** [Serilog.Sinks.MariaDB](https://www.nuget.org/packages/Serilog.Sinks.MariaDB/)

### 配置示例

#### 仅使用代码

所有接收器功能都可以通过代码配置。

```csharp
var log = new LoggerConfiguration()
    .WriteTo.MariaDB(
        connectionString: @"server=...",
        tableName: "Logs",
        autoCreateTable: true,
        useBulkInsert: false,
        options: new MariaDBSinkOptions()
        )
    .CreateLogger();
```

```csharp
var log = new LoggerConfiguration()
    .AuditTo.MariaDB(
        connectionString: @"server=...",
        tableName: "Logs",
        autoCreateTable: true,
        options: new MariaDBSinkOptions()
        )
    .CreateLogger();
```

#### 配置文件

也可通过 _Microsoft.Extensions.Configuration_ 配置源进行配置，包括 .NET Core 的 `appsettings.json`。以下示例假设已安装 [serilog-settings-configuration](https://github.com/serilog/serilog-settings-configuration) 包。

```json
{
    "Serilog": {
        "Using": [
            "Serilog.Sinks.MariaDB"
        ],
        "MinimumLevel": "Debug",
        "WriteTo": [
            {
                "Name": "MariaDB",
                "Args": {
                    "connectionString": "server=...",
                    "autoCreateTable": false,
                    "tableName": "Logs",
                    "restrictedToMinimumLevel": "Warning",
                    "batchPostingLimit": 1000,
                    "period": "0.00:00:30",
                    "options": {
                        "PropertiesToColumnsMapping": {
                            "Exception": "Exception",
                            "Level": "Level",
                            "Message": "Message",
                            "MessageTemplate": "MessageTemplate",
                            "Properties": "Properties",
                            "Timestamp": "Timestamp"
                        },
                        "TimestampInUtc": true,
                        "ExcludePropertiesWithDedicatedColumn": true,
                        "EnumsAsInts": true,
                        "LogRecordsCleanupFrequency": "0.02:00:00",
                        "LogRecordsExpiration": "31.00:00:00",
                    }
                }
            }
        ]
    }
}
```

```json
{
    "Serilog": {
        "Using": [
            "Serilog.Sinks.MariaDB"
        ],
        "MinimumLevel": "Debug",
        "AuditTo": [
            {
                "Name": "MariaDB",
                "Args": {
                    "connectionString": "server=...",
                    "autoCreateTable": false,
                    "tableName": "Logs"
                }
            }
        ]
    }
}
```

## 配置

### 接收器配置选项

接收器配置接受以下参数，顺序不一定相同。强烈建议使用命名参数。

* `connectionString`
* `tableName`
* `autoCreateTable`
* `useBulkInsert`
* `options`
* `period`
* `batchPostingLimit`
* `formatProvider`

### 审计接收器配置选项

审计接收器配置接受以下参数，顺序不一定相同。强烈建议使用命名参数。

* `connectionString`
* `tableName`
* `autoCreateTable`
* `options`
* `formatProvider`

### 基本参数

至少需要提供 `connectionString`。

如果省略 `tableName`，默认值为 `Logs`。

当 `autoCreateTable` 为 `true`（默认是 `false`）时，接收器会在表不存在时创建 SQL 表。表包含 `MariaDBSinkOptions.PropertiesToColumnsMapping` 中定义的所有列。这些列都会创建为 `TEXT` 类型，用户需要自行调整为所需的数据类型。

当 `useBulkInsert` 为 `true`（默认值）时，批次通过一次批量插入写入，性能更好；否则逐条插入，可靠性更高。原项目在 5,000 条事件的基准测试中观察到约 50 倍性能差异。使用批量插入时，请注意限制单条 SQL 语句大小的 [max_allowed_packet](https://mariadb.com/kb/en/library/server-system-variables/#max_allowed_packet)，并根据自己的场景进行基准测试。

这是一个“定期批处理接收器”：事件先进入队列，再批量写入 MariaDB/MySQL。超时后，即使批次未满也会写入。默认批量大小为 50 行，超时时间为 5 秒，可通过 `batchPostingLimit` 和 `period` 调整。

有关 `formatProvider` 参数，请参阅 Serilog Wiki 对[格式提供程序](https://github.com/serilog/serilog/wiki/Formatting-Output#format-providers)的说明。

## MariaDBSinkOptions 对象

日志表的功能和数据持久化方式由 `MariaDBSinkOptions` 对象的以下属性定义：

* `PropertiesToColumnsMapping`
* `PropertiesFormatter`
* `HashMessageTemplate`
* `TimestampInUtc`
* `ExcludePropertiesWithDedicatedColumn`
* `EnumsAsInts`
* `LogRecordsExpiration`
* `LogRecordsCleanupFrequency`

### PropertiesToColumnsMapping

`PropertiesToColumnsMapping` 是事件属性名到 SQL 列名的映射字典：`key` 是事件属性名，值是 SQL 列名。
`PropertiesToColumnsMapping` 的默认值如下。

```csharp
var propertiesToColumns = new Dictionary<string, string>(StringComparer.InvariantCultureIgnoreCase)
        {
            ["Exception"] = "Exception",
            ["Level"] = "LogLevel",
            ["Message"] = "Message",
            ["MessageTemplate"] = "MessageTemplate",
            ["Properties"] = "Properties",
            ["Timestamp"] = "Timestamp",
        };
```

编辑该字典可以重命名内置列。 
例如：

```csharp
// 移除内置 Exception 映射，改用自定义的 ExceptionDetails 属性
propertiesToColumns = 
{
    ["Exception"] = null,
    ["ExceptionDetails"] = "Exception"
};

// 将 Timestamp 保存到自定义列
propertiesToColumns = 
{
    ["Timestamp"] = "Created"
};

// 为事件属性 RequestUrl 使用专用的 URL 列
propertiesToColumns = 
{
    ["RequestUrl"] = "URL"
};
```

### PropertiesFormatter

这是一个 `Func<IReadOnlyDictionary<string, LogEventPropertyValue>, string>` 委托，可自定义 `Properties` 列的数据格式。默认格式为 `JSON`，可通过此属性覆盖。

### HashMessageTemplate

当 `HashMessageTemplate` 为 `true`（默认是 `false`）时，消息模板会使用 SHA256 算法计算哈希，便于按消息模板搜索日志。

### TimestampInUtc

当 `TimestampInUtc` 为 `true`（默认值）时，保存前会将时间戳转换为 UTC；否则保存产生日志事件的机器的本地时间。例如，美国东部时间 07:00 在 UTC 比当地时间快 4 小时的情况下会转换为 11:00。

### ExcludePropertiesWithDedicatedColumn

当 `ExcludePropertiesWithDedicatedColumn` 为 `true`（默认是 `false`）时，已有专用列的自定义属性不会包含在 `Properties` 列中。

### EnumsAsInts

当 `EnumsAsInts` 为 `true`（默认是 `false`）时，枚举在保存前转换为对应整数；否则以字符串保存。

### LogRecordsExpiration

设置 `LogRecordsExpiration` 时间间隔后（默认未设置），接收器会定期尝试删除超过该间隔的记录。记录年龄由配置的 `Timestamp` 列确定。

### DeleteChunkSize

MariaDB/MySQL 的 DELETE 操作[开销较高](https://mariadb.com/kb/en/big-deletes/)，因此通过多次带 `LIMIT x` 子句的 DELETE 查询删除事件。默认每次删除 2,000 行。

### LogRecordsCleanupFrequency

`LogRecordsCleanupFrequency` 时间间隔控制对过期记录执行 DELETE 的频率（默认 12 分钟，仅在设置 `LogRecordsExpiration` 时生效）。


## 表定义

如果不使用自动建表，需要手动在数据库中创建日志事件表；如果使用自动建表，则需要调整列类型，因为所有列都会创建为 `TEXT`。以下表定义对应未更改接收器选项时的默认自动建表结果。

```sql
CREATE TABLE IF NOT EXISTS `Logs` (
    `Id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `Timestamp` TEXT NULL,
    `Level` TEXT NULL,
    `Message` TEXT NULL,
    `MessageTemplate` TEXT NULL,
    `Exception` TEXT NULL,
    `Properties` TEXT NULL
)
```

但你可能希望将其改为：

```sql
CREATE TABLE IF NOT EXISTS `Logs` (
    `Id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `Timestamp` DATETIME DEFAULT NULL,
    `Level` VARCHAR(15) DEFAULT NULL,
    `Message` TEXT NULL,
    `MessageTemplate` TEXT NULL,
    `Exception` TEXT NULL,
    `Properties` TEXT NULL
)
```

**重要：**确保日志表包含 `PropertiesToColumnsMapping` 定义的所有列，否则插入会失败。

## 故障排查

### 首先检查 SelfLog

配置完成后，接收器会执行一系列一致性检查。有些配置问题会在启动时抛出异常，另一些只会通过 Serilog 的 `SelfLog` 产生警告。运行时异常也会通过 `SelfLog` 静默报告。请参阅 Serilog 文档的[调试与诊断](https://github.com/serilog/serilog/wiki/Debugging-and-Diagnostics#selflog)，了解如何启用 `SelfLog` 输出。

### 务必调用 Log.CloseAndFlush

所有 Serilog 应用都应在关闭前调用 `Log.CloseAndFlush`。这对定期批处理接收器尤其重要：日志会为了性能分批写入，调用该方法可确保内存中的批次写入数据库（另请参阅下方 Visual Studio 注意事项）。对于由 `Main` 循环控制启动和关闭的控制台应用，可将调用放入 `finally` 块；示例参见 _Serilog.AspNetCore_。如果通过依赖注入将日志器注册为单例，也可考虑处理 `ProcessExit` 事件：

```csharp
AppDomain.CurrentDomain.ProcessExit += (s, e) => Log.CloseAndFlush();
```

### 在 Visual Studio 之外测试

退出在 Visual Studio 调试模式下运行的应用时，正常关闭流程可能被中断。Visual Studio 结束调试时可能立即终止进程；对于 ASP.NET 和 ASP.NET Core 应用，关闭浏览器时尤其常见，甚至 `finally` 块通常也无法执行。如果最后几条事件没有写入，请尝试在 Visual Studio 之外测试。
