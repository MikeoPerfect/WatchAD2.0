# WatchAD2.0域威胁感知系统

[![Golang version](https://img.shields.io/badge/Golang-1.17.1-017D9C.svg)](https://github.com/golang/go/releases/tag/go1.17.1) [![Kafka version](https://img.shields.io/badge/Kafka-2.8-252122.svg)](https://kafka.apache.org/quickstart) [![MongoDB version](https://img.shields.io/badge/MongoDB-latest-15EB61.svg)](https://www.mongodb.com/docs/) [![Winlogbeat version](https://img.shields.io/badge/Winlogbeat-7.6.1-F04E98.svg)](https://www.elastic.co/guide/en/beats/winlogbeat/current/winlogbeat-installation.html)

[English Version](./README_EN.md) 

## 一、产品简述

WatchAD2.0是360信息安全中心开发的一款针对域安全的日志分析与监控系统，它可以收集所有域控上的事件日志、网络流量，通过特征匹配、协议分析、历史行为、敏感操作和蜜罐账户等方式来检测各种已知与未知威胁，功能覆盖了大部分目前的常见内网域渗透手法。相较于WatchAD1.0，有以下提升：  

1. 更丰富的检测能力：新增了账户可疑活动监测场景，加强了权限提升、权限维持等场景检测能力，涵盖包括异常账户/活动、Zerologon提权、SPN劫持、影子票证等更多检测面。
2. 基于Golang重构分析引擎：将开发语言从Python重构为Golang，利用其更高效的并发能力，提高对海量日志及流量等数据处理效率，确保告警检出及时有效。  
3. 整合简化架构：Web平台和检测引擎整合，简化部署过程，用户只依赖消息队列和存储组件即可完成部署。在提高系统的性能和稳定性的同时，也使得系统更加高效和易用，为用户提供更好的体验。

## 二、总体架构

WatchAD2.0分为四部分， 日志收集Agent、规则检测及日志分析引擎、缓存数据库、Web控制端，架构如下图所示。

![image](./images/Architecture.png)

> 其中流量检测链路暂不开源，可通过抓取域控流量，上传至360宙合SaaS PCAP分析平台进行威胁检测：https://zhouhe.360.cn/

## 三、目前支持的具体检测功能
- 异常活动检测：证书服务活动、创建机器账户事件活动、创建类似DC的用户账户、重置用户账户密码活动、TGT 票据相关活动；
- 凭证盗取：AS-REP 异常的流量请求、Kerberoasting 攻击行为、本地Dump Ntds文件利用；
- 横向移动：目录服务复制、异常的显示凭据登录行为、远程命令执行；
- 权限提升：ACL 异常修改行为、滥用证书服务权限提升、烂土豆提权、MS17-010、新增GPO监控、NTLM 中继检测、基于资源的约束委派、SPN 劫持、攻击打印服务、ZeroLogon 提权攻击；
- 权限维持：DCShadow 权限维持、DSRM 密码重置、GPO 权限委派、SamAccountName欺骗攻击、影子票证、Sid History 权限维持、万能密钥、可用于持久化的异常权限；
- 防御绕过：系统日志清空、关闭系统日志服务；
> 自定义检测规则：在{project_home}/detect_plugins/event_log/目录下可以修改或添加规则，需重新编译以使其生效。
## 四、平台展示
![image](./images/Platform.png)
## 五、编译&部署&运行指南
### 服务端部署操作：
**Docker部署（推荐）：**

WatchAD2.0依赖的组件有kafka、zookpeer，go1.17.1，可使用docker一键部署，操作如下：
在项目根目录下新建`.env`文件，需修改kafka地址、域控连接信息：
```shell
#KAFKA配置，需修改为当前服务器的IP
KAFKAHOST=10.10.10.10
KAFKAADV=PLAINTEXT://10.10.10.10:9092
BROKER=10.10.10.10:9092

#Mongo配置，默认账号密码
MONGOUSER=IATP
MONGOPWD=IATP-by-360
  
#域控配置，其中DCUSER为域内用户的DN
DCNAME="demo.com"
DCSERVER=10.10.10.11
DCUSER="CN=IATP, OU=Users, DC=demo, DC=com"
DCPWD="Pass123"

#WEB配置，可配置为域内任意用户，或DCUSER的CN
WEBUSER="IATP"
```
> 注意：如果您的域控未启用ssl，需将entrypoint.sh文件中的LDAP命令去掉--ssl参数

执行以下命令，以启动WatchAD2.0相关依赖组件、检测引擎及WEB服务。
```
docker-compose build
docker-compose up -d
```
访问服务器80端口进入web后台，输入WEBUSER对应的域用户账号密码即可登录成功。
> 注意：重启docker前，需删除kafka配置文件避免配置冲突：./data/kafka/logs/meta.properties

**手工部署：**

需提前准备Kafka集群和MongoDB集群。
1. 编译go程序
   使用go1.17.1版本在项目根目录下执行编译命令：
   `go mod vendor&&go build -o ./main main.go`
   将编译好的文件main及iatp_wbm目录拷贝至服务器
2. 初始化数据库信息
   `./main init --mongourl mongodb://mongo:password@127.0.0.1:27017`
   该操作将mongourl配置写入到 /etc/iatp.conf 配置文件中,如果需要重装需删除该文件再次由程序生成
3. 配置认证域LDAP
   由于web 管理端依赖于LDAP进行身份验证,所以需提前配置好认证域LDAP的相关配置
   `./main init --mongourl mongodb://mongo: password@127.0.0.1:27017 --domainname demo.com --domainserver 10.10.10.11 --username "IATP" --password "Pass123" --ssl`
4. 初始化数据表索引
   `./main init --mongourl mongodb://mongo: password@127.0.0.1:27017 --index`
5. 初始化kafka消费者配置
   修改为与kafka集群匹配的Brokers、Topic、Group等信息
   `./main init -source --sourcename ITEvent --sourceengine event_log --brokers 10.10.10.10:9092 --topic winlogbeat --group sec-ata --oldest false --kafka true`
6. Web管理端配置
   `./main web --init --authdomain demo.com --user IATP`
   设置初始需要登录的用户账户,该用户账户需要和ldap中的值保持一致.
7. 启动主检测引擎
   `./main run --engine_start`
8. 启动Web控制端
   `./main run --web_start`

访问服务器80端口进入web后台，输入--user对应的域用户账号密码即可登录成功。

**告警外发：**

可在管理后台-系统设置-数据源输出配置中，按照如下格式配置告警外发（当前仅支持kafka）：
```
{
  "Address": "10.10.10.10:9092",
  "Topic": "iatp_alarm"
}
```
### 客户端部署操作：
**开启审核**  
我们的分析基础是所有域控的所有事件日志，所以首先需要打开域控上的安全审核选项，让域控记录所有类型的事件日志。这里以 windows server 2016为例，在 本地安全策略 -> 安全设置 -> 本地策略 -> 审核策略，打开所有审核选项：

![image](./images/AuditPolicy.png)  

**安装winlogbeat**  
我们的分析基础是所有域控的所有事件日志，建议在所有域控服务器上安装winlogbeat，否则会产生误报和漏报。
前往[官网下载](https://www.elastic.co/cn/downloads/beats/winlogbeat)对应版本的winlogbeat，建议版本为7.6.1，其它版本的字段可能有变动，存在不兼容的可能性。  
参照如下示例修改配置文件winlogbeat.yml，假设kafka的IP为10.10.10.10，此时配置文件为：

```
winlogbeat.event_logs:
  - name: Security
    ignore_older: 1h
output.kafka:
    hosts: ["10.10.10.10:9092"]
    topic: winlogbeat
```
参照[官网教程](https://www.elastic.co/guide/en/beats/winlogbeat/current/winlogbeat-installation.html)安装winlogbeat服务即可