# KafkaBrokerMonitorPrometheusGrafana

A simple example of monitoring a single Kafka broker with Prometheus and Grafana

**Prereqs**

This assumes you already possess some know-how in AWS (SSH into boxes, create or use an appropriate VPC,create or use an appropriate security group) as well as run some basic linux commands. You probably would not be here if that was foreign.

You can use whatever boxes and O/S you want.

In this example, we will use a CentOS box in AWS for Kafka and Prometheus and Grafana using the convenient Confluent CLI for illustration.

**Install Java, Confluent**

Install Java

```
sudo yum install java-1.8.0-openjdk
```

Install CP

```
curl -O http://packages.confluent.io/archive/5.1/confluent-5.1.2-2.11.tar.gz
tar -xvf confluent-5.1.2-2.11.tar.gz
```

Install Prometheus JMX Exporter Agent on Kafka broker

```
sudo yum install wget
mkdir prometheus
cd prometheus
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.3.1/jmx_prometheus_javaagent-0.3.1.jar
wget https://raw.githubusercontent.com/prometheus/jmx_exporter/master/example_configs/kafka-0-8-2.yml
cd ..
```

Start Kafka

```
/home/centos/confluent-5.1.2/bin
./confluent start zookeeper
KAFKA_OPTS="-javaagent:/home/centos/prometheus/jmx_prometheus_javaagent-0.3.1.jar=8080:/home/centos/prometheus/kafka-2_0_0.yml" ./confluent start kafka
```

You should see metrics if you curl port 8080.
Those metrics are being exported by the JMX exporter.

```
curl localhost:8080
```

Install Prometheus and have it pick up those JMX metrics.

```
cd
mv prometheus prometheusbackup
wget https://github.com/prometheus/prometheus/releases/download/v2.8.0/prometheus-2.8.0.linux-amd64.tar.gz
tar -xzf prometheus-*.tar.gz
rm prometheus-2.8.0.linux-amd64.tar.gz
mv prometheus-2.8.0.linux-amd64 prometheus
cp prometheusbackup/kafka-0-8-2.yml prometheus/
cp prometheusbackup/kafka-2_0_0.yml prometheus/
cp prometheusbackup/jmx_prometheus_javaagent-0.3.1.jar prometheus/
rm -rf prometheusbackup/
cd prometheus
sudo vi prometheus.yml

global:
 scrape_interval: 10s
 evaluation_interval: 10s
scrape_configs:
 - job_name: 'kafka'
   static_configs:
    - targets:
      - localhost:8080
```

Start Prometheus

```
./prometheus
```

Open Prometheus in your browser (see Prereqs on AWS - hint: security groups)

```
http://<your EC2 server>:9090/graph
```

Let's run it as a service

```
sudo vi /etc/systemd/system/prometheus.service

# /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=centos
ExecStart=/home/centos/prometheus/prometheus --config.file=/home/centos/prometheus/prometheus.yml --storage.tsdb.path=/home/centos/prometheus/data

[Install]
WantedBy=multi-user.target

sudo systemctl start prometheus
sudo systemctl status prometheus
```

Open Prometheus in your browser (see Prereqs on AWS - hint: security groups)

```
http://<your EC2 server>:9090/graph
```

Now let's get Grafana working

```
cd
wget https://dl.grafana.com/oss/release/grafana-6.0.1.linux-amd64.tar.gz
tar -zxvf grafana-6.0.1.linux-amd64.tar.gz
rm grafana-*.gz
mv grafana-*/ grafana
cd grafana
```

Let's change some of the anonymous settings in the defaults config file.
This is not something we do for Production, but for this simple local test it's fine.
```
sudo vi conf/defaults.ini
# enable anonymous access
enabled = true
# specify role for unauthenticated users
org_role = Admin
```

Now start Grafana to make sure it works
```
bin/grafana-server
```

Open Grafana in your browser (see Prereqs on AWS - hint: security groups)

```
http://<your EC2 server>:3000/graph
```

Let's run it as a service

```
sudo vi /etc/systemd/system/grafana.service

# /etc/systemd/system/grafana.service
[Unit]
Description=Grafana Server
After=network-online.target

[Service]
User=centos
WorkingDirectory=/home/centos/grafana
ExecStart=/home/centos/grafana/bin/grafana-server

[Install]
WantedBy=multi-user.target

sudo systemctl start grafana
sudo systemctl status grafana
```

Let's setup a dashboard in Grafana. This one is really old, but still useful as a starting point.


Step 1. Add a new datasource in Grafana
- call it "Prometheus Localhost" with URL http://localhost:9090

Step 2. Add a new dashboard in Grafana
- Import https://grafana.com/dashboards/721 and use the new datasource you just created in Step 1

Produce a little data and refresh your browser.

```
./kafka-topics --zookeeper localhost:2181 --topic dummy_topic --create --replication-factor 1 --partitions 1
./kafka-producer-perf-test --topic dummy_topic --num-records 10000000 --record-size 1 --throughput 1 --producer-props bootstrap.servers=localhost:9092
^C
./kafka-producer-perf-test --topic dummy_topic --num-records 10000000 --record-size 100 --throughput 100 --producer-props bootstrap.servers=localhost:9092
```

**Now let's get some inspiration from Confluent!**

This will give you inspiration on the kinds of metrics you want to include on not just the brokers, but Connect, Schema Registry and more. This is actually an open source dashboard for another project so it won't work out of the box for you. However, this is going to provide you with inspiration.

Open this in your browser:
https://raw.githubusercontent.com/confluentinc/cp-helm-charts/master/grafana-dashboard/confluent-open-source-grafana-dashboard.json


Copy the Raw data.

In Grafana, create a new dashboard, but this time instead of using a URL, paste in the JSON you copied from that git file.

Again, you will probably have either dummy data (like 29 brokers) or a N/A. That's ok because again you are going to use this for inspiration on what you can set up! You are going to use these single stats, graphs, tiles and charts to customize to your setup.

Example:
Open Confluent Kafka -> Brokers Online and Edit that single stat tile:
The current value is:

```
count(cp_kafka_server_replicamanager_leadercount{release=~"$Release"})
```

After you look in Prometheus for "leadercount", you find it says:

```
kafka_server_replicamanager_leadercount
```

So replace the value with:

```
count(kafka_server_replicamanager_leadercount)
```

And voila, your number should show up as 1 broker, 3 brokers or however many you have.

Rinse and repeat for all those other tiles!

Have fun!
