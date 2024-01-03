# Nginx

## 参考资料

Nginx 官网文档：<https://nginx.org/en/docs/>
Nginx 配置指南：<https://juejin.cn/post/7267003603095879714>
Nginx 安装配置详解: <https://mp.weixin.qq.com/s/Cd9T_nhAtJ8hI6waEzZiEg>

## Ngxin 结构组织

| 配置块      | 功能描述                                           |
| ----------- | -------------------------------------------------- |
| 全局块      | 与Nginx运行相关的全局设置                          |
| events 块   | 与网络连接有关的设置                               |
| http 块     | 代理、缓存、日志、虚拟主机等的配置                 |
| server 块   | 虚拟主机的参数设置（一个http块可包含多个server块） |
| location 块 | 定义请求路由及页面处理方式                         |

![nginx.conf](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e24c0f8b7094943938ce58a06797964~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=1012&h=1228&s=83045&e=png&b=d2f0fd)

## Ngxin 命令

```shell
# Windows 启动
start nginx

# 重启
nginx -s reload

# 快速关闭命令
nginx -s stop

# 有序停止（需要进程完成当前工作后再停止）
nginx -s quit

# Linux 杀死 Nginx 进程
killall nginx

# Windows 杀死 Nginx 进程
taskkill /f /im nginx.exe
taskkill /pid 1916 /f # 根据 pid 杀掉进程

# 检验配置文件的语法是否可用
nginx -t
```

## location 路径映射

官方文档：<https://nginx.org/en/docs/http/ngx_http_core_module.html?&_ga=2.58661034.1076441705.1704257483-1207797590.1704257483#location>

参考资料：<https://www.cnblogs.com/zjfjava/p/10760157.html>

### 格式

```yaml
location [ = | ~ | ~* | !~ | !~* | ^~ | @ ] uri {...}
```

- `=`：精确匹配。如果匹配成功，立即停止搜索并处理此请求。
- `^~`：前缀匹配。如果匹配成功，不再匹配其他 `location`，且不查询正则表达式。
- `~`：执行正则匹配，区分大小写。
- `~*`：执行正则匹配，不区分大小写。
- `!~`：执行正则匹配，区分大小写不匹配。
- `!~*`：执行正则匹配，不区分大小写不匹配。
- `@`：指定命名的 `location`，只能用于内部重定向请求，不能被外部请求所访问，如 `error_page` 和 `try_files`。
- `uri`：待匹配的请求字符串。可以是普通字符串或包含正则表达式。

### 优先级

> `=` > `^~` > 正则匹配 (`~`, `~*`, `!~`, `!~*`) > 无特定标识

1. 第一优先级：**精确匹配**

   ```nginx
   location = / {
     # http://abc.com [匹配成功]
     # http://abc.com/index [匹配失败]
   }
   ```

   只有 URI 完全匹配时才会执行指定的 location 块中的指令。

2. 第二优先级：**前缀匹配**

   ```nginx
   location ^~ /img/ {
     # 以 /img/ 开头的请求，都会匹配上
     # http://abc.com/img/a.jpg [匹配成功]
     # http://abc.com/img/b.mp4 [匹配成功]
   }
   ```

   所有以 `/img/` 开头的URI都将被匹配到这个 location 块中。

3. 第三优先级：**正则匹配**

   ```nginx
   # 区分大小写：
   location ~ /documents/Abc/\d+ {
     root /var/www/blog/;
   }
   ```

   所有形如 `/documents/Abc/{一个或多个数字}` 的 URI 都将被匹配到这个 location 块中。

   ```nginx
   # 不区分大小写：
   location ~* /Example/ {
     # 忽略 uri 部分的大小写
     # http://abc.com/test/Example/ [匹配成功]
     # http://abc.com/example/ [匹配成功]
   }
   
   location ~* \.(gif|jpg|jpeg)$ {
     # 所有以gif、jpg 或 jpeg 结尾的 URI 都将被匹配到这个 location 块中。
   }
   ```

4. 第四优先级：**模糊匹配**

   ```nginx
   location / {
     # http://abc.com [匹配成功]
     # http://abc.com/index [匹配成功]
   }
   ```

   表示匹配所有请求，但是其优先级最低，只有当前面的三种匹配方式都无法匹配该请求时，才会执行该 location 块中的指令。

## 反向代理与 proxy_pass

要配置 Nginx 作为反向代理，需要使用以下常用指令：

