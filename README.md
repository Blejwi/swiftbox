# SwiftBox
SwiftBox is a package that helps to build Swift/Vapor microservices.

## SwiftBox Configuration
SwiftBox Configuration allows to pass type-safe configuration from command line, environment variables and other sources by declaring one simple struct.

Configuration can be automatically populated with:
- Command line arguments: `./yourapp --config:simple=test --config:nested.number=1 --config:array.0=string`
- Environment variables: `SIMPLE=test NESTED_NUMBER=1 ARRAY_0=string ./yourapp`
- JSON
- Dictionary

SwiftBox Configuration supports:
- overriding (source that is declared later can override previous values)
- inheritance
- optionals
- type-safety
- nesting (structs and arrays)

### Usage
#### 1. Import
Import module:
```swift
import SwiftBoxConfig
```

#### 2. Configuration structure
When you create your configuration, remember that it in order to be decoded properly, it must conform to the `Configuration` protocol.
```swift
struct Conf: Configuration {
    let simple: String
    let int: Int
    let double: Double
    let nested: NestedConf // nested configuration
    let array: [String?] // array of optionals
    let arraynested: [NestedConf] // nested array

    struct NestedConf: Configuration {
        let value: String? // optional
    }
}
```
#### 3. Bootstrap
Configuration must be bootstrapped before use. To do so, you need to conform to the `ConfigManager` protocol in the first place:
```swift
extension Conf: ConfigManager {
    public static var configuration: Conf? = nil
}
```

Next, call `bootstrap` method on your `ConfigManager` and pass sources you want to use:
```swift
try Conf.bootstrap(from: [EnvSource()])
```

**Remember that bootstrap must be called before using config and cannot be called more than once. Otherwise, `fatalError` will be thrown.**


#### 4. Usage
After completing all the previous steps you  can finally use config in your application.
You can access the configuration instance via `global` property:
```swift
Conf.global.simple
Conf.global.int
Conf.global.double
Conf.global.array[0] // Optional[String]
Conf.global.nested.value // Optional[String]
Conf.global.arraynested[0].value // Optional[String]
```

### Sources

Configuration can be fed with multiple sources.
Sources are passed into bootstrap function.

If you are using multiple sources, outputs are merged (structs are merged recursively, other values are overridden):
```swift
try Conf.bootstrap(from: [
    DictionarySource(dataSource: [
        "int": 1,
        "string": "some",
        "array": [1, 2],
        "nested": ["value1": 1]
    ]),
    DictionarySource(dataSource: [
        "string": "test",
        "array": [2, 3],
        "nested": ["value2": 2]
    ]),
])

// Output config:
[
    "int": 1,
    "string": "test",
    "array": [2, 3],
    "nested": [
        "value1": 1,
        "value2": 2
    ]
]
```


#### Dictionary source
Allows reading configuration from Dictionary, may be used to specify in-code defaults for configuration.

###### Example
```swift
try Conf.bootstrap(from: [
    DictionarySource(dataSource: [
        "int": 1,
        "string": "some",
        "array": [1, 2],
        "nested": ["value1": 1]
    ])
])
```


#### JSON source
Allows reading configuration from JSON data.

###### Example
```swift
try Conf.bootstrap(from: [
    JSONSource(dataSource: "{\"test\": \"sample\"}")
])
```


#### Environment source
Allows reading configuration data from environment.

###### Example
```swift
try Conf.bootstrap(from: [
    EnvSource(prefix: "SAMPLE")
])
```
Prefix can be set for `EnvSource`, so it reads only variables which key starts with a given value.

###### Sample Configuration
```swift
struct Conf: Configuration {
    let simple: String
    let int: Int
    let double: Double
    let nested: NestedConf
    let array: [String?]
    let arraynested: [NestedConf]

    struct NestedConf: Configuration {
        let value: String?
    }
}
```

Above example may be populated using following env variables:
```
SIMPLE="test"
INT="1"
DOUBLE="1.0"
NULL="null"
NESTED_VALUE="test"
ARRAY_0="test0"
ARRAY_1="test1"
ARRAY_2="null"
ARRAYNESTED_0_VALUE="test0"
ARRAYNESTED_1_VALUE="test1"
ARRAYNESTED_2_VALUE="null"
```
**Value "null" is coerced to internal nil value**


#### Command line source
Allows reading configuration data from environment.
###### Example
```swift
Conf.bootstrap(from: [
    CommandLineSource(prefix: "--config:my-prefix-")
])
```
If a prefix is set, only arguments which start with a given value will be read. Defaults to `--config:`

