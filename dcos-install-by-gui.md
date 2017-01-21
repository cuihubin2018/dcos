### 通过GUI安装DC/OS

**注意：** 当前GUI安装模式仅适合用于初步体验，其后续的升级过程非常困难，因此，建议在生产环境部署时通过**高级安装**模式。

#### 1.DC/OS安装文件下载就绪后，在Bootstrap节点上执行如下命令启动安装：

```
$ sudo bash dcos_generate_config.sh --web
```

如果想查看调试输出，可以添加-v参数

```
$ sudo bash dcos_generate_config.sh --web -v
```

#### 2.安装程序启动后，终端会有类似如下日志显示：

```
Running mesosphere/dcos-genconf docker with BUILD_DIR set to /home/centos/genconf

16:36:09 dcos_installer.action_lib.prettyprint:: ====&gt; Starting DC/OS installer in web mode

16:36:09 root:: Starting server ('0.0.0.0', 9000)
```

#### 3.通过浏览器访问DC/OS安装界面：**`http://<bootstrap-node-public-ip>:9000`**

#### 4.点击“**Begin Installation**”开始安装


### FAQ

如果在安装过程中出现问题，查看错误日志，解决问题，删除`/opt/dcos_install_tmp`文件后重新执行，直至安装完成。
