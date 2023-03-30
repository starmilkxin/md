# jenkins持续集成与自动化部署

通过 jenkins持续集成与自动化部署 实现代码 push 后，能够自动打包分发到指定服务器并进行部署。

## 主要流程

我是采用的 jar包 部署：

1. 开发人员将代码 push 到代码库中。
2. jenkins 通过触发器(手动、定期、钩子函数、远程构建等...)得到通知，拉取源码。
3. jenkins 通过 maven 对源码进行打包。
4. 将 jenkins 服务器上的指定文件（jar包）分发到指定服务器上。
5. jenkins 通过 ssh，令接受的服务器执行 jenkins 提前编写好的 shell 脚本（暂停并删除原容器，根据 jar包 重新构建项目镜像，创建并运行容器）。

当项目服务器有很多时，jar包部署所消耗的带宽及耗时很多，最好是采用上传镜像到 Harbor 部署（仅需将 jar包 分发到 Harbor 中即可）：

1. 开发人员将代码 push 到代码库中。
2. jenkins 通过触发器(手动、定期、钩子函数、远程构建等...)得到通知，拉取源码。
3. jenkins 通过 maven 进行打包。
4. jenkins 服务器上的指定文件（jar包）分发到 Harbor上。
5. jenkins 通过 ssh，令 Harbor 对 jar包 构建镜像，并推送到仓库中。
6. jenkins 通过 ssh，令指定服务器暂停并删除原容器，拉取 Harbor 中新的项目镜像，创建并运行容器。

## 步骤

安装 jenkins 对应插件

- 新手入门的所有插件
- Maven Integration ：Maven 项目打包工具
- Publish Over SSH ：SSH 发布工具

添加凭据

- 在 Security 的 Manage Credentials 中添加 github 凭据 和 云服务器凭据。

![添加凭据](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/凭据.jpg)

系统配置

- 在 System Configuration 的 System 中设置 Jenkins Location 和 SSH Servers。

![系统配置](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/系统设置.png)

全局工具配置

- 在 System Configuration 的 Tools 中设置 JDK、Maven、NodeJS 等（可填写 jenkins 所在服务器中工具的绝对路径）。

新建 maven 项目

- 配置 git 仓库地址、凭据、分支。
- 构建触发器
  - 不配置：每次手动点击执行任务。
  - Build whenever a SNAPSHOT dependency is built：此项目所依赖的快照有更新则会触发更新此job。
  - 触发远程构建 (例如,使用脚本)：通过一个网址的访问来触发构建（http://localhost:8848/job/FlashRegistration/build??token=口令&cause=书写构建原因、http://localhost:8848/job//buildWithParameters?token=123456&cause=书写构建原因）。
  - Build after other projects are built：其他项目构建成功、失败、或者不稳定的时候触发项目。
  - Build periodically： 定期构建，不管版本库代码是否发生变化。
  - GitHub hook trigger for GITScm polling：通过 GitHub 的钩子函数，当代码 push 后，Github 调用钩子函数将源码传给指定的服务器（填 jenkins 服务器地址，因为我的jenkins服务器在内网中，所以无法使用该种方式）。
  - Poll SCM：按照设定的时间规则，先比较一次源代码是否发生变更，如果发生变更则构建。

构建环境

- 选好 jdk 的版本。

Build

- 因为需要对项目打包，所以要选择对象pom.xml以及要执行的命令。

![build](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/build.png)

Post Steps

- 选择需要 SSH 的服务器。（可将 Verbose output in console 勾选，输出详细日志）。
- Transfers
  - Transfer Set Source files：需要传送的资源在工作空间中的路径。
  - Remove prefix：因为 jenkins 是把上方的资源路径完整传送到其他服务器中，如果仅需要传送资源而非多余目录，则依靠此配置路径。
  - Remote directory：指定服务器的目标目录。
  - Exec command：传送完资源后需要执行的脚本。

为了能够有效的排错，可以开启服务端console的详细输出：

![verboseoutput](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/verboseoutput.jpg)

工作空间：

![工作空间](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/工作空间.png)

Post Steps：

![sshcommands](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/sshcommands.png)

立即构建

- 点击 立即构建 结束。
