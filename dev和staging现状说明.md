# dev和staging现状说明

标签（空格分隔）： 工作经验

---

## 老版本情况说明

* 老版本没有 staging 环境，在 dev(154) 上测试合适之后，将模块通过 ftp 移动到阿里云。
* 没有专门的静态资源服务器，所有的静态资源，放置在该项目主 service 的 src/main/webpack/frontend 目录下。
* server 层，通过 eclipse 打包，后生成运行文件。
* 前端在 local 环境下进行 build，生成 dist 文件夹，提交到 svn 上，再将文件夹复制到 主项目的静态资源文件夹下面。

如下所示：

项目 | 前端静态资源 svn 路径 | service 的静态资源目录
-- | --
金融数据平台 | /pycf/ProjectsFE/py_platform/trunk | PyPlatform\src\main\webapp\frontend
金融数据平台补充 | /pycf/ProjectsFE/py_platform_others/trunk/dist | PyTrustEvents\src\main\webapp\frontend\dist
自定义监测 | /pycf/ProjectsFE/py_platform_custom_monitor/branches/style_change/dist | PyJianCe\src\main\webapp\dist
银行画像 | /pycf/ProjectsFE/py_platform_huaxiang_system/branches/style_change/dist | \py_huaxiang_system\src\main\webapp\frontend

而后端的服务，则是有一群，一个项目中每个大板块，做成的一个 service。而需要注意的是，由于依赖的是 eclipse 进行打包，需要注意他们的数据库请求的地址，在 dev 和 staging 环境下是内部测试库，而 production 需要是正式库。这一点的打包方式需要静波给到方案。

## 现在新的方式

* 新的 java 和 python 的后台，直接上传源码到 svn，然后在 dev 和 staging 环境中 svn checkout，然后 `maven install` 运行起来
* 前端代码，ignore 掉 dist 里的内容。上传源码到 svn，在 dev 和 staging 环境中 svn checkout，然后 nvm 管理具体的 node.js 的版本，运行 `cnpm install` 然后再运行 `npm run build`，已生成各个项目中的 dist 的代码。
* 在 dev 环境，为了让 server 组可以直接在本机进行调试，所以要配 ngnix，根据端口，将静态文件请求转发到 /static 上，而将其它的接口转发到对应后台开发人员的电脑上。这一点给 server 同学们承诺了很久还没有实现。。。
* dev 环境，同时作为，前端的 api 接口请求的地址。前端开发人员在其 local 环境上跑起来的开发服务器，向 dev 环境进行 proxy
* staging 环境，要暴露测试的页面给测试人员，也要暴露后台 service 的端口和 path 给到他们。用作各种测试。
* 在 staging 上的代码，经过各种测试之后，通过文件 sftp 或者其它方式，直接镜像式地复制到 production 环境
* 要注意，在各个环境之间切换，尽量没有逻辑的改动，仅仅有小部分配置项的不同。这样，在配置项调通之后，进行任何的新增和迭代，就能保证自动化地进行了。
