**English** | [简体中文](README.zh-CN.md)

# serilog-sinks-mariadb [![Build status](https://ci.appveyor.com/api/projects/status/x19dx2d21h6aow9x/branch/master?svg=true)](https://ci.appveyor.com/project/Mantas/serilog-sinks-mariadb/branch/master) ![Nuget](https://img.shields.io/nuget/v/Serilog.Sinks.MariaDB.svg) ![Nuget](https://img.shields.io/nuget/dt/Serilog.Sinks.MariaDB.svg)

A Serilog sink that writes events to **MariaDB/MySQL**. This sink will write the log event to a table. Important properties can also be written to their own separate columns. Properties by default are written to Text column and are formatted as JSON (custom formatter can be provided for them). This sink was hugelly inspired by [MSSqlServer sink](https://github.com/serilog/serilog-sinks-mssqlserver).

This sink inherits from [`PeriodicBatching` Sink](https://github.com/serilog/serilog-sinks-periodicbatching) - events can be inserted as bulk for performance gains or one by one for reliability.

**Nuget Package:** [Serilog.Sinks.MariaDB](https://www.nuget.org/packages/Serilog.Sinks.MariaDB/)

### Configuration Samples

#### Code-Only

All sink features are configurable from code.

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

#### Configuration File

They also can be configured through _Microsoft.Extensions.Configuration_ sources, including .NET Core's `appsettings.json` file. We assume, that you've installed [serilog-settings-configuration](https://github.com/serilog/serilog-settings-configuration) package.

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

## Configuration

### Sink Configuration Options

Sink configuration accept the following arguments, though not necessarily in this order. Use of named arguments is strongly recommended.

* `connectionString`
* `tableName`
* `autoCreateTable`
* `useBulkInsert`
* `options`
* `period`
* `batchPostingLimit`
* `formatProvider`

### Audit Sink Configuration Options

Audit sink configuration accept the following arguments, though not necessarily in this order. Use of named arguments is strongly recommended.

* `connectionString`
* `tableName`
* `autoCreateTable`
* `options`
* `formatProvider`

### Basic Arguments

At minimum, `connectionString` is required.

If `tableName` is omitted, it defaults to `Logs`.

When `autoCreateTable` is `true` (default is `false`), the sink will create a SQL table if it doesn't yet exist. The table will have all the columns provided in `MariaDBSinkOptions.PropertiesToColumnsMapping`. All of them will be created as `TEXT` columns, it's users responsibility to change them to the desired data type.

When `useBulkInsert` is `true` (default is `true`), the batch will be inserted as single bulk insert operation (better performance), otherwise it will insert log events one by one (better reliability). We noticed around 50x performance difference while benchmarking with 5000 bached events. If you choose to use bulk inserts - be carefull regarding [max_allowed_packet](https://mariadb.com/kb/en/library/server-system-variables/#max_allowed_packet), which determines size of maximum single SQL statement, that is sent to the server. Do your own benchmarking to determine what fits for you.

This is a "periodic batching sink." The sink will queue a certain number of log events before they're actually written to MariaDB/MySQL as a bulk insert operation. There is also a timeout period so that the batch is always written even if it has not been filled. By default, the batch size is 50 rows and the timeout is 5 seconds. You can change these through by setting the `batchPostingLimit` and `period` arguments.

Refer to the Serilog Wiki's explanation of [Format Providers](https://github.com/serilog/serilog/wiki/Formatting-Output#format-providers) for details about the `formatProvider` arguments.

## MariaDBSinkOptions Object

Features of the log table and how we persist data are defined by changing properties on a `MariaDBSinkOptions` object:

* `PropertiesToColumnsMapping`
* `PropertiesFormatter`
* `HashMessageTemplate`
* `TimestampInUtc`
* `ExcludePropertiesWithDedicatedColumn`
* `EnumsAsInts`
* `LogRecordsExpiration`
* `LogRecordsCleanupFrequency`

### PropertiesToColumnsMapping

`PropertiesToColumnsMapping` is a dictionary defining event property name to SQL column mapping. The `key` is event property name, the value is SQL column name. 
Default value for `PropertiesToColumnsMapping` is shown below.

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

By editing this dictionary you can rename built-in columns. 
For example:

```csharp
//Remove built-in Exception mapping, because we have our custom custom "ExceptionDetails" property
propertiesToColumns = 
{
    ["Exception"] = null,
    ["ExceptionDetails"] = "Exception"
};

//Save Timestamp in custom column
propertiesToColumns = 
{
    ["Timestamp"] = "Created"
};

//Dedicated column URL for event property RequestUrl
propertiesToColumns = 
{
    ["RequestUrl"] = "URL"
};
```

### PropertiesFormatter

This is a `Func<IReadOnlyDictionary<string, LogEventPropertyValue>, string>` delegate which allows you to modify how data for `Properties` column is formatted. By default we format `Properties` as `JSON`, but if you want to override it - you can using this object property.

### HashMessageTemplate

When `HashMessageTemplate` is `true` (default is `false`), the message template is hashed, using the SHA256 algorithm. This may be more convenient when you want to search logs by message template.

### TimestampInUtc

When `TimestampInUtc` is `true` (default is `true`), the timestamp is converted to UTC before saving. Otherwise, it is saved in local time of the machine that issued the log event. For example, if the event is written at 07:00 Eastern time, the Eastern timezone is +4:00 relative to UTC, so after UTC conversion the time stamp will be 11:00.

### ExcludePropertiesWithDedicatedColumn

When `ExcludePropertiesWithDedicatedColumn` is `true` (default is `false`), custom properties that have their dedicated columns are not included in `Properties` column.

### EnumsAsInts

When `EnumsAsInts` is `true` (default is `false`), enums are converted to their coresponding integer values beforse saving, otherwise enums are stored as strings.

### LogRecordsExpiration

When `LogRecordsExpiration` TimeStamp is set (not set by default), sink tries to periodically delete rows older than set interval. Row age is determined by configured `Timestamp` column.

### DeleteChunkSize

DELETE in MariaDB/MySQL [is an expensive task](https://mariadb.com/kb/en/big-deletes/), so events are deleted by performing multiple DELETE queries with `LIMIT x` clause. Default is 2000 rows.

### LogRecordsCleanupFrequency

`LogRecordsCleanupFrequency` TimeStamp controls how often DELETE SQL command is called on expired rows (default is 12 minutes, only applicable if `LogRecordsExpiration` is set).


## Table Definition

If you don't use automatic creation of the table, you'll have to create a log event table in your database manually. If you use automatic table creation - you'll have to adjust column types, as auto-creation makes all columns as `TEXT`. The table definition shown below reflects the default configuration using automatic table creation without changing any sink options.

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

But you probably want to change it to:

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

**IMPORTANT:** Make sure your log table has all columns which are defined in `PropertiesToColumnsMapping` or inserts will fail.

## Troubleshooting

### Always check SelfLog first

After configuration is complete, this sink runs through a number of checks to ensure consistency. Some configuration issues result in an exception (at startup), but others may only generate warnings through Serilog's `SelfLog` feature. At runtime, exceptions are silently reported through `SelfLog`. Refer to [Debugging and Diagnostics](https://github.com/serilog/serilog/wiki/Debugging-and-Diagnostics#selflog) in the main Serilog documentation to enable `SelfLog` output.

### Always call Log.CloseAndFlush

Any Serilog application should _always_ call `Log.CloseAndFlush` before shutting down. This is especially important in sinks like this one. It is a "periodic batching sink" which means log event records are written in batches for performance reasons. Calling `Log.CloseAndFlush` should guarantee any batch in memory will be written to the database (but read the Visual Studio note below). You may wish to put the `Log.CloseAndFlush` call in a `finally` block in console-driven apps where a `Main` loop controls the overall startup and shutdown process. Refer to the _Serilog.AspNetCore_ sample code for an example. More exotic scenarios like dependency injection may warrant hooking the `ProcessExit` event when the logger is registered as a singleton:

```csharp
AppDomain.CurrentDomain.ProcessExit += (s, e) => Log.CloseAndFlush();
```

### Test outside of Visual Studio

When you exit an application running in debug mode under Visual Studio, normal shutdown processes may be interrupted. Visual Studio issues a nearly-instant process kill command when it decides you're done debugging. This is a particularly common problem with ASP.NET and ASP.NET Core applications, in which Visual Studio instantly terminates the application as soon as the browser is closed. Even `finally` blocks usually fail to execute. If you aren't seeing your last few events written, try testing your application outside of Visual Studio.