| 指令              | 说明                                                        |
| ----------------- | ----------------------------------------------------------- |
| proxy_pass        | 定义后端服务器的地址                                        |
| proxy_set_header  | 修改从客户端传递到代理服务器的请求头                        |
| proxy_hide_header | 隐藏从代理服务器返回的响应头                                |
| proxy_redirect    | 修改从代理服务器返回的响应头中的`Location`和`Refresh`头字段 |

### 示例配置

```nginx
server {
    listen 80;
    server_name example.com;

    location /api {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- 通过 location 指令匹配请求的 uri 为 `/api`，并将这个 uri 转发到 `localhost:8080` 进行处理；
- 通过 `proxy_set_header` 指令，将客户端的 `Host`、`X-Real-IP`、`X-Forwarded-For` 信息发送给后端服务器；

### 注意事项

1. 当使用 `proxy_pass` 指令时，确保后端服务器是可用的，否则 Nginx 将返回错误；
2. 使用 `proxy_set_header` 确保后端服务器接收到正确的请求头；
3. 如果后端服务器和 Nginx 在不同的机器上，确保网络连接是稳定的；
4. 如果你的代理服务器没有权限访问到后端服务器，Nginx 将返回 `502 Bad Gateway` 或者 `504 Gateway Timeout` 错误。此时需要检查代理服务器是否与后端服务器的网络环境通畅

反向代理不仅可以提高网站的性能和可靠性，还可以用于负载均衡、缓存静态内容、维护和安全等多种用途。

## 跨域与 add_header

CORS：<https://developer.mozilla.org/zh-CN/docs/Glossary/CORS>

```nginx
location / {
    # 允许跨域的源（$http_origin 动态获取请求客户端请求的域，）
    # add_header Access-Control-Allow-Origin *; # 不用 * 的原因是带 cookie 的请求不支持 *
    add_header Access-Control-Allow-Origin 'http://127.0.0.1:8009';

    # 表示请求头的字段 动态获取
    add_header Access-Control-Allow-Headers $http_access_control_request_headers;

    # 指定允许跨域的方法，* 代表所有
    # add_header Access-Control-Allow-Methods GET,POST,OPTIONS;
    add_header Access-Control-Allow-Methods *;

    # 预检命令的缓存，单位为秒，如果不缓存每次会发送两次请求
    add_header Access-Control-Max-Age 7200;

    # CORS 预检请求不能包含凭据。预检请求的响应必须指定 true 来表明可以携带凭据进行实际的请求。
    add_header Access-Control-Allow-Credentials true;

    # 也可以针对指定请求方法单独处理（比如 OPTIONS 预检请求）
    if ($request_method = 'OPTIONS') {
        return 204;
    }
}
```

## 单页面应用刷新 404 与 try_files

### try_files

官方文档：<https://nginx.org/en/docs/http/ngx_http_core_module.html#try_files>

参考资料：<https://www.xiewo.net/blog/show/559/>、<https://www.jianshu.com/p/46eb10532ba3>

- 按指定顺序检查文件是否存在，并使用第一个找到的文件进行请求处理，处理只在当前上下文中执行。

- 文件的路径是根据 `root` 和 `alias` 两个指令，从文件参数中构造出来的。

- 如果名称末尾指定斜杠，则表示为一个目录，比如 `$uri/`。此时会在该目录下查找由 `index` 指令指定的初始页（默认初始页为 `index.html`）

- 如果没有找到任何文件，则会对最后一个参数中指定的 uri 进行内部重定向。

  **请注意：**最后一个参数是回退 URI 且必须存在（命名 location 也可以当做最后一个参数使用），否则会出现内部 500 错误。

  ```nginx
  location /images/ {
      root /data/user/;
      index index.gif;
      try_files $uri $uri/ /images/default.gif @gif;
  }
  
  location @gif {
      expires 30s;
  }
  ```
  
  当请求 <http://localhost:8080/images/assets> 时，`$uri` 为 `/images/assets`。查找顺序如下：
  
  1. 查找 `/data/user/images/assets`文件；
  2. 查找 `/data/user/images/assets`文件夹里的 `index` 指定文件，即 `index.gif`；
  3. 查找 `/data/user/images/default.gif` 文件
  4. 都没找到，则对最后一个参数进行重定向，最终匹配到 `@gif` location 模块。

### index

官方文档：<https://nginx.org/en/docs/http/ngx_http_index_module.html#index>

参考资料：<https://blog.csdn.net/qq_32331073/article/details/81945134>

- 该指令后面可以跟多个文件，用空格隔开；
- 如果包括多个文件，Nginx 会根据文件的枚举顺序来检查，直到查找的文件存在；
- 文件可以是相对路径也可以是绝对路径，绝对路径需要放在最后；
- 文件可以使用变量`$`来命名；
- 该指令默认值为 `index index.html`。

index 实际工作方式：

1. 如果文件存在，则使用文件作为路径，发起内部重定向。直观上看就像再一次从客户端发起请求，Nginx 再一次搜索 location 一样。
2. 既然是内部重定向，域名 + 端口不发生变化，所以只会在同一个 server 下搜索。
3. 同样，如果内部重定向发生在 proxy_pass 反向代理后，那么重定向只会发生在代理配置中的同一个server。

示例：

```nginx
server {
    listen      80;
    server_name example.org www.example.org;    
    
    location / {
        root    /data/www;
        index   index.html index.php;
    }
    
    location ~ \.php$ {
        root    /data/www/test;
    }
}
```

1. 如果你使用 `example.org` 或 `www.example.org` 直接发起请求，那么首先会访问到 `/` 的location，结合`root` 与 `index` 指令，会先判断 `/data/www/index.html` 是否存在，如果不存在，则接着查看 `/data/www/index.php`。
2. 如果 `/data/www/index.php` 存在，则使用 `/index.php` 发起内部重定向，就像从客户端再一次发起请求一样，Nginx 会再一次搜索 location，毫无疑问匹配到第二个 `~ \.php$`，从而访问到 `/data/www/test/index.php` 。

### rewrite

> 语法：rewrite regex replacement [flag];
>
> regex：正则表达式；
>
> replacement：替换内容；
>
> flag：命令执行模式，有两个值  [break | last]

`rewrite`主要功能就是使用 Nginx 提供的全局变量或自己设置的变量，结合正则表达式和标志位实现 url 重写以及重定向。

- 对 url 进行重写指的是重写真实请求路径，如果是同域内，浏览器不会发生跳转（302），如果是非同域浏览器会发生跳转（307）。

- 只能对域名后边的除去查询字符串的部分起作用，例如 http://seanlook.com/a/we/index.php?id=1&u=str 只对 `/a/we/index.php` 重写。

- `break` 表示重写后停止不再匹配，一般用于接口重定向；

  例如将 `http://127.0.0.1/down/123.xls` 冲重定向到 `http://192.168.0.1:8080/file/123.xls ` 来解决跨域下载。

