# 前端dev和staging环境部署

## 基础环境

首先安装 node.js 的版本管理工具 [nvm](https://github.com/creationix/nvm)：

```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash
```

或者

```
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash
```

安装我们目前使用的 node.js 的版本

```
nvm install 4.7.3
nvm install 6.9.5
```

查看是否安装成功

```
nvm ls
```

## 每一个项目的构建

```
cd 项目根目录下
npm run build
```