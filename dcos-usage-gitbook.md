## 自动发布Gitbook文档

使用Gitbook Editor编写Markdown文档非常方便，而且Gitbook Editor可以直接加载Git仓库，文档的修改可以立即提交。

编写的Markdown文档可以通过`gitbook build`命令直接编译为静态页面，然后通过Nginx托管。

本文通过下述流程将整个编写及发布的过程自动化起来并最终部署到DC/OS中：

__
Gitbook Editor ---> Gitlab ---> Jenkins ---> Docker(Nginx Image) ---> DC/OS
__

### Gitbook Editor
可以从Gitbook [Editor官网](https://www.gitbook.com/editor)下载编辑器，支持Mac，Linux和Windows环境。



### Gitlab


### Jenkins


### Docker Image


### DC/OS部署
