%Log forwarding and centralisation configuration

In this section, we install a (network listening) rsyslog server on the Logstash server,
configure all components of OpenShift to use syslog, and syslog to push to the
rsyslog server.

# Rsyslog server

We use the default RHEL 6 rsyslog package on logstash.example.com, no software installation is
necessary.

## System configuration

We store logs in `/var/log/rsyslog/` and use one file per program (same file for
all hosts). The `/var/log/rsyslog` directory must be created first and readable by
logstash (eg. `chmod 0755`).

The host firewall must also be configured to allow both UDP and TCP port 514, for example with:

    # iptables -A INPUT -m tcp -p tcp --dport 514 -j ACCEPT
    # iptables -A INPUT -m udp -p udp --dport 514 -j ACCEPT
    # service iptables save

## Rsyslog configuration

Rsyslog is configured by editing `/etc/rsyslog.conf`.

Enable network listening for the server by uncommenting the following lines:

    # Provides UDP syslog reception
    $ModLoad imudp
    $UDPServerRun 514
    
    # Provides TCP syslog reception
    $ModLoad imtcp
    $InputTCPServerRun 514

Add a rule to centralise all non-local logs to `/var/log/rsyslog/` using one
file per originator program for all hosts:

    # change default umask so that logstash can read files
    
    $umask 0000
    $DirCreateMode 0755
    $FileCreateMode 0644
    
    $template TmplMsg, "/var/log/rsyslog/%PROGRAMNAME%.log"
    :fromhost-ip,!isequal,"127.0.0.1" -?TmplMsg
    & ~

Restart the `rsyslog` service.

## Log rotation

