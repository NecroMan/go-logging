go-logging
==========

Fork of another library with additional features:
* Handler's and logger's severity level now is not just a minimum level but a list of levels that the handler and the logger should process;
* Colorizing record's severity level (for enable add "colorize:" to begin of format string); 
* Fixed bugs.

---

```go-logging``` is a Golang library that implements the Python-like logging facility. 

As we all know that logging is essientially significant for server side programming because in general logging the only way to report what happens inside the program. 

The ```logging``` package of Python standard library is a popular logging facility among Pythoners. ```logging``` defines ```Logger``` as logging source, ```Handler``` as logging event destination, and supports Logger hierarchy and free combinations of both.  It is powerful and flexible,  in a similar style like ```Log4j```, which is a popular logging facility among Javaers.

When it comes to Golang, the standard release has a library called ```log``` for logging. It's simple and good to log something into standard IO or a customized IO. In fact it's too simple to use in any **real** production enviroment, especially when compared to some other mature logging library. 

Due to the lack of a good logging facility, many people start to develop their own versions. For example in github there are dozens of logging repositories for Golang. I run into the same problem when I am writing some project in Golang. A powerful logging facility is needed to develop and debug it. I take a search on a few existing logging libraries for Golang but none of them seems to meet the requirement. So I decide to join the parade of "everyone is busy developing his own version", and then this library is created.

## Features

With an obivious intention to be a port of ```logging``` for Golang, ```go-logging``` has all the main features that ```logging``` package has:

1. It supports logging level, logging sources(logger) and destinations(handler) customization and flexible combinations of them
2. It supports logger hierarchy, optional filter on logger and handler, optional formatter on handler
3. It supports handlers that frequently-used in most real production enviroments, e.g. it could write log events to stdout, memory, file, syslog, udp/tcp socket, rpc(e.g., thrift. For the corresponding servers, please refer to the unit test) etc.
4. It could be configured throught handy config file in various format(e.g. yaml, json)

## Usage

Get this library using the standard go tool:

```bash
go get github.com/NecroMan/go-logging
```

#### Example 1: Log to standard output

```go
package main

import (
	"github.com/NecroMan/go-logging"
)

func main() {
	logger := logging.GetLogger("a.b")
	handler := logging.NewStdoutHandler()
	logger.AddHandler(handler)
	logger.Warnf("message: %s %d", "Hello", 2015)
}
```

The code above outputs as the following:

```text
message: Hello 2015
```

#### Example 2: Log to file

```go
package main

import (
	"github.com/NecroMan/go-logging"
	"os"
	"time"
)

func main() {
	filePath := "./test.log"
	fileMode := os.O_APPEND
	bufferSize := 0
	bufferFlushTime := 30 * time.Second
	inputChanSize := 1
	// set the maximum size of every file to 100 M bytes
	fileMaxBytes := uint64(100 * 1024 * 1024)
	// keep 9 backup at most(including the current using one,
	// there could be 10 log file at most)
	backupCount := uint32(9)
	// create a handler(which represents a log message destination)
	handler := logging.MustNewRotatingFileHandler(
		filePath, fileMode, bufferSize, bufferFlushTime, inputChanSize,
		fileMaxBytes, backupCount)

	// the format for the whole log message
	format := "%(asctime)s %(levelname)s (%(filename)s:%(lineno)d) " +
		"%(name)s %(message)s"
	// the format for the time part
	dateFormat := "%Y-%m-%d %H:%M:%S.%3n"
	// create a formatter(which controls how log messages are formatted)
	formatter := logging.NewStandardFormatter(format, dateFormat)
	// set formatter for handler
	handler.SetFormatter(formatter)

	// create a logger(which represents a log message source)
	logger := logging.GetLogger("a.b.c")
	logger.SetLevels([]logging.LogLevelType{logging.LevelInfo})
	logger.AddHandler(handler)

	// ensure all log messages are flushed to disk before program exits.
	defer logging.Shutdown()

	logger.Infof("message: %s %d", "Hello", 2015)
}
```

Compile and run the code above, it would generate a log file "./test.log" under current working directory. The log file contains a single line:

