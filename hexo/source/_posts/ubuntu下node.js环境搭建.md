---
title: ubuntu下node.js环境搭建
date: 2017-07-23 22:42:04
tags: [tools]
---
### 由于使用gulp.js来对前端项目进行管理，所以搭建了node.js的环境<!--more-->
#### 首先， 需要安装node, npm
去官网下载安装包，解压后放到你的安装的目录。在这里我的是/opt/node-v6.11.1-linux-x64
#### 配置环境变量
下载的安装包中有node的可执行文件，但是要在全局环境下运行需要配置环境变量。
1. 使用 export PATH=$PATH:opt/node-v6.11.1-linux-x64/bin 可使环境暂时生效，但是打开新的terminal回话就会失效。
2. 编辑/etc/profile文件，将环境变量写入改文件，保存source /etc/profile便可生效
3. npm -v node -v命令查看版本，node环境便安装成功了

#### 配置淘宝的cnpm
由于npm的下载依赖速度太慢，可以切换到淘宝的源
使用npm install -g cnpm --registry=https://registry.npm.taobao.org命令， 这里的-g命令表示全局安装，也就是在计算机的全局环境都可以使用cnpm命令
这时候会提示没有权限的问题，我们需要使用sudo命令。
但是系统提示：sudo npm 找不到命令。
我们需要建立软连接到/usr/bin目录下，这样我们的sudo命令才能找到node和npm命令

#### 建立node和npm的软链
sudo ln -s /opt/node-v6.11.1-linux-x64/bin/node /usr/bin/node
sudo ln -s /opt/node-v6.11.1-linux-x64/bin/node /usr/lib/node
sudo ln -s /opt/node-v6.11.1-linux-x64/bin/npm /usr/bin/npm

#### 安装成功，大功告成。