## Docker compose 示例

**docker-compose.yml**

```yaml
version: '3.7'

#Settings and configurations that are common for all containers

x-minio-common: &minio-common
 # set your minio version
 image: quay.io/minio/minio:RELEASE.2022-10-15T19-57-03Z
 restart: unless-stopped
 command: server --console-address ":9001" http://minio{1...4}/data{1...2}
 expose:
 - "9000"
 - "9001"
 environment:
 MINIO_ROOT_USER: <YOUR MINIO ROOT USER>
 MINIO_ROOT_PASSWORD: <YOUR MINIO ROOT PASSWORD>
 healthcheck:
 test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
 interval: 30s
 timeout: 20s
 retries: 3 
# starts 4 docker containers running minio server instances.
# using nginx reverse proxy, load balancing, you can access
# it through port 9000.
services:
 minio1:
 <<: *minio-common
 hostname: minio1
 volumes:
 - data1-1:/data1
 - data1-2:/data2 
minio2:
 <<: *minio-common
 hostname: minio2
 volumes:
 - data2-1:/data1
 - data2-2:/data2 
minio3:
 <<: *minio-common
 hostname: minio3
 volumes:
 - data3-1:/data1
 - data3-2:/data2 
minio4:
 <<: *minio-common
 hostname: minio4
 volumes:
 - data4-1:/data1
 - data4-2:/data2 
nginx:
 image: nginx:1.19.2-alpine
 hostname: minio_gateway
 volumes:
 - ./nginx.conf:/etc/nginx/nginx.conf:ro
 ports:
 - "9000:9000"
 - "80:80"
 depends_on:
 - minio1
 - minio2
 - minio3
 - minio4 
## By default this config uses default local driver,
## For custom volumes replace with volume driver configuration.
volumes:
 data1-1:
 data1-2:
 data2-1:
 data2-2:
 data3-1:
 data3-2:
 data4-1:
 data4-2:
```

**nginx.conf**

```nginx
worker_processes  auto;

events {
    worker_connections  1024;
}


stream {
        #log_format basic '$remote_addr [$time_local] '
        #         '$protocol $status $bytes_sent $bytes_received '
        #         '$session_time';
        #access_log /var/log/nginx/stream-access.log basic buffer=32k;


        upstream minio{
            server minio1:9000 weight=1;
            server minio2:9000 weight=1;
            server minio3:9000 weight=1;
            server minio4:9000 weight=1;
    }

        upstream minio_console{
            server minio1:9001 weight=1;
            server minio2:9001 weight=1;
            server minio3:9001 weight=1;
            server minio4:9001 weight=1;
    }

    server{
        listen 9003;
        proxy_pass minio;
    }

    server{
        listen 80;
        proxy_pass minio_console;
    }
}

http {
    server{
        listen 9000;
        # 允许 header 中包含特殊字符
         ignore_invalid_headers off;
         # 允许上传任意大小的文件
         # 可以把值改成像 1000m 这样来限制文件的大小
         client_max_body_size 0;
         # 禁用缓冲
         proxy_buffering off;
        # 允许跨域
        add_header Access-Control-Allow-Origin * always;
        add_header Access-Control-Allow-Headers *;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, OPTIONS";
        add_header Access-Control-Allow-Methods *;

         location / {
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto $scheme;
               proxy_set_header Host $http_host; # 主要是此处，保护 host header

               proxy_connect_timeout 300;
               # 默认是 HTTP/1, keepalive 需要 HTTP/1.1
               proxy_http_version 1.1;
               proxy_set_header Connection "";
               chunked_transfer_encoding off;
                proxy_pass http://localhost:9003;
        }

    }
}
```

## 快速启动

将 `docker-compose.yml` 与 `nginx.conf` 置于统一目录下，然后执行：

```bash
$ docker compose up -d
# -or-
$ docker-compose up -d
```
