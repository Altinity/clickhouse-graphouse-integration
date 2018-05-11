# clickhouse-graphouse-integration

# Table of Contents

 * [Introduction](#introduction)
   * [Graphite](#graphite)
   * [Graphouse](#graphouse)
   * [Solution architecture](#solution-architecture)
 * [Install ClickHouse](#install-clickhouse)
 * [Configure ClickHouse](#configure-clickhouse)
   * [Create ClickHouse tables](#create-clickhouse-tables)
 * [Install Graphouse](#install-graphouse)
 * [Configure Graphouse](#configure-graphouse)
 * [Intermediate results](#intermediate-results)
 * [Setup ClickHouse to report metrics into Graphouse](#setup-clickhouse-to-report-metrics-into-graphouse)
 * [Install Graphite-web](#install-graphite-web)
 * [Monitoring](#monitoring)
 * [Consclusions](#conclusions)

# Introduction

Monitoring is an important part of operating any software in production. There are many monitoring tools available, and [Graphite](http://graphite.readthedocs.io/en/latest/overview.html) is a popular one.

## Graphite

Graphite is positioned as an enterprise-scale monitoring tool that runs well on cheap hardware, and has gained significant popularity lately.
Graphite consists of 3 software components:
 * `carbon` - a daemon that listens for data
 * `whisper` - a database layer for storing data
 * `graphite webapp` - a webapp that renders graphs

All is going well with Graphite, until incoming data stream gets big. What exactly **big** means is really task-specific, but it turned out, that having particular threshold stepped over, storage layer, which is `carbon` + `whisper`, may perform not as well as it could be, presenting the following issues:
 * Lack of replication possibilitites
 * Lack of data consistency
 * High disk IO utilization.
 * High disk space utilization

Let's walk over these issues in more details

### Lack of replication possibilities

In case you'd like to introduce failover capabilities to monitoring system, you'd need to have metrics data replicated (stored in more than one place).
That's introduce some kind of an issue with **Graphite**, since we do not know better way than to just duplicate data stream to each instance of `carbon + whisper` running on different servers.
However, the main inconvenience comes when one of those replicas fails, and gets a gap in data. The simplest way to fill the gap would be with `rsync` and requires additional attention/automation from admins.

### Lack of data consistency

Another not-so-peasant issue is that sometimes, datasets on those replicas are not equal. Not much, typically just several percents, but nevertheless, this brings some feeling of uncertainty to the picture. 
Not necessarily `carbon` + `whisper` are in charge of this issue, most likely the setup in general - with duplicated data streams - but it would be nice to have solution out-of-the-box, and not invent it.

### High disk IO utilization

Having incoming data stream overstepped some threshold, `carbon + whisper` consumes disk IO heavily. Under some circumstances there is even 100% disk IO utilization.

### High disk space utilization

Sometimes, you need to write only several metrics during retention period (this is especially the case, when you write not only hardware/software metrics, but business metrics as well) but `.wsp` file is created by data storage layer for the whole retention period.
In case of multiple situations of this sort, disk space utilization can be not that good at all.


Of course, each of this issues is not a huge problem per se, but the more data you have, the more you'd like to have these issues to be dealt with. And some alternatives are looked for.
And here it is.

## Graphouse

[Graphouse](https://github.com/yandex/graphouse/) allows you to use [ClickHouse](https://clickhouse.yandex/) as a [Graphite](http://graphite.readthedocs.io/en/latest/overview.html) storage.

What we'd like to achieve with such a shift:
 * Get replication capabilities and data consistency out-of-the-box - provided by DBMS itself
 * Lower disk IO utilization
 * Lower disk space utilization

## Solution architecture

As described earlier, **Graphite** consists of 3 software components
 * `carbon` - a daemon that listens for data
 * `whisper` - a database layer for storing data
 * `graphite webapp` - a webapp that renders graphs

**Graphouse** substitutes `carbon` and `whisper` components, presenting it's own data accumulation daemon and **ClickHouse** as a data storage layer. 
Ultimately, you can replace `graphite webapp` with other graphing tool as well, having no **Graphite** components left.

So let's walk over the whole **Graphouse** and **ClickHouse** installation and setup procedures. We need to:
 * Install ClickHouse (it would be used as a data storage layer)
 * Install Graphouse (it would be used as a metrics processing layer)
 * Setup Graphouse - ClickHouse integration

As an additional option, we'd like to:
 * Setup ClickHouse to report its own metrics into Graphouse - to be the first to be monitored by Graphouse.

Also we'd most likely need to have some king of GUI to take a look at metrics, so we'd
 * Setup Graphite-web as a graphing tool

Let's go

# Install ClickHouse

Most likely you already have ClickHouse installed, in this case just move to [Configure ClickHouse](#configure-clickhouse) section.
ClickHouse installation is explained in several sources, such as:
 * for [deb-based systems](https://clickhouse.yandex/docs/en/getting_started/#installing-from-packages-debianubuntu)
 * for [rpm-based systems](https://github.com/Altinity/clickhouse-rpm-install)


# Configure ClickHouse

Let's configure ClickHouse to be a data storage for Graphouse.

Create rollup config file `/etc/clickhouse-server/conf.d/graphite_rollup.xml`.
Settings for thinning data for Graphite.

```bash
sudo mkdir -p /etc/clickhouse-server/conf.d
```
`/etc/clickhouse-server/conf.d/graphite_rollup.xml` can be get [here](conf/graphite_rollup.xml?raw=true)
```xml
<yandex>
<graphite_rollup>
    <path_column_name>metric</path_column_name>
    <time_column_name>timestamp</time_column_name>
    <value_column_name>value</value_column_name>
    <version_column_name>updated</version_column_name>
	<pattern>
		<regexp>^five_sec</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>5</precision>
		</retention>
		<retention>
			<age>2592000</age>
			<precision>60</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>600</precision>
		</retention>
	</pattern>

	<pattern>
		<regexp>^one_min</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>60</precision>
		</retention>
		<retention>
			<age>2592000</age>
			<precision>300</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>1800</precision>
		</retention>
	</pattern>

	<pattern>
		<regexp>^five_min</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>300</precision>
		</retention>
		<retention>
			<age>2592000</age>
			<precision>600</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>1800</precision>
		</retention>
	</pattern>

	<pattern>
		<regexp>^one_sec</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>1</precision>
		</retention>
		<retention>
			<age>2592000</age>
			<precision>60</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>300</precision>
		</retention>
	</pattern>

	<pattern>
		<regexp>^one_hour</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>3600</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>86400</precision>
		</retention>
	</pattern>

	<pattern>
		<regexp>^ten_min</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>600</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>3600</precision>
		</retention>
	</pattern>

	<pattern>
		<regexp>^one_day</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>86400</precision>
		</retention>
	</pattern>

	<pattern>
		<regexp>^half_hour</regexp>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>1800</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>3600</precision>
		</retention>
	</pattern>

	<default>
		<function>any</function>
		<retention>
			<age>0</age>
			<precision>60</precision>
		</retention>
		<retention>
			<age>2592000</age>
			<precision>300</precision>
		</retention>
		<retention>
			<age>31104000</age>
			<precision>1800</precision>
		</retention>
	</default>
</graphite_rollup>
</yandex>
```

## Create ClickHouse tables

Restart ClickHouse, run `clickhouse-client` and create tables. SQL file is available [here](conf/create_tables.sql?raw=true)

```bash
sudo /etc/init.d/clickhouse-server restart
clickhouse-client
```

```sql
CREATE DATABASE graphite;

CREATE TABLE graphite.metrics(
    date Date DEFAULT toDate(0),
    name String,
    level UInt16,
    parent String,
    updated DateTime DEFAULT now(),
    status Enum8(
        'SIMPLE' = 0,
        'BAN' = 1,
        'APPROVED' = 2,
        'HIDDEN' = 3,
        'AUTO_HIDDEN' = 4
    )
) ENGINE = ReplacingMergeTree(date, (parent, name), 1024, updated);

CREATE TABLE graphite.data(
    metric String,
    value Float64,
    timestamp UInt32,
    date Date,
    updated UInt32
) ENGINE = GraphiteMergeTree(date, (metric, timestamp), 8192, 'graphite_rollup');
```


This engine is designed for rollup (thinning and aggregating/averaging) Graphite data. It may be helpful to developers who want to use ClickHouse as a data store for Graphite.

Graphite stores full data in ClickHouse, and data can be retrieved in the following ways:

 * without thinning - with MergeTree engine.
 * with thinning - with GraphiteMergeTree engine.

Settings for thinning data are defined by the `graphite_rollup` parameter in the server configuration.

More details are available in  [official doc](https://clickhouse.yandex/docs/en/table_engines/graphitemergetree/#table_engines-graphitemergetree)


# Install Graphouse

## Add Graphouse debian repo.

In `/etc/apt/sources.list` (or in a separate file, like `/etc/apt/sources.list.d/graphouse.list`), add repository: 
`deb http://repo.yandex.ru/graphouse/xenial stable main`. On other versions of Ubuntu, replace `xenial` with your version.
Such as:
```bash
sudo bash -c 'echo "deb http://repo.yandex.ru/graphouse/xenial stable main" >> /etc/apt/sources.list'
sudo apt update
```

## Install JDK8.

```bash
sudo add-apt-repository ppa:webupd8team/java
sudo apt update
sudo apt install -y oracle-java8-installer
```
Package `oracle-java8-installer` contains a script to install Java.

Set Java 8 as your default Java version.
```bash
sudo apt install -y oracle-java8-set-default
```

Let’s verify the installed version.
```bash
java -version 

java version "1.8.0_171"
Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)
```

## Setup JAVA_HOME and JRE_HOME Variable

You must set `JAVA_HOME` and `JRE_HOME` environment variables, which is used by many Java applications to find Java libraries during runtime. 
Set these variables in `/etc/environment` file:
```bash
sudo bash -c 'cat >> /etc/environment <<EOL
JAVA_HOME=/usr/lib/jvm/java-8-oracle
JRE_HOME=/usr/lib/jvm/java-8-oracle/jre
EOL'
```

## And install Graphouse

```bash
sudo apt install graphouse
```

# Configure Graphouse

Let's configure Graphouse to store metrics in ClickHouse

Edit properties in graphouse config `/etc/graphouse/graphouse.properties`

Setup ClickHouse access. Make sure `graphouse.clickhouse.host` is correct. **WARNING** `//` comments are erroneous ini properties file!
```ini
graphouse.clickhouse.host=localhost
graphouse.clickhouse.hosts=${graphouse.clickhouse.host}
graphouse.clickhouse.port=8123
graphouse.clickhouse.db=graphite
graphouse.clickhouse.user=
graphouse.clickhouse.password=
```

Set `graphouse.clickhouse.retention-config` to `graphite_rollup` as we earlier defined in
```xml
<yandex>
  <graphite_rollup>
    ...
  </graphite_rollup>
</yandex>
```
Set
```ini
graphouse.clickhouse.retention-config=graphite_rollup
```
Config name for `graphouse.clickhouse.retention-config` is not a file path for `/etc/clickhouse-server/conf.d/graphite_rollup.xml`. You should use one of names from ClickHouse `system.graphite_retentions` table.

Start graphouse
```bash
sudo /etc/init.d/graphouse start
```

# Intermediate results

At this point we have **Graphouse** up and running and ready to accept metrics from entities to be monitored.
Next steps would be to:
 * Setup metrics delivery from monitored entities
 * Setup graph-capable GUI

Generally speaking, you can choose any compatible GUI, but from the sake of simplicity we'll use Graphite's webapp.
Let's go on and setup ClickHouse to write its own metrics into Graphouse

# Setup ClickHouse to report metrics into Graphouse

This will provide ClickHouse's self-monitoring, to some extent - ClickHouse will report own metrics, which will be kept back in ClickHouse via Graphouse.
This part is optional, in case you'd like to monitor ClickHouse by other means, or do not monitor ClickHouse at all, just skip this section.

Edit `/etc/clickhouse-server/config.xml` and append something like the following:
```xml
    <graphite>
        <host>127.0.0.1</host>
        <port>2003</port>
        <timeout>0.1</timeout>
        <interval>60</interval>
        <root_path>one_min_cr_plain</root_path>

        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
    </graphite>
    <graphite>
        <host>127.0.0.1</host>
        <port>2003</port>
        <timeout>0.1</timeout>
        <interval>1</interval>
        <root_path>one_sec_cr_plain</root_path>

        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>false</asynchronous_metrics>
    </graphite>
```
Settings description:

 * `host` – host where Graphite is running.
 * `port` – plain text receiver port (2003 is default).
 * `interval` – interval for sending data from ClickHouse, in seconds.
 * `timeout` – timeout for sending data, in seconds.
 * `root_path` – prefix used by Graphite.
 * `metrics` – should data from system_tables-system.metrics table be sent.
 * `events` – should data from system_tables-system.events table be sent.
 * `asynchronous_metrics` – should data from system_tables-system.asynchronous_metrics table be sent.

Multiple `<graphite>` clauses can be configured for sending different data at different intervals.

Restart ClickHouse
```bash
sudo /etc/init.d/clickhouse-server restart
```

Now ClickHouse would report metrics to Graphouse, and they will be stored back inside ClickHouse.
So, we can check metrics are coming:

```bash
 clickhouse-client -q "select count(*) from graphite.data"
 clickhouse-client -q "select count(*) from graphite.metrics"
```

As soon as we see metrics coming, we are ready to move to next step - metrics visualisation. This can be done via any Graphite-compatible interface and we'll use Graphite's webapp, just as an example


# Install Graphite-web

## Install graphite-web.

You don't need `carbon` or `whisper` components of Graphite, Graphouse and ClickHouse completely replace them.
Graphite has detailed [installation docs](http://graphite.readthedocs.io/en/latest/install-pip.html) and [configuration docs](http://graphite.readthedocs.io/en/latest/install.html#initial-configuration)


```bash
sudo apt install -y python3-pip python3-dev libcairo2-dev libffi-dev build-essential
export PYTHONPATH="/opt/graphite/lib/:/opt/graphite/webapp/"
sudo pip3 install --no-binary=:all: https://github.com/graphite-project/graphite-web/tarball/master

sudo apt install -y gunicorn3
sudo apt install -y nginx

sudo touch /var/log/nginx/graphite.access.log
sudo touch /var/log/nginx/graphite.error.log
sudo chmod 640 /var/log/nginx/graphite.*
sudo chown www-data:www-data /var/log/nginx/graphite.*
```
Create `/etc/nginx/sites-available/graphite` ngix config file with the following content:
Write the following configuration in `/etc/nginx/sites-available/graphite` (availabe as a [file](conf/nginx_graphite?raw=true))
```ini

upstream graphite {
    server 127.0.0.1:8080 fail_timeout=0;
}

server {
    listen 80 default_server;

    server_name HOSTNAME;

    root /opt/graphite/webapp;

    access_log /var/log/nginx/graphite.access.log;
    error_log  /var/log/nginx/graphite.error.log;

    location = /favicon.ico {
        return 204;
    }

    # serve static content from the "content" directory
    location /static {
        alias /opt/graphite/webapp/content;
        expires max;
    }

    location / {
        try_files $uri @graphite;
    }

    location @graphite {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_connect_timeout 10;
        proxy_read_timeout 10;
        proxy_pass http://graphite;
    }
}

```

Setup `server_name` host name in nginx config
```bash
sudo vim /etc/nginx/sites-available/graphite
```

Enable configuration for nginx:
```bash
sudo ln -s /etc/nginx/sites-available/graphite /etc/nginx/sites-enabled
sudo rm -f /etc/nginx/sites-enabled/default
```

And reload nginx to use new configuration:
```bash
sudo service nginx reload
```

## Setup Graphite-web DB

We need to create the database tables used by the graphite webapp.

```bash
sudo bash -c 'PYTHONPATH=/opt/graphite/webapp django-admin.py migrate --settings=graphite.settings --run-syncdb'
```

Ensure database created:

```bash
 ls -l /opt/graphite/storage/graphite.db
-rw-r--r-- 1 root root 98304 Apr 19 22:49 /opt/graphite/storage/graphite.db
```

If your webapp is running as the ‘nobody’ user, you will need to fix the permissions like this:

```bash
sudo chown nobody:nogroup /opt/graphite/storage/graphite.db
```

## Setup connection to Graphouse

Add graphouse plugin `/opt/graphouse/bin/graphouse.py `to your graphite webapp root dir. For example, if you dir is `/opt/graphite/webapp/graphite/` use:
```bash
sudo ln -fs /opt/graphouse/bin/graphouse.py /opt/graphite/webapp/graphite/graphouse.py
```

Configure storage finder in your `/opt/graphite/webapp/graphite/local_settings.py`
Open `/opt/graphite/webapp/graphite/local_settings.py` in editor and add:

```python
STORAGE_FINDERS = (
    'graphite.graphouse.GraphouseFinder',
)
```

Start Graphite-web
```bash
cd /opt/graphite/conf
sudo cp graphite.wsgi.example graphite.wsgi
sudo bash -c 'export PYTHONPATH="/opt/graphite/lib/:/opt/graphite/webapp/"; gunicorn3 --bind=127.0.0.1:8080 graphite.wsgi:application'
```

# Monitoring

Point your browser to the host, where Graphite-web is running:

![Graphite screenshot](images/graphite_web_graphouse.png?raw=true "Graphite screenshot")

Now, we have monitoring tool availbale.

# Conclusions

At this moment, we have Graphouse + ClickHouse as a data storage layer installed and setup. From here we can move multiple ways, for example:
 * Setup ClickHouse replication cluster, or
 * Setup different graphing tool (like, say Grafana) in case you'd prefer,

but this are topics for another post.

