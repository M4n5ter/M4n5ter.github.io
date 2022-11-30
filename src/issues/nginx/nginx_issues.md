## 反向代理

[Module ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)

### 保护 host header

在使用 nginx 对 minio 进行反向代理时，遇到了这个 issue [#7936](https://github.com/minio/minio/issues/7936) 。原因是没有保护 Host header。

minio 官方给出的配置如下：

```nginx
server {
 listen 80;
 server_name example.com;

 # To allow special characters in headers
 ignore_invalid_headers off;
 # Allow any size file to be uploaded.
 # Set to a value such as 1000m; to restrict file size to a specific value
 client_max_body_size 0;
 # To disable buffering
 proxy_buffering off;

 location / {
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header X-Forwarded-Proto $scheme;
   proxy_set_header Host $http_host; # 主要是此处，保护 host header

   proxy_connect_timeout 300;
   # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
   proxy_http_version 1.1;
   proxy_set_header Connection "";
   chunked_transfer_encoding off;

   proxy_pass http://localhost:9000; # If you are using docker-compose this would be the hostname i.e. minio
   # Health Check endpoint might go here. See https://www.nginx.com/resources/wiki/modules/healthcheck/
   # /minio/health/live;
 }
}
```

[Module ngx_http_upstream_module#keepalive](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive) 中提到：

> For HTTP, the proxy_http_version directive should be set to “1.1” and the “Connection” header field should be cleared

```nginx
proxy_http_version 1.1;
proxy_set_header Connection "";
```









### 解决跨域

```nginx
location /api {
    add_header Access-Control-Allow-Origin * always;
    add_header Access-Control-Allow-Headers *;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, OPTIONS";
    proxy_pass https://baidu.com;
}
```

 

## location 块

[Module ngx_http_core_module#location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)

路径正则匹配是选择匹配度最高的一条规则（nginx 会先选中前缀最长的，然后在逐个规则匹配，匹配成功后不再继续匹配）。

例如：

```nginx
location /abc {...}
location /abc/d {...}
```

> To find location matching a given request, nginx first checks locations defined using the prefix strings (prefix locations). Among them, the location with the longest matching prefix is selected and remembered. Then regular expressions are checked, in the order of their appearance in the configuration file. The search of regular expressions terminates on the first match, and the corresponding configuration is used.

所以 /abc/d 路径的请求不会被 /abc location 拦截。
