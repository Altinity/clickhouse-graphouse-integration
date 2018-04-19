# clickhouse-graphouse-integration

# Table of Contents

 * [Introduction](#introduction---what-are-we-talking-about)
 * [Install ClickHouse](#install-clickhouse)
 * [Install Graphouse](#install-graphouse)
 * [Install Graphite-web](#install-graphite-web)
 * [Setup ClickHouse - Graphouse integration](#setup-clickhouse---graphouse-integration)
 * [Monitoring](#monitoring)



# Install ClickHouse
ClickHouse installation is explained in several sources, such as:
 * for [deb-based systems](https://clickhouse.yandex/docs/en/getting_started/#installing-from-packages-debianubuntu)
 * for [rpm-based systems](https://github.com/Altinity/clickhouse-rpm-install)


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

## Create tables
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


## Install Graphouse
```bash
sudo apt install graphouse
```

Edit properties in graphouse config `/etc/graphouse/graphouse.properties`

Setup ClickHouse access. Make sure `graphouse.clickhouse.host` is correct. WARNING `//` comments are erroneous ini properties file!
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

# Setup ClickHouse to report metrics into Graphouse.

This will provide ClickHouse's self-monitoring, to some extent - ClickHouse will report own metrics, which will be kept back in ClickHouse via Graphouse.

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

Now ClickHouse would report metrics to Graphouse, and they will be stored back insode ClickHouse.
So, we can check metrics data coming:

```bash
 clickhouse-client -q "select count(*) from graphite.data"
 clickhouse-client -q "select count(*) from graphite.metrics"
```

# Install Graphite-web

## Install graphite-web.

You don't need carbon or whisper, Graphouse and ClickHouse completely replace them.

```bash
sudo apt install -y python3-pip
sudo apt install -y python3-dev libcairo2-dev libffi-dev build-essential
export PYTHONPATH="/opt/graphite/lib/:/opt/graphite/webapp/"
sudo pip3 install --no-binary=:all: https://github.com/graphite-project/graphite-web/tarball/master

sudo apt install -y gunicorn3
sudo apt install -y nginx

sudo touch /var/log/nginx/graphite.access.log
sudo touch /var/log/nginx/graphite.error.log
sudo chmod 640 /var/log/nginx/graphite.*
sudo chown www-data:www-data /var/log/nginx/graphite.*
```

Setup host name in nginx config
```bash
sudo vim /etc/nginx/sites-available/graphite
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
cp graphite.wsgi.example graphite.wsgi
gunicorn3 --bind=127.0.0.1:8080 graphite.wsgi:application
```

Point your browser to the host, where Graphite-web is running:

![Graphite screenshot](images/graphite_web_graphouse.png?raw=true "Graphite screenshot")

