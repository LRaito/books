# 数据库中间件 shardingsphere-proxy

### 需求背景
golang 项目使用 ent 作为数据库 orm，现在需要同时兼容 mysql 和达梦数据库，测试是否能使用中间件来适配多种数据库

### 运行方式
安装 jdk17，运行 bin/start.sh


### 配置说明
本地端口：3307 数据库：obs
目标数据库配置: conf/database-sharding.yaml,dataSources
```
  ds_1:
    url: jdbc:dm://193.169.203.134:5236/iiot
    username: SYSDBA
    password: ZAQ!2wsx
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
```

本地授权信息在: conf/global.yaml
```
authority:
 users:
   - user: SYSDBA
     password: ZAQ!2wsx
   - user: sylink
     password: sylink
```

### 问题
报错还未解决
```
mysql: querying mysql version Error 63529 (42000): line 1, column 24, nearby [VARIABLES] has error: Syntax error
```