- `last` 表示重写后跳到 `server` 块再次用重写后的地址匹配，用于请求路径发生改变的常规需求。

  例如将 `http://127.0.0.1/request/getlist` 的请求代理到 `http://127.0.0.1/api/getlist` 上。

### 具体应用

参考资料：<https://cloud.tencent.com/developer/article/1661636?from=15425&areaSource=102001.2&traceId=cYeKUzFfnuMHEM1pgHD2_>

```nginx
location / {
    # 指定服务器的根目录，也就是前端项目的文件地址
    root html/dist;

    # 指定网站初始页
    index index.html index.htm;

    # 需要指向下面的 @router
    try_files $uri $uri/ @router;
}

# 主要原因是路由的路径资源并不是一个真实的路径，所以无法找到具体的文件
# 因此需要 rewrite 到 index.html 中，然后交给前端路由再处理请求资源
location @router {
	rewrite ^.*$ /index.html last;
}
```

## 内置变量

|变量               |说明                                           |
|-------------------|-----------------------------------------------|
| $host             | 请求行的主机名，或请求头字段 `Host` 中的主机名 |
| $http_user_agent  | 客户端 `agent` 信息                           |
| $http_cookie      | 客户端 `cookie` 信息                          |
| $remote_addr      | 客户端的 IP 地址                              |
| $remote_port      | 客户端的端口                                  |
| $remote_user      | 已经经过 `Auth Basic Module` 验证的用户名     |
| $request_uri      | 包含请求参数的原始 URI                        |
| $request_method   | 客户端请求的动作，如 GET 或 POST              |
| $uri              | 不带请求参数的当前 URI，当用户请求 <http://localhost/example> 时，$uri 就是 `/example` |
| $document_uri     | 与$uri相同                                    |
| $args             | 请求行中的参数，同 `$query_string`            |
| $content_length   | 请求头中的 `Content-length`字段               |
| $content_type     | 请求头中的 `Content-Type` 字段                |
| $document_root    | 当前请求在 `root` 指令中指定的值              |
| $limit_rate       | 可以限制连接速率的变量                        |
| $request_filename | 当前请求的文件路径                            |
| $scheme           | HTTP方法（如http，https）                     |
| $server_protocol  | 请求使用的协议，如HTTP/1.0或HTTP/1.1          |
| $server_addr      | 服务器地址                                    |
| $server_name      | 服务器名称                                    |
| $server_port      | 请求到达服务器的端口号                        |

