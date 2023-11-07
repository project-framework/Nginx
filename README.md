# Nginx

前端 Nginx 常用配置
Documentation is available at <http://nginx.org>

## 常见 FAQ

### Windows 系统启动 nginx 报错

CreateDirectory() "D:\nginx-1.18.0/temp/client_body_temp" failed (3: The system cannot find the path specified)

解决办法：

1.检查 nginx 的目录是否存在中文 ，路径不可以有中文。

2.在 nginx 目录中没有 temp 文件夹，可以手动创建一个空文件夹，名为 temp 即可。
