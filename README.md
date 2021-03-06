# fty-common-logging

This is a library providing :
* A logging system based on the
  [log4cplus 1.1-9](https://github.com/log4cplus/log4cplus/tree/1.1.x)
  utility. For packaging and code maintainability reasons, a fork from
  https://github.com/42ity/log4cplus.git is currently used in practice.

## How to build

To build the `fty-common-logging` project run:

```bash
./autogen.sh
./configure
make
make check # to run self-test
```

## How to use Log System

### Logging levels

By default (with no log configuration file), the logging system uses the
`BIOS_LOG_LEVEL` environment variable to set the log level of the agent
and logs are redirected to the stdout console. If the environment variable
is set (and not empty), it also overrides the default level set in the
confguration file, if present.

This system sets 6 levels for logs with in order of importance (lowest
to highest) :

* `TRACE` Designates finer-grained informational events than the DEBUG.
* `DEBUG` Designates fine-grained informational events that are most useful to debug an application.
* `INFO`  Designates informational messages that highlight the progress of the application at coarse-grained level.
* `WARN`  Designates potentially harmful situations.
* `ERROR` Designates error events that might still allow the application to continue running.
* `FATAL` Designates very severe error events that will presumably lead the application to abort.

with the following matching between the values of `BIOS_LOG_LEVEL`, if set,
and the log4cplus level set by default then:

|    BIOS_LOG_LEVEL   |     Log level    |
| ------------------- | ---------------- |
| LOG_CRIT            |     FATAL        |
| LOG_ERR             |     ERROR        |
| LOG_WARNING         |     WARN         |
| LOG_INFO            |     INFO         |
| LOG_DEBUG           |     DEBUG        |
| Other               |     TRACE        |

### Notes for agents coded in C++

In the main method of the `.cpp` file, use this method :

```C++
void ManageFtyLog::setInstanceFtylog(std::string component, std::string configFile = "")
```

This method has two parameters :
* `component` : name of the agent, example : `fty-alert-list`
* `configFile` (optional) : path to the log config file, example : `/etc/fty-alert-list/logging.conf`.

The Ftylog object is still accessible by using the function :

```C++
Ftylog * ManageFtyLog::getInstanceFtylog()
```

### Notes for agents coded in C

In the main method of the `.c` file, for initialization of the log object,
call this method :

```C
void ftylog_setInstance(const char * component, const char * logConfigFile)
```

Same parameters as the C++ method, but with `configFile` not optional.

And to get the `Ftylog` object, call

```C
Ftylog * ftylog_getInstance()
```

### How to log
Use these macros to log any event in the agent (C++ or C) :

* `log_trace(...)` : log a TRACE event.
* `log_debug(...)` : log a DEBUG event.
* `log_info(...)` : log a INFO event.
* `log_warning(...)` : log a WARN event.
* `log_error(...)` : log a ERROR event.
* `log_fatal(...)` : log a FATAL event.

The `...` section is a string followed by any parameters as in the `printf`
family of functions.

### How to format log
The logging system uses the format from `patternlayout` of `log4cplus` (see
http://log4cplus.sourceforge.net/docs/html/classlog4cplus_1_1PatternLayout.html
for details).

If there is no configuration file, the default format is :
````
"%c [%t] -%-5p- %M (%l) %m%n"
````

For example, with this code :

```C
log_info(test, "This is a %s log test number %d", "info", 1);
```

The log generated will be :
````
log-default-67358592 [140004551534400] -INFO - log_fty_log_test (src/log/fty_log.cc:481) This is a info log test number 1
````

In your system, you can export an environment variable named `BIOS_LOG_PATTERN`
to set a format pattern for all agents using `fty-common-logging` and if the agent does
not use a specific log configuration file.

### Log configuration file
The agent can set a path to a log configuration file. The file uses the syntax
of a `log4cplus` configuration file (which is largely inspired from `log4j`
syntax).

So for a hypothetical `fty-test` agent, to redirect `INFO` messages to the
console and save `ERROR` messages in a file, the log configuration file will
be:

````
#Logger definition
log4cplus.logger.fty-test=INFO, console, file

#Console Definition
log4cplus.appender.console=log4cplus::ConsoleAppender
log4cplus.appender.console.layout=log4cplus::PatternLayout
log4cplus.appender.console.layout.ConversionPattern=[%-5p][%d] %m%n

#File definition
log4cplus.appender.file=log4cplus::RollingFileAppender
log4cplus.appender.file.File=/tmp/logging.txt
log4cplus.appender.file.MaxFileSize=16MB
log4cplus.appender.file.MaxBackupIndex=1
log4cplus.appender.file.Threshold=ERROR
log4cplus.appender.file.layout=log4cplus::PatternLayout
log4cplus.appender.file.layout.ConversionPattern=[%-5p][%D{%Y/%m/%d %H:%M:%S:%q}][%-l][%t] %m%n
````

Note that the name after the `log4cplus.logger.` string **MUST BE** the
same as the "component" parameter when you create a `Ftylog` object.

Every minute, a small thread checks if the log configuration file has
been modified or not. If modified, the logging system would reload the
log configuration file. This does not apply to the case if the file was
not present at the time of logging system initialization.

The object where log events are redirected is called an "appender".
Log4cplus defines several types of appenders :

|    Type                  |     Description  |
| ------------------------ | ---------------- |
| AsyncAppender            | Log events asynchronously.   |
| CLFSAppender             | Log object based on Microsoft Common Log File System API   |
| ConsoleAppender          | Log events in stdout         |
| FileAppender             | Log events in a file         |
| DailyRollingFileAppender | Log events in a file with backup after a period of time     |
| RollingFileAppender      | Log events in file with backup when a size has been reached |
| Log4jUdpAppender         | Sends log events as Log4j XML to a remote log server.       |
| NullAppender             | Never outputs a log event to any device.           |
| Qt4DebugAppender         | Log object based on Qt4's qDebug(), qWarning() and qCritical() functions. |
| SocketAppender           | Log events to a remote log server, usually a socketNode.  |
| SyslogAppender           | Log events to a remote syslog daemon.              |

See http://log4cplus.sourceforge.net/docs/html/classlog4cplus_1_1Appender.html
for more information about appenders.

### Verbose mode

For an agent with a verbose mode, you can call the C++ class method
`FtyLog::setVerboseMode()` (or `ftylog_setVerboseMode(Ftylog* log)`
for C code) to change the logging system :

* It sets (or overwrites if existing) a `ConsoleAppender` object with
  the `TRACE` logging level with default format or with the format
  defined by the `BIOS_LOG_PATTERN` environment variable.

### Utilities

The following C++ class functions test if a log level is included in the
log level of the agent :

* `bool FtyLog::isLogTrace()`
* `bool FtyLog::isLogDebug()`
* `bool FtyLog::isLogInfo()`
* `bool FtyLog::isLogWarning()`
* `bool FtyLog::isLogError()`
* `bool FtyLog::isLogFatal()`

And for C code :

* `bool ftylog_isLogTrace(Ftylog * log)`
* `bool ftylog_isLogDebug(Ftylog * log)`
* `bool ftylog_isLogInfo(Ftylog * log)`
* `bool ftylog_isLogWarning(Ftylog * log)`
* `bool ftylog_isLogError(Ftylog * log)`
* `bool ftylog_isLogFatal(Ftylog * log)`

For example if the log level of the agent is INFO, the function `isLogError()`
will return `true` and the function `isLogDebug()` will return `false`.

### Use for Test only

The following methods change dynamically the logging level of the `Ftylog`
object to a specific log level, so **only use it for testing**.

This class methods are :
* `void FtyLog::setLogLevelTrace()`
* `void FtyLog::setLogLevelDebug()`
* `void FtyLog::setLogLevelInfo()`
* `void FtyLog::setLogLevelWarning()`
* `void FtyLog::setLogLevelError()`
* `void FtyLog::setLogLevelFatal()`

And for C code :
* `void ftylog_setLogLevelTrace(Ftylog * log)`
* `void ftylog_setLogLevelDebug(Ftylog * log)`
* `void ftylog_setLogLevelInfo(Ftylog * log)`
* `void ftylog_setLogLevelWarning(Ftylog * log)`
* `void ftylog_setLogLevelError(Ftylog * log)`
* `void ftylog_setLogLevelFatal(Ftylog * log)`

## How to use Common functions

### For agent coded in C++

In the main `.h` file add:

```C++
#include <fty_log.h>
```

### For agents coded in C

In the main `.h` file add

```C
#include <fty-log/fty_logger.h>
```

### Macro

The following macros are defined :
 * `#define STR(X) #X`
 * `#define streq(s1,s2) (!strcmp ((s1), (s2)))` (if not already defined)

## How to compile and test using 42ITy standards

### project.xml

Add this block in the `project.xml` file :

````
    <use project = "fty-common-logging" libname = "libfty_common_logging" header="fty_log.h"
        repository = "https://github.com/42ity/fty-log.git"
        release = "master"
        test = "fty_log_selftest" >

        <use project = "log4cplus" header = "log4cplus/logger.h" test = "appender_test"
            repository = "https://github.com/42ity/log4cplus.git"
            release = "1.1.2-FTY-master"
            />

        <use project = "cxxtools" test="cxxtools::Utf8Codec::Utf8Codec" header="cxxtools/allocator.h"
            repository = "https://github.com/42ity/cxxtools.git"
            release = "2.2-FTY-master"
            />
    </use>
````

NOTE: The `header` value must be changed from `fty_log.h` to
`fty-log/fty_logger.h` for a C project.

NOTE: In this `use` section, remove the dependecy already needed directly
by the particular agent/library whose project you are configuring, to
simplify maintenance of the configuration later.

### How to pass Travis CI checks

Re-generating the project structure and recipes with `zproject` templates
will define installation of `log4cplus` package from the OS distribution,
which is too old for our needs and is lacking otherwise. You should comment
it away, to ensure that the 42ITy fork referenced above would be used.

In recent zproject revisions, dependency code that can be compiled from
source or taken from packages (the 42ITy project does not publish any at
the moment) is defined in a separate list, so it suffices to comment away
the reference to this list:

````
# NOTE: Our forks are checked out and built without pkg dependencies in use
pkg_deps_prereqs: &pkg_deps_prereqs
#    - *pkg_deps_prereqs_source
    - *pkg_deps_prereqs_distro
````

Alternately, to avoid any errors, you can add these two lines in the
`before_install` section of the `.travis.yml` file, to remove these
packages if added into the build system (e.g. by some other dependencies):

````
before_install:
- sudo apt-get remove liblog4cplus-dev
- sudo apt-get autoremove
````

It is not recommended to do this right away (before the problem bites in
practice) because calls to packaging have considerable overheads in the
run-times of the tests.
