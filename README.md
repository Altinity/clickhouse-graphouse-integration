# clickhouse-graphouse-integration

# Table of Contents

 * [Introduction](#introduction---what-are-we-talking-about)
 * [Install ClickHouse](#install-clickhouse)
 * [Install Graphouse](#install-graphouse)
 * [Install Graphite-web](#install-graphite-web)
 * [Setup ClickHouse - Graphouse integration](#setup-clickhouse---graphouse-integration)
 * [Monitoring](#monitoring)



# Install Clickhouse
ClickHouse installation is explained in several sources, such as:
 * for [deb-based systems](https://clickhouse.yandex/docs/en/getting_started/#installing-from-packages-debianubuntu)
 * for [rpm-based systems](https://github.com/Altinity/clickhouse-rpm-install)


reate rollup config /etc/clickhouse-server/conf.d/graphite_rollup.xml. Pay attention to graphite_rollup tag name. The name is used below.

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


Create tables

CREATE DATABASE graphite;

CREATE TABLE graphite.metrics ( date Date DEFAULT toDate(0),  name String,  level UInt16,  parent String,  updated DateTime DEFAULT now(),  status Enum8('SIMPLE' = 0, 'BAN' = 1, 'APPROVED' = 2, 'HIDDEN' = 3, 'AUTO_HIDDEN' = 4)) ENGINE = ReplacingMergeTree(date, (parent, name), 1024, updated);

CREATE TABLE graphite.data ( metric String,  value Float64,  timestamp UInt32,  date Date,  updated UInt32) ENGINE = GraphiteMergeTree(date, (metric, timestamp), 8192, 'graphite_rollup');



Graphouse

    Add Graphouse debian repo. In /etc/apt/sources.list (or in a separate /etc/apt/sources.list.d/graphouse.list file), add the repository: deb http://repo.yandex.ru/graphouse/trusty stable main. On other versions of Ubuntu, replace trusty with xenial or precise.
    Install JDK8.


Step 1 – Install Java 8 on Ubuntu

You need to enable additional repository to your system to install Java 8 on Ubuntu VPS. After that install Oracle Java 8 on an Ubuntu system using apt-get. This repository contains package named oracle-java8-installer, Which is not an actual Java package. Instead of that, this package contains a script to install Java on Ubuntu.

Run below commands to install Java 8 on Ubuntu and LinuxMint.

sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer

Step 2 – Verify Java Inatallation

The apt repository also provides package oracle-java8-set-default to set Java 8 as your default Java version. This package will be installed along with Java installation. To make sure run below command.

sudo apt-get install oracle-java8-set-default

After successfully installing Oracle Java 8 using the above steps, Let’s verify the installed version using the following command.

java -version 

java version "1.8.0_161"
Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)

Step 3 – Setup JAVA_HOME and JRE_HOME Variable

After installing Java on Linux system, You must have to set JAVA_HOME and JRE_HOME environment variables. Which is used by many Java applications to find Java libraries during runtime. You can set these variables in /etc/environment file using the following command.

cat >> /etc/environment <<EOL
JAVA_HOME=/usr/lib/jvm/java-8-oracle
JRE_HOME=/usr/lib/jvm/java-8-oracle/jre
EOL


    Install Graphouse sudo apt-get install graphouse
    Set graphouse.clickhouse.retention-config property in graphouse config /etc/graphouse/graphouse.properties. You can skip this step, then default config will be used.
    Start graphouse sudo /etc/init.d/graphouse start

If you have any problems check graphouse log dir for details /var/log/graphouse. See Configuration for more details.

Notice: Config name for graphouse.clickhouse.retention-config is not a file path, that you have copied /etc/clickhouse-server/conf.d/graphite_rollup.xml! You should use one of names from ClickHouse system.graphite_retentions table that you may retrieve with query:


Basically if you have used XML config from an example above, this name will be graphite_rollup, that defined inside <yandex></yandex> child elements:

<yandex>
  <graphite_rollup>
    ...
  </graphite_rollup>
</yandex>


Graphite-web

    Install graphite-web, if you don't have it already. You don't need carbon or whisper, Graphouse and ClickHouse completely replace them.



Install Graphite-web

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

sudo vim /etc/nginx/sites-available/graphite

cd /opt/graphite/conf
cp graphite.wsgi.example graphite.wsgi
gunicorn3 --bind=127.0.0.1:8080 graphite.wsgi:application


http://192.168.74.149/




vim /opt/graphite/webapp/graphite/local_settings.py


    Add graphouse plugin /opt/graphouse/bin/graphouse.py to your graphite webapp root dir. For example, if you dir is /opt/graphite/webapp/graphite/ use command below

sudo ln -fs /opt/graphouse/bin/graphouse.py /opt/graphite/webapp/graphite/graphouse.py

    Configure storage finder in your local_settings.py

STORAGE_FINDERS = (
    'graphite.graphouse.GraphouseFinder',
)

    Restart graphite-web




