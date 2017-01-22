## FAQ

### DCOS集群的DNS域名服务器变更后，如何重新配置DCOS集群？

修改 `/opt/mesosphere/etc/dns_config` （最多只支持[__3个DNS地址__](https://docs.mesosphere.com/1.8/administration/installing/custom/configuration-parameters/#resolvers)）

重启 `dcos-spartan.service`

在所有节点上重复上述步骤



### DC/OS系统出现问题的排查思路

0. 首先确保`ip-detect`脚本的正确性并且在`config.yaml`中配置的DNS解析正常工作。安装过程中出现的问题有95%的占比是因为这两个配置项存在问题而导致的。

可以在集群所有节点上手动执行`ip-detect`脚本或者在已安装了DC/OS的节点上通过`/opt/mesosphere/bin/detect_ip`命令检查返回的IP地址是否有效，例如，返回结果不应含有额外新的行，不能含有空格，不能含有其它特殊的或不可见的字符。



Please try to copy or base your `_`ip-detect`_` on the examples listed in `_[_**`https:\/\/dcos.io\/docs\/1.8\/administration\/installing\/custom\/advanced\/`**_](https://dcos.io/docs/1.8/administration/installing/custom/advanced/)_` w.r.t DNS resolution: _Ideally`_`, forward `_`and`_` reverse lookups for FQDNs, Short Hostnames and IP addresses should work, but we can run DC\/OS in environments without valid DNS support but the following `_`must`_` work (esp. for Spark and some other frameworks), regardless: hostname -f `_`must`_` return the FQDN hostname -s `_`must`_` return the Short Hostname Check the output of hostnamectl for sanity on all your nodes as well
In general, this is the sequence in which we will have to troubleshoot:`_`(Exhibitor -> Mesos Master -> Mesos-DNS -> Spartan -> {Marathon, Metronome, Adminrouter})`_` and checking that all services are up and healthy on the Masters before moving to the Agents`

1.Ensure that firewalls or whatever connection-filtering mechanism is in use is not interfering with cluster component communications. TCP, UDP _and_ ICMP need to be permitted.

2.Check to see if Exhibitor has come up and has settled `http://<PICK_A_MASTER_IP>:8181/exhibitor otherwise check the Exhibitor service logs: journalctl -flu dcos-exhibitorPlease ensure that /tmp is mounted `_`without`_` noexec otherwise Exhibitor will fail to bring up Zookeeper because Java JNI won't be able to exec a file it creates in /tmp with permission denied errors in the Exhibitor service log file: journalctl -flu dcos-exhibitorTo fix: mount -o remount,exec /tmpCheck the output of /exhibitor/v1/cluster/status and ensure that it has the correct number of masters that you expect and that `_`all of them`_` are "serving" `_`but only one`_` of them has "isLeader": true`

e.g., While SSH'ed into a Master node:

```
curl -fsSL http://localhost:8181/exhibitor/v1/cluster/status | python -m json.tool
[ 
    { "code": 3, "description": "serving", "hostname": "10.0.6.70", "isLeader": false }, 
    { "code": 3, "description": "serving", "hostname": "10.0.6.69", "isLeader": false }, 
    { "code": 3, "description": "serving", "hostname": "10.0.6.68", "isLeader": true }
]
```

Note: You _may_ have to wait a while \(~10-15 minutes\) for this to happen in multi-master setups. If it doesn't converge after this much time has elapsed you may need to look through `journalctl -flu dcos-exhibitor logs with a fine-toothed comb.`

3.Assuming Exhibitor is up, check if you can ping `ready.spartan otherwise check the Spartan service logs: journalctl -flu dcos-spartan`

4.Assuming Spartan is up and you can ping `ready.spartan check if you can ping leader.mesos and master.mesos otherwise check the Mesos-DNS service logs: journalctl -flu dcos-mesos-dns`

5.If you can ping `ready.spartan but not leader.mesos you may also have to check the Mesos master(s) service logs: journalctl -flu dcos-mesos-master because the Mesos master needs to be up and a leader must be elected before Mesos-DNS can generate its DNS records from /state`





