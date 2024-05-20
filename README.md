SugarChain Blockchain Explorer - 1.7.4
================
<details>
<summary>Click to view manual deployment</summary>
<br>
An open source block explorer written in node.js.

*Note: This block explorer needs to be on the same server as the API node, using mongodb to save data, requiring 40GB of space. Please check if the hard disk space is sufficient*

### Requires

*  Ubuntu >= 20.04
*  node.js >= 12.14.0
*  mongodb 4.4.x
*  *coind

### nvm install
	
	sudo apt-get update
	cd && curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.9/install.sh | bash

	vim /etc/profile

Append at the end of the file

	export NVM_DIR="$HOME/.nvm"
	[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm
	[ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
	
Then `:wq` save and re source the file

	source /etc/profile

### Nodejs install

	nvm install v12.14.0

### MongoDB install

	sudo apt-get install -y libcurl4 openssl
	sudo apt-get install gnupg
	wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
	echo "deb https://mirrors.tuna.tsinghua.edu.cn/mongodb/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
	sudo apt-get update
	sudo apt-get install -y mongodb-org
	sudo systemctl daemon-reload
	sudo systemctl start mongod

### Create database

Enter MongoDB cli:

    $ mongo

Create databse:

    > use explorerdb

Create user with read/write access:

    > db.createUser( { user: "mongo-user", pwd: "mongo-pwd", roles: [ "readWrite" ] } )
	> exit

### Get the source

    git clone https://github.com/bailaoshijiadao/sugarchain-blockchain-explorer

### Install node modules

    cd sugarchain-blockchain-explorer && npm install --production

### Configure

*Make required changes in settings.json*

*dbsettings*

	"dbsettings": {
		"user": "mongo-user",
		"password": "mongo-pwd",
		......
	},

*wallet*
  
	"wallet": {
    ......
    "username": "baihe",
    "password": "passwordbaihe"
	}, //username and password based on API node sugarchain.conf file.

*port*

  "port" : 3001, //port modifiable, default 3001

### Start Explorer

    npm start

*Note: mongod must be running to start the explorer*

As of version 1.4.0 the explorer defaults to cluster mode, forking an instance of its process to each cpu core. This results in increased performance and stability. Load balancing gets automatically taken care of and any instances that for some reason die, will be restarted automatically. For testing/development (or if you just wish to) a single instance can be launched with

    node --stack-size=10000 bin/instance

To stop the cluster you can use

    npm stop

### Syncing databases with the blockchain

sync.js (located in scripts/) is used for updating the local databases. This script must be called from the explorers root directory.

    Usage: node scripts/sync.js [database] [mode]

    database: (required)
    index [mode] Main index: coin info/stats, transactions & addresses
    market       Market data: summaries, orderbooks, trade history & chartdata

    mode: (required for index database only)
    update       Updates index from last sync to current block
    check        checks index for (and adds) any missing transactions/addresses
    reindex      Clears index then resyncs from genesis to current block

    notes:
    * 'current block' is the latest created block when script is executed.
    * The market database only supports (& defaults to) reindex mode.
    * If check mode finds missing data(ignoring new data since last sync),
      index_timeout in settings.json is set too low.
	  
use new terminals execute command
	
	cd sugarchain-blockchain-explorer
	rm -f ./tmp/index.pid
	node scripts/sync.js index update

### COMPLETE

#  Optional Settings

## Domain settings

### Point domain to your server

### Install Nginx

	sudo apt-get update
	sudo apt install nginx -y
	
### Create nginx config (replace explorer.example.com with your domain)

	sudo unlink /etc/nginx/sites-enabled/explorer.example.com.conf
	rm -rf /etc/nginx/sites-available/explorer.example.com.conf
	sudo vim /etc/nginx/sites-available/explorer.example.com.conf
	
Write the following content (replace explorer.example.com with your domain)
	
	server {
		server_name explorer.example.com;

		location / {
			proxy_pass http://localhost:3001;
			proxy_http_version 1.1;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection 'upgrade';
			proxy_set_header Host $host;
			proxy_cache_bypass $http_upgrade;
		}

		location /socket.io {
			include proxy_params;
			proxy_http_version 1.1;
			proxy_buffering off;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection "Upgrade";
			proxy_pass http://127.0.0.1:3001/socket.io;
		}

		listen 80;
	}

### Activate nginx config (replace explorer.example.com with your domain)

	sudo ln -s /etc/nginx/sites-available/explorer.example.com.conf /etc/nginx/sites-enabled
	
### Install certbot for ssl certificate

	sudo apt install snapd -y
	sudo snap install --classic certbot
	
### Obtain certificate (replace explorer.example.com with your domain)

	sudo certbot --nginx -d explorer.example.com
	
After that blockchain explorer should be accessible via domain you pointed


### OTHER

*It is recommended to have this script launched via a cronjob at 1+ min intervals.*

**crontab**

*Note: Set scheduled tasks after sync is completed*

fix forever location for crontab
	
	sudo ln -s $(which node) /usr/bin/node

*Example crontab; update index every minute and market data every 2 minutes*

    */1 * * * * cd /root/sugarchain-explorer && node scripts/sync.js index update > /dev/null 2>&1
    */2 * * * * cd /root/sugarchain-explorer && node scripts/sync.js market > /dev/null 2>&1

*for Example*

	*/1 * * * * rm -f ./tmp/index.pid && cd /root/sugarchain-explorer && node scripts/sync.js index update && node scripts/sync.js market && node scripts/peers.js > /dev/null 2>&1

### Wallet

SugarChain Blockchain Explorer is intended to be generic, so it can be used with any wallet following the usual standards. The wallet must be running with atleast the following flags

    -daemon -txindex
    
### Security

Ensure mongodb is not exposed to the outside world via your mongo config or a firewall to prevent outside tampering of the indexed chain data. 

### Known Issues

**script is already running.**

If you receive this message when launching the sync script either a) a sync is currently in progress, or b) a previous sync was killed before it completed. If you are certian a sync is not in progress remove the index.pid and db_index.pid from the tmp folder in the explorer root directory.

	cd sugarchain-blockchain-explorer
    rm tmp/index.pid
    rm tmp/db_index.pid

