# StackdriverLogging
A swift-log  `LogHandler` that logs GCP Stackdriver formatted JSON to a file. (https://github.com/apple/swift-log)

For more information on Stackdriver structured logging, see: https://cloud.google.com/logging/docs/structured-logging and [LogEntry](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry)

## Dependencies 
This Stackdriver `LogHandler` has a dependency on swift-nio which is used to create and save your new log entries in a non-blocking fashion.
https://github.com/apple/swift-nio 

## Bootstrapping 
Here is an example of how a  `StackdriverLogHandler` could be bootstrapped, notice that the `StackdriverLogHandler` initializer will throw if it receives an invalid filepath.  

```Swift
LoggingSystem.bootstrap { label -> LogHandler in
    // ...
    if label == "Stackdriver" {
        do {
            var logHandler = try StackdriverLogHandler(logFilePath: "/var/log/my_app.log")
            logHandler.logLevel = .error
            return logHandler
        } catch {
            // The logFilePath does not exist or is inacessible, defaulting to a console logger
            print("Failed to create a StackdriverLogHandler with error: '\(error.localizedDescription)'")
            return ConsoleLogger()
        }
    }
    // ...
}    
```
## Logging JSON values using `Logger.MetadataValue`
To log metadata values as JSON, simply log all JSON values other than `String` as a `Logger.MetadataValue.stringConvertible` and, instead of the usual conversion of your value to a `String` in the log entry, it will keep the original JSON type of your values whenever possible.

For example:
```Swift
var logger = Logger(label: "Stackdriver")
logger[metadataKey: "jsonpayload-example-object"] = [
    "json-null": .stringConvertible(NSNull()),
    "json-bool": .stringConvertible(true),
    "json-integer": .stringConvertible(1),
    "json-float": .stringConvertible(1.5),
    "json-string": .string("Example"),
    "stackdriver-timestamp": .stringConvertible(Date()),
    "json-array-of-numbers": [.stringConvertible(1), .stringConvertible(5.8)],
    "json-object": [
        "key": "value"
    ]
]
logger.info("test")
```
Will log the non pretty-printed representation of:
```json
{  
   "sourceLocation":{  
      "function":"boot(_:)",
      "file":"\/Sources\/App\/boot.swift",
      "line":25
   },
   "jsonpayload-example-object":{  
      "json-bool":true,
      "json-float":1.5,
      "json-string":"Example",
      "json-object":{  
         "key":"value"
      },
      "json-null":null,
      "json-integer":1,
      "json-array-of-numbers":[  
         1,
         5.8
      ],
      "stackdriver-timestamp":"2019-07-15T21:21:02.451Z"
   },
   "message":"test",
   "severity":"INFO"
}
```

## Stackdriver logging agent + fluentd config 
You can use this LogHandler in combination with the Stackdriver logging agent https://cloud.google.com/logging/docs/agent/installation and a matching json format
google-fluentd config (/etc/google-fluentd/config.d/example.conf) to automatically send your JSON logs to Stackdriver. 

Here's an example google-fluentd conf file that monitors a json based logfile and send new log entries to Stackdriver:
```
<source>
    @type tail
    # Format 'JSON' indicates the log is structured (JSON).
    format json
    # The path of the log file.
    path /var/log/example.log
    # The path of the position file that records where in the log file
    # we have processed already. This is useful when the agent
    # restarts.
    pos_file /var/lib/google-fluentd/pos/example-log.pos
    read_from_head true
    # The log tag for this log input.
    tag exampletag
</source>
```