You may want to add a specific rule for log rotation in `/var/log/rsyslog`, for
example by creating the `/etc/logrotate.d/syslog-server` file with the following
content:

    /var/log/rsyslog/*.log {
        daily
        rotate 30
        copytruncate
        compress
        missingok
        notifyempty
    }

# General configuration of OpenShift servers

All OpenShift servers (brokers, broker support nodes, nodes) are configured to
forward system logs to the logstash.example.com. We deploy a newer version of
rsyslog from the OpenShift repository to allow additional metadata in the log
files provided by a plugin.

## Rsyslog7 installation and configuration

    yum install -y rsyslog7 rsyslog7-mmopenshift

Important! For OpenShift Enterprise 2.2: OpenShift Enterprise 2.2 runs on
RHEL 6.6, which comes with its own package for rsyslog7. Because the RHEL
rsyslog7 is a replacement for the default rsyslog package, you’ll need to
use yum shell to run a transaction:

    yum shell
    erase rsyslog
    install rsyslog7 rsyslog7-mmopenshift
    transaction run

The rsyslog7 configuration will be in `/etc/rsyslog7.conf` for OpenShift
Enterprise 2.1 and `/etc/rsyslog.conf` starting with OpenShift Enterprise 2.2.

Disable Systemd-specific options by commenting out the following lines:

    # $ModLoad imjournal # provides access to the systemd journal

    # Turn off message reception via local log socket;
    # local messages are retrieved through imjournal now.
    #$OmitLocalLogging on
    
    # File to store the position in the journal
    #$IMJournalStateFile imjournal.state

Make sure rsyslog7 is loading additional configuration files from
`/etc/rsyslog.d/*.conf`

    $IncludeConfig /etc/rsyslog.d/*.conf

*On OpenShift nodes only*, enable the rsyslog plugin by changing the modules section:

    #$ModLoad imuxsock # provides support for local system logging (e.g. via logger command)
    # $ModLoad imjournal # provides access to the systemd journal
    $ModLoad imklog   # provides kernel logging support (previously done by rklogd)
    #$ModLoad immark  # provides --MARK-- message capability
    # OpenShift plugin module
    module(load="imuxsock" SysSock.Annotate="on" SysSock.ParseTrusted="on" SysSock.UsePIDFromSystem="on")
    module(load="mmopenshift")

Add a forwarding rule by creating `/etc/rsyslog7.d/forward.conf` with the following content:

    $WorkDirectory /var/lib/rsyslog
    $ActionQueueFileName fwdRule1 # unique name prefix for spool files
    $ActionQueueMaxDiskSpace 1g   # 1gb space limit (use as much as possible)
    $ActionQueueSaveOnShutdown on # save messages to disk on shutdown
    $ActionQueueType LinkedList   # run asynchronously
    $ActionResumeRetryCount -1    # infinite retries if host is down
    *.* @@logstash.example.com:514

*On OpenShift nodes only*, create a template for the plugin by creating
`/etc/rsyslog7.d/openshift.conf` with the following content:

    # OpenShift plugin configuration
    template(name="OpenShift" type="list")
      {
      property(name="timestamp" dateFormat="rfc3339")
      constant(value=" ")
      property(name="hostname")
      constant(value=" ")
      property(name="syslogtag")
      constant(value=" app=")
      property(name="$!OpenShift!OPENSHIFT_APP_NAME")
      constant(value=" ns=")
      property(name="$!OpenShift!OPENSHIFT_NAMESPACE")
      constant(value=" appUuid=")
      property(name="$!OpenShift!OPENSHIFT_APP_UUID")
      constant(value=" gearUuid=")
      property(name="$!OpenShift!OPENSHIFT_GEAR_UUID")
      property(name="msg" spifno1stsp="on")
      property(name="msg" droplastlf="on")
      constant(value="\n")
    }

## Replacing Rsyslog with Rsyslog7

On OpenShift Enterprise 2.1, the rsyslog and rsyslog7 packages cohexist,
you need to stop and disable the default rsyslog service on all OpenShift
servers to allow rsyslog7 to replace it.

    # service rsyslog stop
    # chkconfig rsyslog off
    # service rsyslog7 start
    # chkconfig rsyslog7 on

Starting with OpenShift 2.2, the rsyslog7 package replaces the default rsyslog,
that is why we had to do this `yum shell` dance earlier. Just ensure that the
rsyslog service is enabled and running.

    # service rsyslog start
    # chkconfig rsyslog on

# Configuring OpenShift brokers to use syslog

## The Broker application

Add the following line to `/etc/openshift/broker.conf`

    SYSLOG_ENABLED="true"

Restart the `openshift-broker` service.

## The Console application

Add the following line to `/etc/openshift/console.conf`

    SYSLOG_ENABLED="true"

Restart the `openshift-console` service.

# Configuring  broker support nodes to use syslog

Depending on your OpenShift environment, these services may run on the broker
itself (or brokers) or on separate so-called broker support nodes.

## MongoDB

In `/etc/mongodb.conf` comment out the logpath parameter and enable syslog

    #logpath = /var/log/mongodb/mongodb.log
    syslog = true

Restart the `mongod` service.

## ActiveMQ

ActiveMQ uses Log4j for logging, it must be configured in
`/etc/activemq/log4j.properties` by changing the `rootLogger` parameter and
configuring a `SYSLOG` appender

    #log4j.rootLogger=INFO, logfile
    log4j.rootLogger=INFO, logfile, SYSLOG

    # Syslog appender
    log4j.appender.SYSLOG = org.apache.log4j.net.SyslogAppender
    log4j.appender.SYSLOG.syslogHost = 172.28.151.90:514
    log4j.appender.SYSLOG.layout=org.apache.log4j.PatternLayout
    log4j.appender.SYSLOG.layout.ConversionPattern=%d{MMM dd HH:mm:ss} activemq: %-5p %m%n
    log4j.appender.SYSLOG.Facility = LOCAL0

Restart the `activemq` service.

# Configuring OpenShift nodes to use syslog

## Configuring Apache httpd to use syslog and annotate log messages

Add the following options to `/etc/sysconfig/httpd`

    OPTIONS="-DOpenShiftFrontendSyslogEnabled -DOpenShiftAnnotateFrontendAccessLog"

Restart the `httpd` service.

## Node application and logshifter

Add the following parameters to `/etc/openshift/node.conf`

    # Enable logging to syslog with annotations
    PLATFORM_LOG_CONTEXT_ENABLED=1
    PLATFORM_LOG_CONTEXT_ATTRS=request_id,container_uuid,app_uuid
    PLATFORM_LOG_CLASS=SyslogLogger
    # Enable Metrics
    WATCHMAN_METRICS_ENABLED=true
    # How often should the watchman plugin gather metrics
    # WATCHMAN_METRICS_INTERVAL=60
    # Metadata to include in messages
    METRICS_METADATA="appName:OPENSHIFT_APP_NAME,gear:OPENSHIFT_GEAR_UUID,app:OPENSHIFT_APP_UUID,ns:OPENSHIFT_NAMESPACE"

Change following parameter in `/etc/openshift/logshifter.conf`

    outputtype = syslog

Restart the `ruby193-mcollective` and `openshift-watchman` services

## Application logging/metrics

Logshifter is now configured to output cartridges and applications logs/metrics
to syslog, but all existing applications have to be restarted to use the new
configuration.

# Conclusion

At this stage, we have a centralised logging server using rsyslog, and all components of the OpenShift environment are configured to use syslog for logging. All OpenShift logs go to syslog, and syslog ships everything to logstash.example.com, including logging by applications that were already using syslog (sshd, cron...).

There should be one .log file for each application in /var/log/rsyslog.
