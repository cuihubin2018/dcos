## 日志归集

**注意**，默认情况下，使用Docker引擎运行容器镜像时，是没有存储资源限制的。而且Docker的容器日志默认使用**json-file**驱动，如果不加以限定可能会耗尽宿主机的存储资源。

ELK提供了一组日志采集、清洗、存储、分析和展示的完整服务生态。通过ELK可以将DC/OS集群中各节点上的系统（节点操作系统及服务日志，DC/OS服务组件日志）及应用（容器、容器内应用等）日志进行归集分析处理。

下述通过`Filebeat -> ElasticSearch -> Kibana`展示如何将DC/OS集群节点上的日志进行归集展示。

### 安装配置Filebeat

**范围：所有节点（Masters和Agents）**

1. 安装Filebeat：

 ```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.0.0-x86_64.rpm
sudo rpm -vi filebeat-5.0.0-x86_64.rpm
```

2. 创建`/var/log/dcos`目录
 ```
 sudo mkdir -p /var/log/dcos
 ```

3. 归档Filebeat的默认配置
 ```
 sudo mv /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.BAK
 ```

4. 创建一个新的`filebeat.yml`配置文件，添加一条指向`/var/log/dcos/dcos.log`文件的新记录，该文件(后续步骤使用）用于归集DC/OS集群中的日志：

 ```
  sudo tee /etc/filebeat/filebeat.yml <<-EOF 
 filebeat.prospectors:
 - input_type: log
   paths:
     - /var/lib/mesos/slave/slaves/*/frameworks/*/executors/*/runs/latest/stdout
     - /var/lib/mesos/slave/slaves/*/frameworks/*/executors/*/runs/latest/stderr
     - /var/log/mesos/*.log
     - /var/log/dcos/dcos.log
 tail_files: true
 output.elasticsearch:
   hosts: ["$ELK_HOSTNAME:$ELK_PORT"]
 EOF
 ```
 
 **注意：需要将`$ELK_HOSTNAME`和`$ELK_PORT`替换为实际Elastic配置**

### 归集Master节点上的日志

**范围：所有Master节点**

创建一个systemd服务，该服务解析DC/OS集群通过journalctl输出的日志并将其导向`/var/log/dcos/dcos/dcos.log`。

```
sudo tee /etc/systemd/system/dcos-journalctl-filebeat.service<<-EOF 
[Unit]
Description=DCOS journalctl parser to filebeat
Wants=filebeat.service
After=filebeat.service

[Service]
Restart=always
RestartSec=5
ExecStart=/bin/sh -c '/usr/bin/journalctl --no-tail -f \
  -u dcos-3dt.service \
  -u dcos-3dt.socket \
  -u dcos-adminrouter-reload.service \
  -u dcos-adminrouter-reload.timer   \
  -u dcos-adminrouter.service        \
  -u dcos-bouncer.service            \
  -u dcos-ca.service                 \
  -u dcos-cfn-signal.service         \
  -u dcos-cosmos.service             \
  -u dcos-download.service           \
  -u dcos-epmd.service               \
  -u dcos-exhibitor.service          \
  -u dcos-gen-resolvconf.service     \
  -u dcos-gen-resolvconf.timer       \
  -u dcos-history.service            \
  -u dcos-link-env.service           \
  -u dcos-logrotate-master.timer     \
  -u dcos-marathon.service           \
  -u dcos-mesos-dns.service          \
  -u dcos-mesos-master.service       \
  -u dcos-metronome.service          \
  -u dcos-minuteman.service          \
  -u dcos-navstar.service            \
  -u dcos-networking_api.service     \
  -u dcos-secrets.service            \
  -u dcos-setup.service              \
  -u dcos-signal.service             \
  -u dcos-signal.timer               \
  -u dcos-spartan-watchdog.service   \
  -u dcos-spartan-watchdog.timer     \
  -u dcos-spartan.service            \
  -u dcos-vault.service              \
  -u dcos-logrotate-master.service  \
  > /var/log/dcos/dcos.log 2>&1'
ExecStartPre=/usr/bin/journalctl --vacuum-size=10M

[Install]
WantedBy=multi-user.target
EOF
```

### 归集Agent节点上的日志

**范围：所有Agent节点**

创建一个systemd服务，该服务解析DC/OS集群通过journalctl输出的日志并将其导向`/var/log/dcos/dcos/dcos.log`。

```
sudo tee /etc/systemd/system/dcos-journalctl-filebeat.service<<-EOF 
[Unit]
Description=DCOS journalctl parser to filebeat
Wants=filebeat.service
After=filebeat.service

[Service]
Restart=always
RestartSec=5
ExecStart=/bin/sh -c '/usr/bin/journalctl --no-tail -f      \
  -u dcos-3dt.service                      \
  -u dcos-logrotate-agent.timer            \
  -u dcos-3dt.socket                       \
  -u dcos-mesos-slave.service              \
  -u dcos-adminrouter-agent.service        \
  -u dcos-minuteman.service                \
  -u dcos-adminrouter-reload.service       \
  -u dcos-navstar.service                  \
  -u dcos-adminrouter-reload.timer         \
  -u dcos-rexray.service                   \
  -u dcos-cfn-signal.service               \
  -u dcos-setup.service                    \
  -u dcos-download.service                 \
  -u dcos-signal.timer                     \
  -u dcos-epmd.service                     \
  -u dcos-spartan-watchdog.service         \
  -u dcos-gen-resolvconf.service           \
  -u dcos-spartan-watchdog.timer           \
  -u dcos-gen-resolvconf.timer             \
  -u dcos-spartan.service                  \
  -u dcos-link-env.service                 \
  -u dcos-vol-discovery-priv-agent.service \
  -u dcos-logrotate-agent.service          \
  > /var/log/dcos/dcos.log 2>&1'
ExecStartPre=/usr/bin/journalctl --vacuum-size=10M

[Install]
WantedBy=multi-user.target
EOF
```

### 启用日志归集和Filebeat服务

**范围：所有节点**

```
sudo chmod 0755 /etc/systemd/system/dcos-journalctl-filebeat.service
sudo systemctl daemon-reload
sudo systemctl start dcos-journalctl-filebeat.service
sudo chkconfig dcos-journalctl-filebeat.service on
sudo systemctl start filebeat
sudo chkconfig filebeat on
```

### 使用Logstash解析DC/OS集群日志

1. 在`$PATTERNS_DIR`目录下创建dcos模式匹配文件：
 ```
 PATHELEM [^/]+
 TASKPATH ^/var/lib/mesos/slave/slaves/%{PATHELEM:agent}/frameworks/%{PATHELEM:framework}/executors/%{PATHELEM:executor}/runs/%{PATHELEM:run}
 ```

2. 修改Logstash的配置包含下述`grok`过滤：
 ```
 filter {
     grok {
         patterns_dir => "$PATTERNS_DIR"
         match => { "file" => "%{TASKPATH}" }
     }
 }
 ```
 
 **注：$PATTERNS_DIR需要替换为实际路径**

3. 重启Logstash服务

 通过Kibana使用`framework:*`作为查询条件，过滤查询效果类似下图：

 ![](/assets/logstash-framework-exists.png)

### 注意事项

1. 本示例通过Filebeat直接将日志导向ElasticSearch服务器，如果需要在导向Elastic之前需要对日志进行解析过滤，可用Logstash替代Filebeat。

2. 示例中Agent节点上的Filebeat默认配置仅采集正在运行的Task的`stdout`和`stderr`的日志输出。如果Task的日志输出不是`stdout`和`stderr`，如Cassandra和Kafka，需要对Filebeat的日志配置进行调整。