```text
2015-04-04 14:20:33.714 INFO (main2.go:40) a.b.c message: Hello 2015
```

#### Example 3: Config Log via configuration file.

Write a configuration file ```config.yml``` as the following:

```yaml
formatters:
    f:
        format: "%(asctime)s %(levelname)s (%(filename)s:%(lineno)d) %(name)s %(message)s"
        datefmt: "%Y-%m-%d %H:%M:%S.%3n"
    t:
        format: "colorize: %(asctime)s %(levelname)s (%(filename)s:%(lineno)d) %(name)s %(message)s"
        datefmt: "%H:%M:%S.%3n"
handlers:
    h:
        class: RotatingFileHandler
        filepath: "./test.log"
        mode: O_APPEND
        bufferSize: 0
        # 30 * 1000 ms -> 30 seconds
        bufferFlushTime: 30000
        inputChanSize: 1
        # 100 * 1024 * 1024 -> 100M
        maxBytes: 104857600
        backupCount: 9
        formatter: f
loggers:
    a.b.c:
        levels: [INFO]
        handlers: [h]

```

and use it to config logging facility like:

```go
package main

import (
	"github.com/NecroMan/go-logging"
)

func main() {
	configFile := "./config.yml"
	if err := logging.ApplyConfigFile(configFile); err != nil {
		panic(err.Error())
	}
	logger := logging.GetLogger("a.b.c")
	defer logging.Shutdown()
	logger.Infof("message: %s %d", "Hello", 2015)
}
```

It will write log as the same as the above example 2.

## Configurable parameters

#### filters

Example:

```yaml
filters:
  db: "db."
  pgsql: "db.pgsql"
```

Value of the filter is exact logger name or its name's prefix (add dot (".") in the end) by which messages are filtered
among all messages.

#### formatters

Example:

```yaml
formatters:
  f:
    format: "%(asctime)s %(levelname)s (%(filename)s:%(lineno)d) %(name)s %(message)s"
    datefmt: "%Y-%m-%d %H:%M:%S.%3n"
  t:
    format: "colorize: %(asctime)s %(levelname)s %(name)s %(message)s"
    datefmt: "%H:%M:%S.%3n"
```

Parameters:

`format` - record's format string, supporting variables:
* `%(name)s` - logger's name
* `%(levelno)d` - severity leven in decimal (0-50)
* `%(levelname)s` - severity level name (NOTSET, TRACE, DEBUG, INFO, WARN, ERROR, FATAL)
* `%(pathname)s` - path name of file triggered record
* `%(filename)s` - file name triggered record
* `%(lineno)d` - line number in file triggered record
* `%(funcname)s` - function name triggered record
* `%(created)d` - record creation time in Unix nanoseconds (since January 1st 1970 UTC)
* `%(asctime)s` - formatted record creation time
* `%(message)s` - message

`datefmt` - date and time format for variable `asctime`, supporting:
* `%a` - Locale’s abbreviated weekday name
* `%A` - Locale’s full weekday name
* `%b` - Locale’s abbreviated month name
* `%B` - Locale’s full month name
* `%c` - Locale’s appropriate date and time representation
* `%d` - Day of the month as a decimal number [01,31]
* `%H` - Hour (24-hour clock) as a decimal number [00,23]
* `%I` - Hour (12-hour clock) as a decimal number [01,12]
* `%j` - Day of year
* `%m` - Month as a decimal number [01,12]
* `%M` - Minute as a decimal number [00,59]
* `%p` - Locale’s equivalent of either AM or PM
* `%S` - Second as a decimal number [00,61]
* `%U` - Week number of the year
* `%w` - Weekday as a decimal number
* `%W` - Week number of the year
* `%x` - Locale’s appropriate date representation
* `%X` - Locale’s appropriate time representation
* `%y` - Year without century as a decimal number [00,99]
* `%Y` - Year with century as a decimal number
* `%Z` - Time zone name (no characters if no time zone exists)

#### handlers

Example:

