nexus安装
参考官方docker安装



配置maven仓库


配置npm仓库
1. 创建npm库：
![创建npm库-1](https://0to.github.io/posts/20597/npm_create1.png)
![创建npm库-2](https://0to.github.io/posts/20597/npm_create2.png)
![创建npm库-3](https://0to.github.io/posts/20597/npm_create3.png)
![创建npm库-4](https://0to.github.io/posts/20597/npm_create4.png)

2. 权限配置：
- 激活 npm Bearer Token Realm
![激活 npm Bearer Token Realm](https://0to.github.io/posts/20597/npm_realms.png)

- 创建开发权限组对hosted npm私服库读写权限
![创建dev组](https://0to.github.io/posts/20597/group.png)
![添加hosted到dev组](https://0to.github.io/posts/20597/group2.png)

- 创建dev帐户并加入到开发权限组
![创建dev帐户](https://0to.github.io/posts/20597/user.png)
![加入到开发权限组](https://0to.github.io/posts/20597/user2.png)

PS: 匿名用户可以下载私服npm包，只有dev组内的帐户才能publish包到hosted私服


使用：
1.配置nexus私服
查看本机私服配置：

```
$ npm config get registry
```


设置本机配置到group私服：

```
#（写入到本机.npmrc文件）
$ npm config set registry http://${ip}:8081/repository/${npm_group}/
```


此时，项目执行npm install即从nexus上面进行下载包

2.配置publish帐户
有以下两种方式

Authentication Using Realm and Login
```
$ npm login --registry=http://${ip}:8081/repository/${npm_hosted}/
```


Username: dev （各位根据上面创建的帐户自行替换）

Password: ${dev_pass} （各位根据上面创建的密码自行替换）

Email: (this IS public) XXX@XXX.com (各位根据上面创建的邮箱自行替换)

Logged in as dev on http://${ip}:8081/repository/${npm_hosted}/.
        （写入到本机.npmrc文件）


Authentication Using Basic Auth
```
$ echo -n 'dev:${dev_pass}' | openssl base64  （dev帐户密码base64编码）
```


本机.npmrc文件里面增加以下行

```
email=XXX@XXX.com
always-auth=true
_auth=${base64编码后的值}
```


3.推送npm包到nexus
有以下两种方式

命令行 + 发布路径

$ npm publish –registry http://${ip}:8081/repository/${npm_hosted}


package.json配置发布路径（推荐）

项目package.json增加以下配置：

```
"publishConfig" : {
  "registry" : "http://${ip}:8081/repository/${npm_hosted}/"
},
```


执行以下命令即可
$ npm publish


配置Docker仓库


配置yum仓库