**exceeding stack size**

    RangeError: Maximum call stack size exceeded

Nodes default stack size may be too small to index addresses with many tx's. If you experience the above error while running sync.js the stack size needs to be increased.

To determine the default setting run

    node --v8-options | grep -B0 -A1 stack_size

To run sync.js with a larger stack size launch with

    node --stack-size=[SIZE] scripts/sync.js index update

Where [SIZE] is an integer higher than the default.

*note: SIZE will depend on which blockchain you are using, you may need to play around a bit to find an optimal setting*

### License

Copyright (c) 2015, Iquidus Technology  
Copyright (c) 2015, Luke Williams  
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.

* Neither the name of Iquidus Technology nor the names of its
  contributors may be used to endorse or promote products derived from
  this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

</details>

<details>
<summary>点击查看手动部署</summary>
<br>
一个用node.js编写的开源区块浏览器

*注意: 此区块浏览器需要与API节点同一服务器上, 使用mongodb保存数据, 需要40GB的空间. 请检查硬盘空间是否足够*

### 依赖

*  node.js >= 12.14.0
*  mongodb 4.4.x
*  *coind

### nvm 安装
	
	sudo apt-get update
	cd && curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.9/install.sh | bash

	vim /etc/profile

英文输入法状态下按下字母i按键, 在文件最后追加以下内容

	export NVM_DIR="$HOME/.nvm"
	[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm
	[ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
	
然后按下 `Esc` 按键, 输入 `:wq` 保存并重新加载系统环境变量并立即生效

	source /etc/profile

### Nodejs 安装

	nvm install v12.14.0

### MongoDB 安装

	sudo apt-get install -y libcurl4 openssl
	sudo apt-get install gnupg
	wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
	echo "deb https://mirrors.tuna.tsinghua.edu.cn/mongodb/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
	sudo apt-get update
	sudo apt-get install -y mongodb-org
	sudo systemctl daemon-reload
	sudo systemctl start mongod

### 数据库创建

输入命令进入 MongoDB 控制台 :

    $ mongo

创建数据库 :

    > use explorerdb

创建具有 读/写 访问权限的用户, 然后退出:

    > db.createUser( { user: "mongo-user", pwd: "mongo-pwd", roles: [ "readWrite" ] } )
	> exit

### 获取源码

    git clone https://github.com/bailaoshijiadao/sugarchain-blockchain-explorer

### 安装 node 依赖

    cd sugarchain-blockchain-explorer && npm install --production

### Configure

*settings.json 文件中进行所需的更改*

根据前面设置的信息修改数据库配置*dbsettings*

	"dbsettings": {
		"user": "mongo-user",
		"password": "mongo-pwd",
		......
	},

钱包配置 *wallet*
  
	"wallet": {
    ......
    "username": "baihe",
    "password": "passwordbaihe"
	}, //糖链 API 节点 sugarchain.conf 文件中设置的用户名和密码.

对外端口设置 *port*

  "port" : 3001, //端口修改, 默认 3001

### 启动区块浏览器

    npm start

*注意: mongodb必须运行后才能启动区块浏览器*

从1.4.0版本开始, 资源管理器默认为集群模式, 将其进程的一个实例分叉到每个cpu核心, 这提高了性能和稳定性, 负载平衡会自动得到处理. 任何由于某种原因而失效的实例都会自动重新启动, 对于测试/开发 (或者如果您只是想), 可以使用启动单个实例

    node --stack-size=10000 bin/instance

要停止群集, 可以使用

    npm stop

### 数据库与区块链同步

sync.js (位于scripts/中) 用于更新本地数据库, 必须从 sugarchain-blockchain-explorer 根目录调用此脚本, 命令参数详解如下

    Usage: node scripts/sync.js [database] [mode]

    database: (required)
    index [mode] Main index: coin info/stats, transactions & addresses
    market       Market data: summaries, orderbooks, trade history & chartdata

    mode: (required for index database only)
    update       Updates index from last sync to current block
    check        checks index for (and adds) any missing transactions/addresses
    reindex      Clears index then resyncs from genesis to current block

    notes:
    * 'current block' is the latest created block when script is executed.
    * The market database only supports (& defaults to) reindex mode.
    * If check mode finds missing data(ignoring new data since last sync),
      index_timeout in settings.json is set too low.
	  
开启新的终端执行命令
	
	cd sugarchain-explorer
	rm -f ./tmp/index.pid
	node scripts/sync.js index update

### 完成

#  可选的一些设置

## 域名设置

### 将域名解析到自己服务器的IP地址

### 安装 Nginx

	sudo apt-get update
	sudo apt install nginx -y
	
### 创建 nginx 配置文件 (将 explorer.example.com 替换为你的域名)

	sudo unlink /etc/nginx/sites-enabled/explorer.example.com.conf
	rm -rf /etc/nginx/sites-available/explorer.example.com.conf
	sudo vim /etc/nginx/sites-available/explorer.example.com.conf
	
写入以下内容 (将 explorer.example.com 替换为你的域名)
	
	server {
		server_name explorer.example.com;

		location / {
			proxy_pass http://localhost:3001;
			proxy_http_version 1.1;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection 'upgrade';
			proxy_set_header Host $host;
			proxy_cache_bypass $http_upgrade;
		}

		location /socket.io {
			include proxy_params;
			proxy_http_version 1.1;
			proxy_buffering off;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection "Upgrade";
			proxy_pass http://127.0.0.1:3001/socket.io;
		}

		listen 80;
	}

### 激活 nginx 配置 (将 explorer.example.com 替换为你的域名)

	sudo ln -s /etc/nginx/sites-available/explorer.example.com.conf /etc/nginx/sites-enabled
	
### 为 ssl 证书安装 certbot

	sudo apt install snapd -y
	sudo snap install --classic certbot
	
### 获得证书 (将 explorer.example.com 替换为你的域名)

	sudo certbot --nginx -d explorer.example.com
	
之后, 区块浏览器应该可以通过你指向的域名进行访问


### OTHER

*建议每隔 1 分钟以上通过 crontab 启动此脚本.*

**crontab**

*注意: 数据和区块都同步完成后再设置计划任务*

修复 crontab 任务的 node 位置

	sudo ln -s $(which node) /usr/bin/node

*示例crontab: 每分钟更新数据, 每2分钟更新数据*

    */1 * * * * cd /root/sugarchain-explorer && node scripts/sync.js index update > /dev/null 2>&1
    */2 * * * * cd /root/sugarchain-explorer && node scripts/sync.js market > /dev/null 2>&1

*完整示例, 每分钟更新数据*

	*/1 * * * * rm -f ./tmp/index.pid && cd /root/sugarchain-explorer && node scripts/sync.js index update && node scripts/sync.js market && node scripts/peers.js > /dev/null 2>&1

### 钱包设置

糖链区块浏览器是通用的, 因此它可以按照通常的标准与任何钱包一起使用. 钱包运行时必须至少具有以下标志

    -daemon -txindex
    
### 安全注意

确保 mongodb 不会通过 mongo 配置或防火墙暴露给外部世界, 以防止外部篡改索引链数据.

### 已知问题

**执行命令提示 script is already running.**

如果您在启动同步脚本时收到此消息，则a) 当前正在进行同步. 或者b) 上一次同步在完成之前已终止. 如果您确信没有进行同步，请从根目录中的tmp文件夹中删除index.pid和db_index.pid.

	cd sugarchain-blockchain-explorer
    rm tmp/index.pid
    rm tmp/db_index.pid

**超过堆栈大小**

    RangeError: Maximum call stack size exceeded

节点默认堆栈大小可能太小, 无法用许多tx索引地址. 如果在运行sync.js时遇到上述错误, 则需要增加堆栈大小.

要确定使用默认设置, 请运行

    node --v8-options | grep -B0 -A1 stack_size

要使用较大的堆栈大小运行sync.js, 请使用下面命令启动

    node --stack-size=[SIZE] scripts/sync.js index update

其中 [SIZE] 是一个高于默认值的整数.

*注意: [SIZE]大小将取决于您使用的区块链, 您可能需要玩一玩才能找到最佳设置*

### License

Copyright (c) 2015, Iquidus Technology  
Copyright (c) 2015, Luke Williams  
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.

* Neither the name of Iquidus Technology nor the names of its
  contributors may be used to endorse or promote products derived from
  this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

</details>