```yaml
handlers:
  stdout:
    class: StdoutHandler
    levels: [trace, debug, info, warn, error, fatal]
    formatter: t
  base:
    class: RotatingFileHandler
    filepath: "./logs/log.log"
    mode: O_APPEND
    # no memory buffer
    bufferSize: 0
    # 30 * 1000 ms -> 30 seconds
    bufferFlushTime: 30000
    inputChanSize: 1
    # 100 * 1024 * 1024 -> 100M
    maxBytes: 104857600
    backupCount: 3
    levels: [debug, info, warn, error, fatal]
    formatter: f
  error:
    class: RotatingFileHandler
    filepath: "./logs/errors.log"
    mode: O_APPEND
    # no memory buffer
    bufferSize: 0
    # 30 * 1000 ms -> 30 seconds
    bufferFlushTime: 30000
    inputChanSize: 0
    # 100 * 1024 * 1024 -> 100M
    maxBytes: 104857600
    backupCount: 3
    levels: [error, fatal]
    formatter: f
```

Typical parameters:

`class` - handler's class.\
`levels` - severity levels that the handler should process.\
`filters` - list of filters that should be checked to pass-through the message.\
`formatter` - name of formatter using to format log message.

* `NullHandler` - null handler :) without additional parameters
* `MemoryHandler` - in memory buffer with limited capacity forwarding messages to another handler. Parameters:
    * `capacity` - buffer capacity before flush messages to target
    * `level` - minimum severity level which triggers buffer flush to target
    * `target` - target handler to which messages are sent when buffer flushing
* `StdoutHandler` - outputs messages to stdout, no additional parameters
* `FileHandler` - save messages to file. Parameters:
    * `filename` - path and file name; it will be created if not exists
    * `mode` - file open mode (O_RDONLY, O_WRONLY, O_RDWR... the same as in `os` package)
    * `bufferSize` - buffer size to keep data in memory before flush them to file
* `RotatingFileHandler` - save messages to file with ability to rollover by file size. Parameters:
    * `filepath` - path to save files to; it will be created if not exists
    * `mode` - file open mode (O_RDONLY, O_WRONLY, O_RDWR... the same as in `os` package)
    * `bufferSize` - buffer size to keep data in memory before flushes them to file
    * `bufferFlushTime` - interval in milliseconds when in memory buffer flushes to file
    * `inputChanSize` - if positive, handler starts no go routine with specified chan size
    * `maxBytes` - maximum size of one file before creating a new one
    * `backupCount` - number of backup files to keep
* `TimedRotatingFileHandler` - save messages to file with ability to rollover by time interval. Parameters:
    * `filepath` - path to save files to; it will be created if not exists
    * `mode` - file open mode (O_RDONLY, O_WRONLY, O_RDWR... the same as in `os` package)
    * `bufferSize` - buffer size to keep data in memory before flushes them to file
    * `when` - type of rollover interval (S - Seconds, M - Minutes, H - Hours, D - Days, midnight - roll over at midnight, W{0-6} - roll over on a certain weekday; 0 - Monday)
    * `interval` - size of interval
    * `backupCount` - number of backup files to keep
    * `utc` - boolean value when `true` for UTC time zone and `false` for Local
* `SyslogHandler` - sends messages to syslog server. Parameters:
    * `priority` - combination of the syslog facility and severity
    * `tag` - syslog writer tag
* `DatagramHandler` - sends messages in gob format through UDP. Parameters:
    * `host` - host address
    * `port` - port
* `SocketHandler` - sends messages in gob format through TCP. Parameters:
    * `host` - host address
    * `port` - port

#### loggers

Example:

```yaml
root:
  levels: [trace, debug, info, warn, error, fatal]
  handlers: [stdout, base, error]
loggers:
  db:
    propagate: true
    handlers: [base, db]
  api:
    propagate: true
    handlers: [api]
```

`root` is a topmost logger with special name. It handles all messages without configured logger and all propagated messages.
Root logger has no `propagate` parameter.

Parameters:

`levels` - severity levels that the logger should process.\
`handlers` - list of handlers that should be called to handle log record.\
`filters` - list of filters that should be checked to pass-through the message.\
`propagate` - boolean (`true` | `false`) flag that enables or disables propagation of record processing to "upper" logger(s).