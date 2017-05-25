## HAProxy配置参考

https://github.com/mesosphere/marathon-lb/wiki#enabling-haproxy-logging

- Enabling HAProxy logging

```json
{
  "id": "/marathon-lb",
  "container": {
    "type": "DOCKER",
    "volumes": [
      {
        "containerPath": "/dev/log",
        "hostPath": "/dev/log",
        "mode": "RW"
      }
    ],
    "docker": {
      "image": "mesosphere/marathon-lb:latest",
      "network": "HOST",
      "privileged": true,
      "parameters": [],
      "forcePullImage": true
    }
  }
}
```
Now, we'll set option httplog on one backend to enable logging. In this example, I'm using my personal website:

```json
{
  "id": "/my-crappy-website",
  "cmd": null,
  "cpus": 0.5,
  "mem": 64,
  "disk": 0,
  "instances": 2,
  "container": {
    "type": "DOCKER",
    "volumes": [],
    "docker": {
      "image": "brndnmtthws/my-crappy-website",
      "network": "BRIDGE",
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 0,
          "servicePort": 10012,
          "protocol": "tcp",
          "labels": {}
        }
      ],
      "privileged": false,
      "parameters": [],
      "forcePullImage": true
    }
  },
  "healthChecks": [
    {
      "path": "/",
      "protocol": "HTTP",
      "portIndex": 0,
      "gracePeriodSeconds": 10,
      "intervalSeconds": 15,
      "timeoutSeconds": 2,
      "maxConsecutiveFailures": 3,
      "ignoreHttp1xx": false
    }
  ],
  "labels": {
    "HAPROXY_GROUP": "external",
    "HAPROXY_0_BACKEND_HTTP_OPTIONS": "  option httplog\n  option forwardfor\n  http-request set-header X-Forwarded-Port %[dst_port]\n  http-request add-header X-Forwarded-Proto https if { ssl_fc }\n"
  },
  "portDefinitions": [
    {
      "port": 10012,
      "protocol": "tcp",
      "labels": {}
    }
  ]
}
```

Note: Enabling the httplog option will only affect the backend for the service port. To enable it for ports 80 and 443, you must modify the global HAProxy template.

Now, if you SSH into any public slave, you can view the logs using journalctl:

```
journalctl -f -l SYSLOG_IDENTIFIER=haproxy
```

- 查看HAProxy的负载端口列表

 `http://[PUBLIC_NODE_IP]:9090/haproxy?stats`


- 获取HAProxy的配置

 `http://[PUBLIC_NODE_IP]:9090/_haproxy_getconfig`


### 参考

https:\/\/github.com\/mesosphere\/marathon-lb\/blob\/master\/Longhelp.md