###### Sample Configuration
```swift
struct Conf: Configuration {
    let simple: String
    let int: Int
    let double: Double
    let null: String?
    let nested: NestedConf
    let array: [String?]
    let arraynested: [NestedConf]

    struct NestedConf: Configuration {
        let value: String?
    }
}
```

The example above may be populated using following command line arguments:
```
--config:simple=test
--config:int=1
--config:double=1.0
--config:null=null
--config:nested.value=test
--config:array.0=test0
--config:array.1=test1
--config:array.2=null
--config:arraynested.0.value=test0
--config:arraynested.1.value=test1
--config:arraynested.2.value=null
```
**Value "null" is coerced to internal nil value**


#### Custom sources
To create custom sources, you need to create a class that conforms to `ConfigSource`.
`DictionarySource` is the simplest working source that can be used as an example:
```swift
public typealias Storage = [String: Any?]

public class DictionarySource: ConfigSource {
    let dataSource: Storage

    public init(dataSource: Storage) {
        self.dataSource = dataSource
    }

    public func getConfig() throws -> Storage {
        return self.dataSource
    }
}
```

## SwiftBoxLogging
Logging system for Swift.

### Usage

#### 1. Import
```swift
import SwiftBoxLogging
```

#### 2. Bootstrap
Logging should be bootstrapped before use (it defaults to `PrintLogger`).
Bootstrap requires one parameter which is the logger factory.
Logger factory must return `Logger` from `Console/Logging` package.
```swift
Logging.bootstrap({ name in Logger2(name) })
```

#### 2. Usage
Create a logger instance:
```swift
fileprivate var logger = Logging.make(#file)
```

Log a message:
```swift
logger.verbose("verbose")
logger.debug("debug")
logger.info("info")
logger.warning("warning")
logger.error("error")
logger.fatal("fatal")
```

### Custom Loggers
To create custom loggers your class must conform to `Logger` protocol from `Console/Logging` package.

### Vapor
You can use same logging in Vapor and logging package:
```swift
private func configureLogging(_ config: inout Config, _ env: inout Environment, _ services: inout Services) {
    /// Register Logger2
    services.register(Logger2.self)

    switch env {
    case .production:
        config.prefer(Logger2.self, for: Logger.self)
        Logging.bootstrap({ name in Logger2(name) })
    default:
        config.prefer(PrintLogger.self, for: Logger.self)
        Logging.bootstrap({ _ in PrintLogger() })
    }
}
```


## SwiftBoxMetrics
Metrics for microservices, based on SSWG standard with Statsd built-in support.

Supported metric types:
- Counters
- Timers
- Gauges

### Usage

#### 1. Import
```swift
import SwiftBoxMetrics
```

#### 2. Bootstrap
Metrics must be bootstrap with Handler, that conforms to `MetricsHandler` protocol:
```swift
Metrics.bootstrap(
    try StatsDMetricsHandler(
        baseMetricPath: AppConfig.global.statsd.basePath!,
        client: UDPStatsDClient(
            config: UDPConnectionConfig(
                host: AppConfig.global.statsd.host!,
                port: AppConfig.global.statsd.port!
            )
        )
    )
)
```

#### 3. Usage
Timer metric may be used via convenient closure calls:
```swift
let result = try Metrics.global.withTimer(name: "myComputations") { () -> Double in
    return self.someComputationsToMeasure()
}
```

Manual usage:
```swift
Metrics.global.sendMetric(metric: TimerMetric(name: "timerMetric", value: 1001.1))
Metrics.global.sendMetric(metric: CounterMetric(name: "counterMetric", value: 1))
Metrics.global.sendMetric(metric: GaugeMetric(name: "gaugeMetric", value: 1, type: .increment))
```

### Handlers

#### LoggerMetricsHandler
Default handler for metrics that prints gathered metrics to console.

#### StatsDMetricsHandler
StatsD Metrics Handler responsible for sending gathered logs to statsD server. Supports TCP and UDP protocols.
Metrics are sent in separate thread, so operation is non-blocking for application.
```swift
try StatsDMetricsHandler(
    baseMetricPath: AppConfig.global.statsd.basePath!,
    client: UDPStatsDClient(
        config: UDPConnectionConfig(
            host: AppConfig.global.statsd.host!,
            port: AppConfig.global.statsd.port!
        )
    )
)
```
`baseMetricPath` is a path that will be prepended to every metric sent via handler.
`client` is a `TCPStatsDClient` or `UDPStatsDClient` instance.

#### Custom Handlers
To create custom handlers, conform to `MetricsHandler` class.
