%ELK and Graphite installation and configuration

In this section, we install and configure all components of the log and metrics management
solution itself: Logstash, Elasticsearch, Kibana, Graphite and Grafana.

# Logstash and Elasticsearch base configuration

## Logstash installation

Install java-1.7.0-openjdk first, then use the RPM package from the Elasticsearch website:
<https://download.elasticsearch.org/logstash/logstash/packages/centos/logstash-1.4.2-1_2c0f5a1.noarch.rpm>
The checksum is here <https://download.elasticsearch.org/logstash/logstash/packages/centos/logstash-1.4.2-1_2c0f5a1.noarch.rpm.sha1.txt>

    yum install logstash-1.4.2-1_2c0f5a1.noarch.rpm

Create a test configuration for Logstash in
`/etc/logstash/conf.d/openshift.conf` with the following content:

    input {
        file {
          type => "syslog"
          path => "/var/log/rsyslog/*"
          exclude => "*.gz"
        }
    }
    filter {
        grok {
            match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: \[?(%{LOGLEVEL}\s)?\]?\s*%{GREEDYDATA:syslog_message}" }
            add_field => [ "received_at", "%{@timestamp}" ]
            add_field => [ "received_from", "%{host}" ]
        }
        syslog_pri { }
        date {
            match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
    }
    output {
        stdout {
            codec => "rubydebug"
        }
        elasticsearch {
            protocol => "http"
            cluster => "openshift"
        }
    }

Validate the configuration with

    /opt/logstash/bin/logstash -f /etc/logstash/conf.d/openshift.conf --configtest

It's good practice to validate the logstash configuration every time you make a
change to it.

## Elasticsearch installation

Use the RPM package from the Elasticsearch website:
<https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.3.4.noarch.rpm>
The checksum is here <https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.3.4.noarch.rpm.sha1.txt>

    yum install elasticsearch-1.3.4.noarch.rpm

You can also use the upstream yum repository, it doesn't include Logstash or Kibana at the moment though.

Change the Elasticsearch cluster name in `/etc/elasticsearch/elasticsearch.yaml`

    name: openshift

We use openshift as our cluster name as an example, it's good practice to
choose a unique name for your Elasticsearch cluster because nodes will autojoin
a cluster with the same name. *If you use another cluster name, make sure you
use it in `/etc/logstash/conf.d/openshift.conf` too.*

Start and enable the elasticsearch service, test that it is working with

    curl localhost:9200/

Communication with Elasticsearch is done through the REST API.

## Log processing

Start and enable the Logstash service. To be sure that it's working, follow the
output log:

    tail -f /var/log/logstash/logstash.stdout

This output log is enabled in `/etc/logstash/conf.d/openshift.conf` by

        stdout {
            codec => "rubydebug"
        }

in the `output` section, it should be commented out or removed for normal
operations.

If Logstash's Elasticsearch output is working, it should create an index for
each day with a name like `logstash-YYY.MM.DD`. List Elasticsearch indexes with

    curl localhost:9200/_cat/indices?v

To delete test indexes and remove Logstash's logfile position tracker, stop the
logstash service and use

    curl -XDELETE 'http://localhost:9200/logstash-YYY-MM-DD
    rm /var/lib/logstash/.sincedb_*

# Kibana

Kibana is a browser based Javascript application, it is accessed through a web
server. Apache httpd is used to serve Kibana files and as a reverse proxy for
Elasticsearch.

## Apache httpd configuration

Apache httpd is installed from standard repositories.

    yum install -y httpd

Several components are accessed through TCP port 80 on the server using named virtual hosts
for Kibana, Elasticsearch, etc.

Enable named virtual hosts in `/etc/httpd/conf/httpd.conf` by uncommenting the following line

    NameVirtualHost *:80

Configure a virtual host for Elasticsearch and Kibana by creating
`/etc/httpd/conf.d/kibana.conf` with the following content

    <VirtualHost *:80>
      ServerName kibana.example.com
      ServerAlias es.example.com
    
      DocumentRoot /var/www/kibana
      <Directory /var/www/kibana>
        Allow from all
        Options -Multiviews
      </Directory>
    
      # Set global proxy timeouts
      <Proxy http://127.0.0.1:9200>
        ProxySet connectiontimeout=5 timeout=90
      </Proxy>
    
      # Proxy for _aliases and .*/_search
      <LocationMatch "^/(_nodes|_aliases|.*/_aliases|_search|.*/_search|_mapping|.*/_mapping)$">
        ProxyPassMatch http://127.0.0.1:9200/$1
        ProxyPassReverse http://127.0.0.1:9200/$1
      </LocationMatch>
    
      # Proxy for kibana-int/{dashboard,temp} and grafana-dash/dashboard
      # To save Kibana and Grafana dashboards in Elasticsearch
      <LocationMatch "^/(kibana-int/dashboard/|grafana-dash/dashboard/|kibana-int/temp)(.*)$">
        ProxyPassMatch http://127.0.0.1:9200/$1$2
        ProxyPassReverse http://127.0.0.1:9200/$1$2
      </LocationMatch>
    </VirtualHost>

Start and enable the `httpd` service.

Because we are using httpd as a reverse proxy, we must set an SELinux boolean to
allow it to connect to the network

    setsebool -P httpd_can_network_connect 1

## Kibana installation

There's no package for Kibana, it's just a tarball. Download version 3.0.1 from
the upstream website, *we don't use the last version, it doesn't work with the
current version of Logstash*
<https://download.elasticsearch.org/kibana/kibana/kibana-3.0.1.tar.gz>

Extract the dowloaded tarball and move its content to `/var/www/kibana`

    tar xvf kibana-3.0.1.tar.gz
    mkdir /var/www/kibana
    mv kibana-3.0.1/* /var/www/kibana/

You may have to label the files as httpd content to make SELinux happy:

    chcon -Rv --type=httpd_sys_content_t /var/www/kibana

and make the security context change permanent with

    semanage fcontext -a -t httpd_sys_content_t "/var/www/kibana(/.*)?" 

Kibana uses Elasticsearch as a backend, change its configuration to use the
correct Elasticsearch URL in `/var/www/kibana/config.js`

    // elasticsearch: "http://"+window.location.hostname+":9200",
    elasticsearch: "http://es.example.com",

We've defined a proxy for Elasticsearch in Apache httpd's configuration, so we
don't use ElasticSearch's port 9200.

Check that everything works by browsing to <http://kibana.example.com> and clicking
the link to the default Logstasg dashboard. It should display the data from
the logs that have been processed.

# Graphite and Grafana installation

Graphite and its dependencies are installed from EPEL. For more information
about enabling the EPEL repository for your system go to
<https://fedoraproject.org/wiki/EPEL>.

If you can't/won't install EPEL, you can alternatively download the individual
packages from, eg. <http://dl.fedoraproject.org/pub/epel/6/x86_64/>, and
install them. The list of package is:

    graphite-web-0.9.12-5.el6.noarch.rpm
    graphite-web-selinux-0.9.12-5.el6.noarch.rpm
    pyparsing-1.5.6-1.el6.noarch.rpm
    python-carbon-0.9.12-3.el6.noarch.rpm
    python-whisper-0.9.12-1.el6.noarch.rpm
    Django14-1.4.14-1.el6.noarch.rpm
    django-tagging-0.3.1-3.el6.noarch.rpm

If you are installing from the downloaded packages, install them all together
using yum. If you are using the EPEL repository, install with

    yum install graphite-web graphite-web-selinux python-carbon python-whisper

Create the initial Graphite database. You may want to consider using another
database than the local sqlite DB used here as an example.

    python /usr/lib/python2.6/site-packages/graphite/manage.py syncdb

Start and enable the `carbon-cache` and `memcached` services.

Change the timezone setting in
`/usr/lib/python-2.6/site-packages/graphite/local_settings.py` to the timezone of
your environment, eg. Europe/London.

Give ownership of the sqlite database to the Apache user

    chown apache:apache /var/lib/graphite-web/graphite.db

Configure a virtual host for Graphite by creating
`/etc/httpd/conf.d/graphite-web.conf` with the following content

    # Graphite Web Basic mod_wsgi vhost
    <VirtualHost *:80>
        ServerName graphite.example.com
        DocumentRoot "/usr/share/graphite/webapp"
        ErrorLog /var/log/httpd/graphite-web-error.log
        CustomLog /var/log/httpd/graphite-web-access.log common
        Alias /media/ "/usr/lib/python2.6/site-packages/django/contrib/admin/media/"
    
        WSGIScriptAlias / /usr/share/graphite/graphite-web.wsgi
        WSGIImportScript /usr/share/graphite/graphite-web.wsgi process-group=%{GLOBAL} application-group=%{GLOBAL} 
    
        <Location "/content/">
            SetHandler None
        </Location>
    
        <Location "/media/">
            SetHandler None
        </Location>
    
    </VirtualHost>

Restart the `httpd` service.

Check that Graphite is accessible at <http://graphite.example.com>

Test that graphite can receive data by running the test script and checking the
metrics appear in the Web UI

    python /usr/share/doc/graphite-web-0.9.12/example-client.py

## Grafana installation

Grafana is based on Kibana, the installation and configuration is very similar.
Download the tarball from the upstream website at <http://grafana.org/>

    wget http://grafanarel.s3.amazonaws.com/grafana-1.8.1.tar.gz

Extract the dowloaded tarball and move its content to `/var/www/grafana`

    tar xvf grafana-1.8.1.tar.gz
    mkdir /var/www/grafana
    mv grafana-1.8.1/* /var/www/grafana/

Again, you may have to label the files as httpd content to make SELinux happy:

    chcon -Rv --type=httpd_sys_content_t /var/www/grafana

and make the security context change permanent with

    semanage fcontext -a -t httpd_sys_content_t "/var/www/grafana(/.*)?" 

Change the configuration to use Graphite and Elasticsearch (Elasticsearch is
used to save the dashboards) in `/var/www/grafana/config.js`

        // Graphite & Elasticsearch example setup
        datasources: {
          graphite: {
            type: 'graphite',
            url: "http://graphite.example.com",
          },
          elasticsearch: {
            type: 'elasticsearch',
            url: "http://es.example.com",
            index: 'grafana-dash',
            grafanaDB: true,
          }
        },

Configure a virtual host for Grafana by creating
`/etc/httpd/conf.d/grafana.conf` with the following content

    <VirtualHost *:80>
      ServerName grafana.example.com
    
      DocumentRoot /var/www/grafana
      <Directory /var/www/grafana>
        Allow from all
        Options -Multiviews
      </Directory>
    </VirtualHost>

To allow Grafana to access Graphite, we need to enable CORS
(Cross Origin Resource Sharing) in the Graphite virtual host.
Add the following lines to `/etc/httpd/conf.d/graphite-web.conf`

    # Set headers for CORS
    Header set Access-Control-Allow-Origin "http://grafana.example.com"
    Header set Access-Control-Allow-Methods "GET, OPTIONS"
    Header set Access-Control-Allow-Headers "origin, authorization, accept"

Restart the `httpd` service.

Test that Grafana works and metrics are accessible at
<http://grafana.example.com>

# Logstash final configuration

Now that all the components are working and configured, we can configure Logstash to parse logs and metrics.

Copy the included `openshift.conf` file to `/etc/logstash/conf.d`, check that
it doesn't contain any error and restart the logstash service.

# Conclusion

The full solution is now deployed. The next step is to do some more useful
parsing of logs (metrics should work as-is) and design dashboards for different
type of users.

