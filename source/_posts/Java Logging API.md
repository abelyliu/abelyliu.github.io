---
title: Java Logging API
date: 2016-10-24 10:56:54
tags: Java
category: Java
---
使用JDK自带的日志工具类，记录日志。
<!--more-->
## 快速入门

```java
import java.util.logging.Logger;

public class Main {
    private  final static Logger LOGGER = Logger.getLogger(Main.class.getName());
    public static void main(String[] args) {
        LOGGER.info("info");
        LOGGER.warning("warning");
    }
}
```

控制台输出
```
十月 24, 2016 10:01:35 上午 Main main
信息: info
十月 24, 2016 10:01:35 上午 Main main
警告: warning
```

日志的级别如下：
- OFF
- SEVERE
- WARNING
- INFO
- CONFIG
- FINE
- FINER
- FINEST
- ALL

## 日志配置

```java
import java.io.IOException;
import java.util.logging.FileHandler;
import java.util.logging.Level;
import java.util.logging.Logger;

public class Main {
    private final static Logger LOGGER = Logger.getLogger(Main.class.getName());

    public static void main(String[] args) throws IOException {
        LOGGER.setLevel(Level.WARNING);
        FileHandler fileHandler = new FileHandler("log.txt");
        LOGGER.addHandler(fileHandler);
        LOGGER.info("info");
        LOGGER.warning("warning");
    }
}
```

log.txt
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE log SYSTEM "logger.dtd">
<log>
<record>
  <date>2016-10-24T10:21:51</date>
  <millis>1477275711521</millis>
  <sequence>0</sequence>
  <logger>Main</logger>
  <level>WARNING</level>
  <class>Main</class>
  <method>main</method>
  <thread>1</thread>
  <message>warning</message>
</record>
</log>
```

控制台
```
十月 24, 2016 10:21:51 上午 Main main
警告: warning
```

常用的有两个Handler
- ConsoleHandler
- FileHandler

## 配置输出格式

```java
import java.io.IOException;
import java.util.logging.FileHandler;
import java.util.logging.Level;
import java.util.logging.Logger;
import java.util.logging.SimpleFormatter;

public class Main {
    private final static Logger LOGGER = Logger.getLogger(Main.class.getName());

    public static void main(String[] args) throws IOException {
        LOGGER.setLevel(Level.WARNING);
        FileHandler fileHandler = new FileHandler("log.txt");
        SimpleFormatter simpleFormatter = new SimpleFormatter();
        fileHandler.setFormatter(simpleFormatter);
        LOGGER.addHandler(fileHandler);
        LOGGER.info("info");
        LOGGER.warning("warning");
    }
}
```

log.txt

```
十月 24, 2016 10:27:16 上午 Main main
警告: warning
```

## 自定义格式

SimpleFormatter时JDK中自带的一种简单的格式，如果我们想自定义格式，可以继承Formatter类。

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.logging.Formatter;
import java.util.logging.Handler;
import java.util.logging.Level;
import java.util.logging.LogRecord;

// this custom formatter formats parts of a log record to a single line
class HtmlFormatter extends Formatter {
    // this method is called for every log records
    public String format(LogRecord rec) {
        StringBuffer buf = new StringBuffer(1000);
        buf.append("<tr>\n");

        // colorize any levels >= WARNING in red
        if (rec.getLevel().intValue() >= Level.WARNING.intValue()) {
            buf.append("\t<td style=\"color:red\">");
            buf.append("<b>");
            buf.append(rec.getLevel());
            buf.append("</b>");
        } else {
            buf.append("\t<td>");
            buf.append(rec.getLevel());
        }

        buf.append("</td>\n");
        buf.append("\t<td>");
        buf.append(calcDate(rec.getMillis()));
        buf.append("</td>\n");
        buf.append("\t<td>");
        buf.append(formatMessage(rec));
        buf.append("</td>\n");
        buf.append("</tr>\n");

        return buf.toString();
    }

    private String calcDate(long millisecs) {
        SimpleDateFormat date_format = new SimpleDateFormat("MMM dd,yyyy HH:mm");
        Date resultdate = new Date(millisecs);
        return date_format.format(resultdate);
    }

    // this method is called just after the handler using this
    // formatter is created
    public String getHead(Handler h) {
        return "<!DOCTYPE html>\n<head>\n<style>\n"
                + "table { width: 100% }\n"
                + "th { font:bold 10pt Tahoma; }\n"
                + "td { font:normal 10pt Tahoma; }\n"
                + "h1 {font:normal 11pt Tahoma;}\n"
                + "</style>\n"
                + "</head>\n"
                + "<body>\n"
                + "<h1>" + (new Date()) + "</h1>\n"
                + "<table border=\"0\" cellpadding=\"5\" cellspacing=\"3\">\n"
                + "<tr align=\"left\">\n"
                + "\t<th style=\"width:10%\">Loglevel</th>\n"
                + "\t<th style=\"width:15%\">Time</th>\n"
                + "\t<th style=\"width:75%\">Log Message</th>\n"
                + "</tr>\n";
    }

    // this method is called just after the handler using this
    // formatter is closed
    public String getTail(Handler h) {
        return "</table>\n</body>\n</html>";
    }
}
```

```java
import java.io.IOException;
import java.util.logging.FileHandler;
import java.util.logging.Level;
import java.util.logging.Logger;

public class Main {
    private final static Logger LOGGER = Logger.getLogger(Main.class.getName());

    public static void main(String[] args) throws IOException {
        LOGGER.setLevel(Level.ALL);
        FileHandler fileHandler = new FileHandler("log.html");
        HtmlFormatter htmlFormatter = new HtmlFormatter();
        fileHandler.setFormatter(htmlFormatter);
        LOGGER.addHandler(fileHandler);
        LOGGER.info("info");
        LOGGER.warning("warning");
        LOGGER.config("config");
    }
}
```

输出结果  
![](/images/4.png)
