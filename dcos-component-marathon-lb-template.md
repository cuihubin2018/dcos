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

- 查看HAProxy的负载端口列表

 `http://[PUBLIC_NODE_IP]:9090/haproxy?stats`


- 获取HAProxy的配置

 `http://[PUBLIC_NODE_IP]:9090/_haproxy_getconfig`


### 参考

https:\/\/github.com\/mesosphere\/marathon-lb\/blob\/master\/Longhelp.md

