# 配置 Meta 节点
- Meta 节点配置设置
  - Global options
  - Enterprise license [enterprise]
  - Meta node [meta]
  - TLS [tls]

# Global 全局配置
### reporting-disabled
每 24 小时会将使用情况数据报告至 usage.influxdata.com。报告数据包含随机 ID、操作系统、架构、版本、系列数量以及其他使用情况数据。用户数据库的数据不会传输。将此选项更改为 true 可禁用报告功能。
- 默认值: false

### bind-address
RPC服务用于节点间通信和备份与恢复的TCP绑定地址。
- 默认值: ":8088"
- 环境变量: INFLUXDB_BIND_ADDRESS

### hostname
主机名
- 默认值: "localhost"
- 环境变量: INFLUXDB_HOSTNAME

# Enterprise license 企业许可证
[enterprise]

### license-key
在InfluxPortal上为您创建的许可证密钥。元节点通过端口 80 或 443 将许可证密钥传输至portal.influxdata.com，并接收一个临时的 JSON 许可证文件。服务器会在本地缓存该许可证文件。如果没有有效的许可证文件，数据进程将只能运行有限时间。如果您的服务器无法与https://portal.influxdata.com通信，则必须使用license-path设置离线授权。
- 默认值: ""
- 环境变量: INFLUXDB_ENTERPRISE_LICENSE_KEY

### license-path
您从 InfluxData 收到的永久 JSON 许可证文件的本地路径，适用于无法访问互联网的实例。如果没有有效的许可证文件，数据处理将只能运行有限的时间。如果需要许可证文件，请联系sales@influxdb.com 。

许可证文件应保存在集群中的每台服务器上，包括元节点、数据节点和企业节点。该文件包含 JSON 格式的许可证，并且必须可供influxdb用户读取。集群中的每台服务器都会独立验证其许可证。
- 默认值: ""
- 环境变量: INFLUXDB_ENTERPRISE_LICENSE_PATH

# Meta 节点
[meta]

### dir
存储路径
- 默认值: "/var/lib/influxdb/meta"
- 环境变量: INFLUXDB_META_DIR

### bind-address
监听地址
- 默认值: ":8089"
- 环境变量: INFLUXDB_META_BIND_ADDRESS

### http-bind-address
http 监听地址
监听地址
- 默认值: ":8091"
- 环境变量: INFLUXDB_META_HTTP_BIND_ADDRESS

### https-enabled
是否启用 https
- 默认值: false
- 环境变量: INFLUXDB_META_HTTPS_ENABLED

### https-certificate
https 证书
- 默认值: ""
- 环境变量: INFLUXDB_META_HTTPS_CERTIFICATE

### https-private-key
https 私钥
- 默认值: ""
- 环境变量: INFLUXDB_META_HTTPS_PRIVATE_KEY

### https-insecure-tls
元节点是否跳过通过 HTTPS 相互通信的证书验证。这在使用自签名证书进行测试时很有用。
- 默认值: false
- 环境变量: INFLUXDB_META_HTTPS_INSECURE_TLS

### data-use-tls
与 Data 节点通信是否使用 tls 加密
- 默认值: false
- 环境变量: INFLUXDB_META_DATA_USE_TLS

### data-insecure-tls
与 Data 节点通信是否允许不安全证书
- 默认值: false
- 环境变量: INFLUXDB_META_DATA_INSECURE_TLS

### gossip-frequency
广播频率
- 默认值: "5s"

### announcement-expiration
公告有效期
- 默认值: "30s"

### retention-autocreate
创建bucket时自动创建保留策略
- 默认值: true

### election-timeout
选举超时
- 默认值: "1s"

### heartbeat-timeout
心跳超时
- 默认值: "1s"

### leader-lease-timeout
领导者 lease 超时，领导者租约超时是指 Raft 领导者在未收到大多数节点的回复时，仍能保持领导者地位的时间。超时后，领导者将降级为跟随者状态。节点间延迟较高的集群可能需要增加此参数，以避免不必要的 Raft 选举。
- 默认值: "500ms"
- 环境变量: INFLUXDB_META_LEADER_LEASE_TIMEOUT

### commit-timeout
提交超时是指领导者向追随者发送包含领导者提交索引的消息之间的时间间隔。默认设置适用于大多数系统。
- 默认值: "50ms"
- 环境变量: INFLUXDB_META_COMMIT_TIMEOUT

### consensus-timeout
在获取最新的 Raft 快照之前等待达成共识的超时。
- 默认值: "30s"
- 环境变量: INFLUXDB_META_CONSENSUS_TIMEOUT

### cluster-tracing
集群详细日志
- 默认值: false
- 环境变量: INFLUXDB_META_CLUSTER_TRACING

### logging-enabled
是否打印日志
- 默认值: true
- 环境变量: INFLUXDB_META_LOGGING_ENABLED

### pprof-enabled
是否开启 pprof
- 默认值: true
- 环境变量: INFLUXDB_META_PPROF_ENABLED

### lease-duration
数据节点从元节点获取的租约的默认期限。租约在满足期限后自动到期。
租约可确保在给定时间内只有一个数据节点在运行某些任务。例如，连续查询 (CQ) 使用租约，这样所有数据节点就不会同时运行相同的 CQ。
- 默认值: "1m0s"
- 环境变量: INFLUXDB_META_LEASE_DURATION

### auth-enabled
是否开启认证
- 默认值: false
- 环境变量: INFLUXDB_META_AUTH_ENABLED

ldap-allowed
是否开启 ldap 认证
- 默认值: false
- 环境变量: INFLUXDB_META_SHARED_SECRET

### internal-shared-secret
内部共享密钥
- 默认值: ""
- 环境变量: INFLUXDB_META_INTERNAL_SHARED_SECRET

### password-hash
密码哈希
- 默认值: "bcrypt"
- 环境变量: INFLUXDB_META_PASSWORD_HASH

### ensure-fips
启动时是否检查 Influx 数据平台是否符合 FIPS 安全标准
- 默认值: false
- 环境变量: INFLUXDB_META_ENSURE_FIPS

# TLS TLS加密
[tls] 参考 Data 节点的 TLS 设置