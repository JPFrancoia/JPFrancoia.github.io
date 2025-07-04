---
layout: posts
title:  "Observability: logging in Python"
image: logging_python.png

excerpt: "Let's explore how to setup logging in Python. We'll go through the
configurations of a colored handler for local development, and of a structured
handler for production. We'll also talk about a few tips to get the most out of
your logging if you're running your app on GCP with Stackdriver."

---

# Observability: logging in Python

In this article, I'll try and centralize the knowledge I gathered over the
years about logging in Python. The topic is much more vast than what one
might think...

I would like to build on top of the official documentation - instead of just
copy-pasting it - and share some tips and tricks that I learned along the
way. I have included a prerequisites section with links to a few helpful
resources to get you started.


## Prerequisites

- [Basic logging tutorial](https://docs.python.org/3/howto/logging.html#basic-logging-tutorial)
- [Advanced logging tutorial](https://docs.python.org/3/howto/logging.html#advanced-logging-tutorial)


## Reminders

Logging in Python revolves around three elements:

- **Loggers**: these are the objects that you use to log messages. They are
  usually created using the `logging.getLogger()` function.
- **Handlers**: these are the objects that send the log messages to their
  final destination. They can send messages to the console, to a file, or to
  a remote server.
- **Formatters**: these are the objects that format the log messages before
  they are sent to their final destination. They can add timestamps, log levels,
  and other information to the messages.

## Tips and tricks


### Use a config file, not code, to configure the logging

I like to decouple the configuration of the logging from the code itself. In
theory, this allows you to change the logging configuration without touching
the code. I normally initialize the logging like this:

```python
import logging
import logging.config

from app import config


def init_logging() -> None:
    if (filename := config.LOGGING_CONFIG) is not None:
        # Set 'disable_existing_loggers' to 'False' so
        # module-level loggers aren't disabled
        logging.config.fileConfig(filename, disable_existing_loggers=False)
    else:
        logging.warning(
            "LOGGING_CONFIG environment variable is not set, "
            "falling back to basic logging"
        )

        # Clear the handlers otherwise basicConfig will not work
        logging.getLogger().handlers = []

        logging.basicConfig(
            format="%(asctime)s [%(levelname)s] %(message)s",
            level=logging.DEBUG,
        )
```

and in `app/config.py`, I have something resembling this:

```python
import os

LOGGING_CONFIG = os.getenv("LOGGING_CONFIG", "logging.conf")
```

I just need to call `init_logging()` at the start of my application/script
and the logging is configured. I normally bundle `logging.conf` with my
code (in my Docker image for example). If I need to change the logging
configuration, I just need to provide a different file and to change the
`LOGGING_CONFIG` environment variable to point to the new file. If I need
a quick config change, I can just create a volume in my Docker container
to provide a new config file without rebuilding the image. We will see a
handful of examples of logging configurations later in this article.

When I'm working in a local development environment, I like to use a different
configuration file that uses a colored handler and prints the logs to the
console (we'll talk about that later). This config is different from the
one I use in production, and I need to be able to switch between the two
without changing the code. I can run my application like this:

```bash
LOGGING_CONFIG=logging.dev.conf python -m app.main
```

### Use the config file to do different things

Here is a basic configuration I always like to use when I start a new project:

```ini
# Configure the logging
# https://docs.python.org/3/library/logging.config.html#logging-config-fileformat

[loggers]
keys=root

[handlers]
keys=consoleHandler,fileHandler

[formatters]
keys=default

[logger_root]
level=DEBUG
handlers=consoleHandler,fileHandler

[handler_consoleHandler]
class=logging.StreamHandler
level=DEBUG
formatter=default

[handler_fileHandler]
class=logging.handlers.RotatingFileHandler
level=DEBUG
formatter=default
args=("activity.log",)
kwargs={"maxBytes": 5242880}

[formatter_default]
format=%(asctime)s - %(levelname)8s %(message)-50s [%(filename)s, l %(lineno)d in %(funcName)s]
```

It uses two handlers:
- A **console handler** that print the logs to the console
- A **file handler** that writes the logs to a file called `activity.log`,
with a maximum size of 5MB, after which the file will be rotated

Both handlers use a custom formatter that make the logs more readable, which is
perfect for local development. But let's have a closer look at the file
handler:

```ini
[handler_fileHandler]
class=logging.handlers.RotatingFileHandler
level=DEBUG
formatter=default
args=("activity.log",)
kwargs={"maxBytes": 5242880}
```

Notice the line `class=logging.handlers.RotatingFileHandler`? This configures a
handler from the `logging.handlers` module, which is a part of the standard
library. But we can also use any third-party library that provides a handler!
[Here](https://kislyuk.github.io/watchtower/#examples-python-logging-config) is
an example of using the `watchtower` library to send logs to AWS CloudWatch
(the config is in yaml, but it could also be in ini format):

```yaml
version: 1
handlers:
  # ...Omit the console and file handlers for brevity...
  watchtower:
    class: watchtower.CloudWatchLogHandler
    level: DEBUG
    log_group_name: watchtower
    log_stream_name: "{logger_name}-{strftime:%y-%m-%d}"
    send_interval: 10
    create_log_group: False
```

It's not important here to understand how CloudWatch works, what matters is
what the handler does: it takes standard log messages, massages them a bit,
and "sends" them to CloudWatch. During the massaging steps, anything could
happen (tagging of the log messages, adding metadata, etc). And none of this
is related to the application code itself.

Most cloud providers (AWS and GCP do for sure) provide libraries that
integrate with the standard logging module. They don't always document
precisely how to integrate, but it's always possible. I always refrain
from doing the integration in the code. Incidentally, enforcing the use a
config file naturally filters out what I consider "bad" logging libraries:
libraries that require configuration or initialization in the application
code itself. If a library doesn't integrate with the standard logging module,
I don't want to use it.

### Colored logging for local development

Because debugging is much easier with colored logs. I have had this formatter forever:

```python
import json
import logging
from typing import Any, Mapping


# Taken from:
# https://github.com/MyColorfulDays/jsonformatter/blob/f7908f1b2bc9e556aea29f26307643e732ac8b5e/src/jsonformatter/jsonformatter.py#L89
_LogRecordDefaultAttributes = {
    'name',
    'msg',
    'args',
    'levelname',
    'levelno',
    'pathname',
    'filename',
    'module',
    'exc_info',
    'exc_text',
    'stack_info',
    'lineno',
    'funcName',
    'created',
    'msecs',
    'relativeCreated',
    'thread',
    'threadName',
    'processName',
    'process',
    'message',
    'asctime',
    "otelSpanID",
    "otelTraceID",
    "otelTraceSampled",
    "otelServiceName",
    "taskName",
}


class ColorFormatter(logging.Formatter):
    """Logging colored formatter, adapted from https://stackoverflow.com/a/56944256/3638629"""

    grey = "\x1b[38;20m"
    cyan = "\x1b[36;20m"
    bold_green = "\x1b[32;1m"
    yellow = "\x1b[33;20m"
    red = "\x1b[31;20m"
    bold_red = "\x1b[31;1m"
    reset = "\x1b[0m"

    def __init__(self, fmt: str, *args: Any, **kwargs: Mapping[str, Any]) -> None:
        super().__init__()
        self.fmt = fmt
        self.FORMATS = {
            logging.DEBUG: self.cyan + self.fmt + self.reset,
            logging.INFO: self.bold_green + self.fmt + self.reset,
            logging.WARNING: self.yellow + self.fmt + self.reset,
            logging.ERROR: self.red + self.fmt + self.reset,
            logging.CRITICAL: self.bold_red + self.fmt + self.reset,
        }

    def format(self, record: logging.LogRecord) -> str:
        log_fmt = self.FORMATS.get(record.levelno)
        formatter = logging.Formatter(log_fmt)

        extras = get_records_extra_attrs(record)

        if (extras := get_records_extra_attrs(record)):
            record.msg = f"{record.msg}. Extras: {json.dumps(extras)}"

        return formatter.format(record)


def get_records_extra_attrs(record: logging.LogRecord) -> Mapping[str, Any]:
    """Extract extra attributes from a log record.

    Largely inspired from:
    https://github.com/MyColorfulDays/jsonformatter/blob/master/src/jsonformatter/jsonformatter.py#L344

    Args:
        record: extract extras from this record

    Returns:
        extra dict passed to the logger, as a full dict.

    """
    extras = {
        k: record.__dict__[k]
        for k in record.__dict__
        if k not in _LogRecordDefaultAttributes
    }
    return extras
```

And to use it in the config file, just create an additional formatter:

```ini
[formatter_color]
format=%(asctime)s - %(levelname)8s %(message)-50s [%(filename)s, l %(lineno)d in %(funcName)s]
class=your_module_name_here.color_formatter.ColorFormatter
```

You'll notice a bit of code related to the `extra` field of the log record.
Normally, you can pass an extra dict of data when logging a message:

```python
logger.info("This is an info message", extra={"key": "value"})
```

But since my custom `ColorFormatter` reimplements the `format` method, I needed
to reimplement the extraction of the `extra` field from the log record. The
code could be simplified if you're not interested in the `extra` field.

Ultimately, this is what it looks like when log messages are printed to the
console:

<p align="center">
  <img src="{{ site.baseurl }}/images/logging/console_logging.png">
</p>


### Structured logging

Once an application is in production, things are a bit different. You
won't really look at the logs *directly* (you won't look directly at the
output of stdout): your application will most likely be containerized
and deployed with several replicas, and it will most likely serve several
users, all accessing your application at the same time. This represents
*a lot* of logs. At this point, you'll want your logs to be outputted in
a machine-readable format, i.e: json. *Something* will read and centralize
these logs so that you can analyze them. I am being vague on purpose here:
it could be something like AWS's CloudWatch, GCP's Logs Explorer, Signoz,
etc. Ultimately, the logs will be presented to you with some form of UI
that allows you to filter logs by log level, timestamp, etc.

For all this to work in practice, your application just needs to output
json-formatted logs to stdout. That's all there is to it. Then the *something*
mentioned above will read the stdout stream of your application and will
process the logs. A basic version of this concept can be configured with:

```ini
[loggers]
keys=root

[handlers]
keys=consoleHandler

[formatters]
keys=json

[logger_root]
level=DEBUG
handlers=consoleHandler

[handler_consoleHandler]
class=logging.StreamHandler
level=DEBUG
formatter=json

[formatter_json]
format=%(asctime)s %(levelname)s %(message) %(otelTraceID)s %(otelSpanID)s %(otelServiceName)s %(otelTraceSampled)s %(filename)s %(lineno)d %(funcName)s
class=pythonjsonlogger.jsonlogger.JsonFormatter
```

And this is an example of the output:

```json
{"asctime": "2025-06-08 19:01:18,792", "levelname": "INFO", "message": "This is an info message", "otelTraceID": null, "otelSpanID": null, "otelServiceName": null, "otelTraceSampled": null, "filename": "test.py", "lineno": 30, "funcName": "main"}
{"asctime": "2025-06-08 19:01:18,793", "levelname": "INFO", "message": "This is an info message", "otelTraceID": null, "otelSpanID": null, "otelServiceName": null, "otelTraceSampled": null, "filename": "test.py", "lineno": 31, "funcName": "main", "key": "value"}
{"asctime": "2025-06-08 19:01:18,793", "levelname": "DEBUG", "message": "This is a debug message", "otelTraceID": null, "otelSpanID": null, "otelServiceName": null, "otelTraceSampled": null, "filename": "test.py", "lineno": 32, "funcName": "main"}
{"asctime": "2025-06-08 19:01:18,793", "levelname": "WARNING", "message": "This is a warning message", "otelTraceID": null, "otelSpanID": null, "otelServiceName": null, "otelTraceSampled": null, "filename": "test.py", "lineno": 33, "funcName": "main"}
{"asctime": "2025-06-08 19:01:18,793", "levelname": "ERROR", "message": "This is an error message", "otelTraceID": null, "otelSpanID": null, "otelServiceName": null, "otelTraceSampled": null, "filename": "test.py", "lineno": 34, "funcName": "main"}
{"asctime": "2025-06-08 19:01:18,793", "levelname": "CRITICAL", "message": "This is a critical message", "otelTraceID": null, "otelSpanID": null, "otelServiceName": null, "otelTraceSampled": null, "filename": "test.py", "lineno": 35, "funcName": "main"}
```

In this config, I'm using a vendor-agnostic formatter:
`pythonjsonlogger.jsonlogger.JsonFormatter`. The standard logging fields are
used as keys of the log records. If you're running your app in the cloud
(on AWS, GCP, etc) you'll probably want to use the formatter provided by
the provider. This formatter will add specific fields to the log records,
and these fields will be picked up by the your provider's monitoring solution.


#### Example: Logging on GCP

This is just an example of a custom json formatter that adds an extra
field (the open telemetry trace id) to the log records. It inherits from
`pythonjsonlogger.jsonlogger.JsonFormatter` and enhances the `add_fields`
method to add an extra field to the log record. There is nothing more to it
than adding an extra field to a json object. However, GCP's logging agents
will recognize the `logging.googleapis.com/trace` field and will use it for
different things. The list of special json fields on GCP can be found [here](
https://cloud.google.com/logging/docs/structured-logging#structured_logging_special_fields).

```python
from opentelemetry import trace
from pythonjsonlogger import jsonlogger

class CustomJsonFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super(CustomJsonFormatter, self).add_fields(log_record, record, message_dict)

        # The logging needs to be instrumented with OTEL, it won't work as is
        if not log_record.get("span") and log_record.get("otelTraceID", "0") != "0":
            # The trace id needs to be formatted with hexadecimal otherwise the ID isn't
            # readable in cloud logging
            # https://cloud.google.com/trace/docs/trace-log-integration
            # https://stackoverflow.com/a/76046022/1585507
            trace_id = format(
                trace.get_current_span().get_span_context().trace_id, "032x"
            )

            # Can be retrieved in many ways
            project_id = config.PROJECT_ID

            # Field with a special name.
            # MUST use this name to make cloud trace pick it up
            # https://cloud.google.com/logging/docs/structured-logging
            log_record[
                "logging.googleapis.com/trace"
            ] = f"projects/{project_id}/traces/{trace_id}"
```

When I'm deploying applications on GCP Cloud Run, that's the only formatter I
use. I could probably use Google's
[StructuredLogHandler](https://cloud.google.com/python/docs/reference/logging/latest/std-lib-integration#manual-handler-configuration)
instead of my custom formatter, but since the custom formatter is working and is
simple, I never took the time to try something else...


### Sending logs to OpenTelemetry

I'll briefly mention [OpenTelemetry (Otel)](https://opentelemetry.io/) and how it
ties in with logs (I will most likely write a dedicated article about Otel in the
future). Otel is a set of tools that generate all sorts of telemetry (metrics, logs,
traces) from your applications. This telemetry is absolutely vital for monitoring
and debugging. Otel is open source and vendor neutral, and has emerged as (I think)
the observability standard in the last few years. It's the future.

Otel provides APIs for traces and metrics. That means once *instrumented*, your
applications will make API calls (in the background) to an OpenTelemetry
*collector* to record traces and metrics.  

But Otel also provides a [logs
API](https://opentelemetry.io/docs/specs/otel/logs/api/), which is slightly
less known. Once instrumented, your applications could send logs to an Otel
collector through API calls. This is a bit different than what is explained
above: instead of being read and parsed from the stdout of your applications,
the logs are sent to the collector. I believe this simplifies things (no
need for a custom formatter to add the trace id for example). However,
the logs API isn't supported yet by all the observability solutions out
there. For example, GCP's monitoring solution accepts API calls for traces
and metrics, but not for logs.

## Conclusion

Setting up proper logging in Python isn't complicated, once you know the tips and
tricks of this article. Here are the key takeaways:

- Use colored logging for local development to make debugging easier 
- Switch to structured JSON logging for production to enable proper monitoring
- Decouple your logging configuration from your code

I would always advise to start with the basic configuration examples from this
article, then gradually add more sophisticated features like cloud integration or
OpenTelemetry as your observability needs grow. Good logging practices will save
you countless hours when debugging issues in production.