## 常用模板

### 80 端口转发本地 devServer

```nginx
server {
        listen 80;
        server_name hahaha.com;
        proxy_set_header Host $host;
        proxy_connect_timeout 600;
        proxy_read_timeout 600;
        proxy_send_timeout 600;
	    charset koi8-r;
        access_log logs/host.access.log format;

        location / {
        	# 将 80 指向 8009
	        proxy_pass http://127.0.0.1:8009;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_set_header X-Nginx-Proxy true;
            proxy_redirect off;
            # 允许跨域的源（$http_origin 动态获取请求客户端请求的域，不用 * 的原因是带 cookie 的请求不支持 * 号）
            add_header Access-Control-Allow-Origin 'http://127.0.0.1:8009';
            # add_header Access-Control-Allow-Origin *;

            # 表示请求头的字段 动态获取
            add_header Access-Control-Allow-Headers $http_access_control_request_headers;

            # 指定允许跨域的方法，* 代表所有
            # add_header Access-Control-Allow-Methods GET,POST,OPTIONS;
            add_header Access-Control-Allow-Methods *;

            # 预检命令的缓存，如果不缓存每次会发送两次请求
            add_header Access-Control-Max-Age 7200;

            # CORS 预检请求不能包含凭据。预检请求的响应必须指定 true 来表明可以携带凭据进行实际的请求。
            add_header Access-Control-Allow-Credentials true;
        }

    	# 后端接口代理
        location /api/ {
            proxy_pass http://192.168.1.10;
        }

        # redirect server error pages to the static page /50x.html
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root html;
        }
    }
```

### 443 端口转发本地 devServer

```nginx
# HTTPS server
server {
    listen 443 http2 ssl;
    server_name www.baidu.com;
    ssl_certificate D:/ssl/server.crt;
    ssl_certificate_key D:/ssl/server.key;
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 5m;
    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-Nginx-Proxy true;
        proxy_pass https://127.0.0.1:8080;
        proxy_redirect off;
    }
}
```

### 本地打包部署测试

```nginx
server {
    # 配置完 server_name, 记得配置 host, 然后访问 http://server_name:listen
    server_name hahaha.com;
    listen 8080;
    proxy_set_header Host $host;
    charset koi8-r;
    access_log logs/host.access.log format;

    # 打包部署
    location / {
        # 将打包好的项目放入 Nginx 根目录下的 html 目录中
        root html/dist;
        index index.html index.htm;
        try_files $uri $uri/ @router;
    }
    location @router {
        rewrite ^.*$ /index.html last;
    }

    # 后端反向代理
    location /api/ {
        proxy_pass http://qu-live-api.u.sdo.com;
        proxy_set_header Host $host;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect default;
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Headers X-Requested-With;
        add_header Access-Control-Allow-Methods GET,POST,OPTIONS;
    }

    # redirect server error pages to the static page /50x.html
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
}
```

## 常见 FAQ

### Windows 系统启动 Nginx 报错

```shell
CreateDirectory() "D:\nginx-1.18.0/temp/client_body_temp" failed (3: The system cannot find the path specified)
```

解决办法：

1.检查 nginx 的目录是否存在中文 ，路径不可以有中文。

2.在 nginx 目录中没有 `temp` 文件夹，可以手动创建一个空文件夹，名为 `temp` 即可。

### "user" is not supported

```shell
[warn] 44860#10952: "user" is not supported, ignored in D:\nginx-1.18.0/conf/nginx.conf:4
```

`user` 指令在 Windows 上不生效，请将 `user` 指令注释掉。

### invalid event type "epoll"

操作系统不支持 epoll 事件驱动机制，会出现以上错误。

1. 换成其他的事件模型，例如 `poll` 或 `select`；
2. 直接注释掉，默认情况下 Nginx 会找出最适合系统的事件模型。
