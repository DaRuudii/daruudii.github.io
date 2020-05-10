---
layout: single
classes: wide
title:  "Choosing the correct syslog format for QRadar"
date:   2020-05-10 16:00:00 +0200
category: Log Source Configuration
tags: [qradar, linux, syslog]
---

When sending events from a Linux system to QRadar one must configure a syslog daemon to send the locally written logs to the QRadar component which accepts events (console, event collector or event processor). On most Linux systems there may already be such daemon installed. The most popular ones are:

* rsyslog
* syslog-ng

In this blog post the environment uses the following systems and software:

__QRadar 7.3.3 Patch 3__ \
IP: 192.168.0.20 \
hostname: qradar733

__CentOS 8.1.1911__ \
IP: 192.168.0.42 \
hostname: vctest01 \
with rsyslog 8.37.0

# The basic configuration

For sending events to remote systems like QRadar it is best to create a new file in the rsyslog configuration directory of the Linux system. In my case I created the file `/etc/rsyslog.d/qradar.conf` as it indicates events are sent to QRadar. Note that such configuration files must always end with `.conf` to be activated when rsyslog is started.

```
authpriv.*;*.err      @@192.168.0.20:514
```

With this configuration all events from the syslog facility `authpriv` and error messages from all syslog facilities are send to the given IP address via TCP with the default forwarding format of rsyslog. If you want to send via UDP only use one `@`.

After some authentication actions (`su -` and a `logout`) have been performed on the Linux system events have been send to QRadar. Unfortunately, as seen in the screenshot from QRadar the logs have not been assigned to a log source as no log source has been created. Often log sources are auto discovered in QRadar but this depends heavily on the log source type and the incoming logs.

![Screenshot of the so far unknown logs](/assets/img/2020-05-10-syslog-format-qradar/qradar-simgeneric.png)

After the log source has been created with the hostname of the CentOS system as identifier the same commands lead to correct events in QRadar.

![Screenshot of the parsed logs](/assets/img/2020-05-10-syslog-format-qradar/qradar-logs.png)

At this point one could already be satisfied with the result. But in my opinion, there can be done more.

# Tuning the configuration

If we look closely on the current payload format of the events, we can see some potential.

```
<86>May  9 19:24:53 vctest01 su[489]: pam_unix(su-l:session): session opened for user root by root(uid=0)
```

The current timestamp is in the format `MMM d HH:mm:ss`. We have no information about the current year, the millisecond or in general at what time the event was generated because we have no information about the time zone where the system is located and we have to assume the given time is the systems local time.

To change the format of the timestamp or the format of the whole event we must specify a template which should be used for sending the events.

The default template is considerably basic and does not use a specific format for the timestamp.

```
template(name="RSYSLOG_TraditionalFileFormat" type="string"
    string="%TIMESTAMP% %HOSTNAME% %syslogtag%%msg:::sp-if-no-1st-sp%%msg:::drop-last-lf%\n"
```

To get a more specific timestamp format we can use one of the built-in templates which does specify which format is used for timestamps.

```
template(name="RSYSLOG_SyslogProtocol23Format" type="string"
    string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n")
```

The template which should be used when sending events with rsyslog can be specified in the same location where we specify what should be sent. One just must add the name of the template at the end of that line.

```
authpriv.*;*.err    @@192.168.0.20:514;RSYSLOG_SyslogProtocol23Format
```

With this format we can get the following payload from the same action we performed previously.

```
<86>1 2020-05-09T19:58:38.224749+02:00 vctest01 su 10077 - - pam_unix(su-l:session): session opened for user root by root(uid=0)
```

We now have a more specific timestamp which includes all information like the year, milliseconds and the time zone. Additionally, this format supports other improvements like structured data, which is not used in this example.

This new format can also lead to some events being mapped to other QIDs in QRadar. In my experience the new QIDs were more specific regarding the event where in the basic format the event name often is something generic.

# Implementing a heartbeat

Often when sending events to a SIEM like QRadar one wants to monitor if the sending system is still active. Depending on the configuration of the types of events a system will send, that system can be quiet for days before sending another event. In the meantime, while it does not send anything one cannot be sure if the system is still active or not.

For this case there are MARK messages. These are messages which are sent regularly if no other events are sent. These events can be used as a kind of heartbeat.

To enable MARK messages, edit the general rsyslog configuration file under `/etc/rsyslog.conf` and un-comment the line where the `immark` module is loaded and modify it with the correct interval.

```
module(load="immark" interval="60") # provides --MARK-- message capability
```

If the module is enabled, it can be used to forward MARK messages to QRadar. Add the following line to `/etc/rsyslog.d/qradar.conf`.

```
:msg, contains, "MARK"  @@192.168.0.20:514;RSYSLOG_SyslogProtocol23Format
```

In QRadar the MARK messages have no specific QID so the events are displayed as `Linux login messages Message`. Which is one of the generic fallback events for Linux log sources if no other QID can be mapped.

![Screenshot of the MARK events in QRadar](/assets/img/2020-05-10-syslog-format-qradar/qradar-mark.png)

# Hostname or IP address?

One common question is whether to put the hostname or the IP address of a system in its syslog header. Per default it's always the hostname which makes sense as it makes it easy to identify the system from an event.

But there is one downside to having the hostname in the syslog header: relaying syslog servers.

When events are sent via an additional syslog server or a log management solution and the syslog header only has the hostname it can lead to false positives in the worst case. This is because of how QRadar normalizes incoming events.

Given the following payload we have no IP addresses which could populate the source and destination IP fields in QRadar.

```
<86>1 2020-05-09T22:00:56.750253+02:00 vctest01 su 29186 - - pam_unix(su-l:session): session opened for user root by root(uid=0)
```

In that case QRadar uses the source IP address from the network packet in which the payload was sent.

![Screenshot of the IP information of an event in QRadar](/assets/img/2020-05-10-syslog-format-qradar/qradar-ipinfo.png)

In my case the IP address in QRadar is the correct one because the system is sending its logs directly to QRadar. In the case of an additional syslog server between log source and QRadar the IP address shown in QRadar would be the one of the syslog server.

Depending on the use cases and rules this can lead to false positives when the wrong IP address matches.

To circumvent this case the rsyslog template on the Linux system can be modified to include the IP address instead of the hostname. In that case QRadar has an IP address it can assign the normalized fields if there is no other IP address in the rest of the payload.

For this to work a new template must be created in the `/etc/rsyslog.conf` file. This should be added to the file at the top.

```
template(name="QRadarForwardFormat" type="string"
     string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% 192.168.0.42 %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n")
```

This template is the same as the `RSYSLOG_SyslogProtocol23Format` except it features the IP address instead the hostname. The IP address must be "hardcoded" into the template on each system, because there is no working parameter for the IP address.

Then the template must be used in the `/etc/rsyslog.d/qradar.conf` file.

```config
:msg, contains, "MARK"  @@192.168.0.20:514;QRadarForwardFormat
authpriv.*;*.err        @@192.168.0.20:514;QRadarForwardFormat
```

The log source in QRadar has to be updated to then use the IP address instead of the hostname as the log source identifier.

This modification can also be done on the additional syslog server if the software used on it supports modification of the payload.
