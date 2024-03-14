# Rsyslog处理Nginx日志
> The rocket-fast system for log processing
> 使用rsyslog处理nginx日志，存储到PG数据库中

## cmd
```bash
# show conf
egrep -v "^$|#" /etc/rsyslog.conf

systemctl status rsyslog.service 
systemctl restart rsyslog.service 

# 监听端口
# -l listen -p source-port -4 ipv4 -u udp
nc -l -p 514 -4 -u

# 发送日志测试
logger "hello logs" --server {{hostname}} --port 514
logger "hello logs" --server 127.0.0.1 --port 514
# 监听输出： <5>Mar  8 15:49:08 root: hello logs 

# 检查配置文件是否正确
rsyslogd -N 1 
```

## Install
```bash
# query packagge
yum list rsyslog-*

# query installed packages starting with rsyslog-
rpm -qa | grep ^rsyslog-

yum install rsyslog
rsyslogd -v
yum install rsyslog-mysql rsyslog-pgsql -y
yum install rsyslog-pgsql -y
```

## Conf
* rsyslogd
> `/etc/rsyslog.d/` 下的文件名会影响启动顺序
```
ll /etc/rsyslog.d/10-nginx.conf
# 10-nginx.conf


# 加载imudp模块，用于接收来自网络的日志,主配置中已经开启则这里可以注释
module(load="imudp")

# 定义输入源，接收来自Nginx的日志
input(type="imudp" port="514")

# Load the ompgsql output module for sending logs to PostgreSQL server
module(load="ompgsql")

# 定义模板，包含时间戳、主机名、主机IP、应用名、严重性和消息内容
template(name="pgsql-template" type="list" option.sql="on") {
  constant(value="INSERT INTO dev.accesslog (event_timestamp, hostname, fromhost_ip, app_name, syslogseverity_text, log_line) values ('")
  property(name="timestamp" dateformat="pgsql")
  constant(value="','")
  property(name="hostname")
  constant(value="','")
  property(name="fromhost-ip")
  constant(value="','")
  property(name="app-name")
  constant(value="','")
  property(name="syslogseverity-text")
  constant(value="','")
  property(name="msg")
  constant(value="')")
}

action(type="ompgsql" server="10.0.0.0" port ="5432"
    user="uid"
    pass="pwd!"
    db="dev"
    template="pgsql-template"
    queue.type="linkedList")
```

* nginx
```
# demo
error_log syslog:server=192.168.1.1 debug;
access_log syslog:server=unix:/var/log/nginx.sock,nohostname;
access_log syslog:server=[2001:db8::1]:12345,facility=local7,tag=nginx,severity=info combined;
access_log syslog:server=127.0.0.1,facility=local6 main;

# log format： 可以自定义各nginx实例的属性

log_format  json_analytics escape=json '{'
    '"msec": "$msec", ' # request unixtime in seconds with a milliseconds resolution
    '"connection": "$connection", ' # connection serial number
    '"connection_requests": "$connection_requests", ' # number of requests made in connection
    '"pid": "$pid", ' # process pid
    '"request_id": "$request_id", ' # the unique request id
    '"request_length": "$request_length", ' # request length (including headers and body)
    '"remote_addr": "$remote_addr", ' # client IP
    '"remote_user": "$remote_user", ' # client HTTP username
    '"remote_port": "$remote_port", ' # client port
    '"time_local": "$time_local", '
    '"time_iso8601": "$time_iso8601", ' # local time in the ISO 8601 standard format
    '"request": "$request", ' # full path no arguments if the request
    '"request_uri": "$request_uri", ' # full path and arguments if the request
    '"args": "$args", ' # args
    '"status": "$status", ' # response status code
    '"body_bytes_sent": "$body_bytes_sent", ' # the number of body bytes exclude headers sent to a client
    '"bytes_sent": "$bytes_sent", ' # the number of bytes sent to a client
    '"http_referer": "$http_referer", ' # HTTP referer
    '"http_user_agent": "$http_user_agent", ' # user agent
    '"http_x_forwarded_for": "$http_x_forwarded_for", ' # http_x_forwarded_for
    '"http_host": "$http_host", ' # the request Host: header
    '"server_name": "$server_name", ' # the name of the vhost serving the request
    '"request_time": "$request_time", ' # request processing time in seconds with msec resolution
    '"upstream": "$upstream_addr", ' # upstream backend server for proxied requests
    '"upstream_connect_time": "$upstream_connect_time", ' # upstream handshake time incl. TLS
    '"upstream_header_time": "$upstream_header_time", ' # time spent receiving upstream headers
    '"upstream_response_time": "$upstream_response_time", ' # time spend receiving upstream body
    '"upstream_response_length": "$upstream_response_length", ' # upstream response length
    '"upstream_cache_status": "$upstream_cache_status", ' # cache HIT/MISS where applicable
    '"scheme": "$scheme", ' # http or https
    '"request_method": "$request_method", ' # request method
    '"server_protocol": "$server_protocol", ' # request protocol, like HTTP/1.1 or HTTP/2.0
    '"pipe": "$pipe", ' # "p" if request was pipelined, "." otherwise
    '"gzip_ratio": "$gzip_ratio" '
    '}';

```

* rsyslog.conf
> 注释默认的 `local7.*` 日志输出

## SQL script
```sql
CREATE TABLE dev.accesslog (
    id SERIAL PRIMARY KEY,
    event_timestamp TIMESTAMP NOT NULL,
    hostname VARCHAR(255),
    fromhost_ip VARCHAR(255),
    app_name VARCHAR(255),
    syslogseverity_text VARCHAR(255),
    log_line JSONB ,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);  
COMMENT ON TABLE dev.accesslog IS 'nginx 访问日志';

-- http_status_code 不等于 200 的数据
select log_line->>'status' http_status_code, *
FROM dev.accesslog 
WHERE (log_line->>'status')::integer <> 200;

-- request 以“GET /API/order” 开头的数据
SELECT *
FROM dev.accesslog
WHERE (log_line->>'request') LIKE 'GET /API/order”%';

-- request等于 “POST /API/queryOrderProcess HTTP/1.1”的数据
SELECT *
FROM dev.accesslog
WHERE (log_line->>'request') = 'POST /API/queryOrderProcess HTTP/1.1';

-- upstream_response_time 大于1的请求
SELECT (log_line->>'upstream_response_time')::numeric upstream_response_time, *
FROM dev.accesslog
WHERE (log_line->>'upstream_response_time') SIMILAR TO '([0-9]+\.?[0-9]*|[0-9]*\.?[0-9]+)'
AND (log_line->>'upstream_response_time')::numeric > 1;

```

## Doc
* [PostgreSQL Database Output Module (ompgsql)](https://www.rsyslog.com/doc/configuration/modules/ompgsql.html)
* [How to send Nginx logs to SQL database](https://www.shubhamdipt.com/blog/send-nginx-logs-to-sql-database/)


## TODO
> 部分版本的PG会出现如下报错，原因暂未知：`rsyslogd[15587]: db error (1): lost synchronization with server: got message type "S", length 1129464141   [v8.2404.0.master]`
