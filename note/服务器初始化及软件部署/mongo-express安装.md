下载地址：https://github.com/mongo-express/mongo-express/

安装node参考：Node环境安装

修改配置编译环境pm2启动

```
cd mongo-express
 
修改配置
vim config.default.js
修改约70行的connectionString：'mongodb://admin:admin@localhost:27015/admin'
修改约99行开启basic认证 admin的值始终为true
修改约130行的basePath：'/mongo-express'
修改约133行的host：'127.0.0.1'
修改约149行的password：'123456'
 
编译build
npm i && npm run build
首次启动
pm2 start npm --name mongo-express -- run start
重载
pm2 reload mongo-express
查看日志
pm2 log mongo-express 如下
0|mongo-ex | Mongo Express server listening at http://127.0.0.1:8081
0|mongo-ex | Database connected
0|mongo-ex | Admin Database connected
 
验证启动
curl http://127.0.0.1:8081/mongo-express
```

配置nginx反向代理

server {
    listen               *:6666;
    server_name          localhost;
    access_log           logs/localhost.log;
    error_log            logs/localhost.error.log;
  
    charset               utf-8;
  
    location / {
        root   html;
        index  index.html index.htm;
    }
  
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
 
    location /mongo-express/ {
        proxy_http_version   1.1;
        proxy_set_header     Upgrade         $http_upgrade;
        proxy_set_header     Connection      'upgrade';
        proxy_set_header     Host            $host;
        proxy_set_header     X-Real-IP       $remote_addr;
        proxy_set_header     X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass   $http_upgrade;
  
        proxy_redirect       off;
        proxy_pass           http://172.20.30.162:8081/mongo-express/;
    }
}



mongodb 导入导出
```
mongoexport --host=localhost --port=27015 -u admin -p admin --authenticationDatabase=admin -d test -c test --type=csv --query='{userType:{$eq:3}}' --fields="_id"   -o ./test.csv

mongoimport --host=localhost --port=27015 -u admin -p admin --authenticationDatabase=admin -d test -c test --type=csv --fields="_id,gmtCreate.date\(2006-01-02 15:04:05\),gmtModified.date\(2006-01-02 15:04:05\),state,loginTime,loginIp,loginCountry,loginProvince,loginCity,loginArea,lastLoginTime,lastLoginIp,registerTime,registerIp,countryCode.string\(\),cellphone,email,password,salt,nickName" --headerline --columnsHaveTypes --ignoreBlanks --file=./test.csv

整库导入导出

mongodump --host=localhost --port=27015 -u admin -p admin --authenticationDatabase=admin -d test -o /home/mongod-backup/ --gzip
mongorestore --host=localhost --port=27015 -u admin -p admin --authenticationDatabase=admin -d test /home/test/ --gzip

mongodump --host=localhost --port=27015 -u admin -p admin --authenticationDatabase=admin -d test -o /home/mongod-backup/ --gzip
mongorestore --host=localhost --port=27015 -u admin -p admin --authenticationDatabase=admin -d test /home/test/ --gzip
```


