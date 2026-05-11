# 阿里云ESA-IX 单443入口对接N个落地 小白可视化全套文档
## 一、整体架构
同一ESA IX入口 + 统一443端口 → Nginx反代gRPC → 3x-ui → 路由分流 → 多个不同落地节点
全程宝塔+3x-ui面板操作，无复杂命令行，支持多用户多落地分流。

## 二、IX服务器操作步骤
### 1. 安装3x-ui
终端执行：
```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

### 2. 3x-ui创建VLESS gRPC节点
- 协议：VLESS
- 端口：43999
- 传输：gRPC
- Service Name：Download
- **不配置TLS、不搞证书**，其余默认保存

### 3. 安装宝塔国际版
```bash
URL=https://www.aapanel.com/script/install_panel_en.sh && if [ -f /usr/bin/curl ];then curl -ksSO $URL ;else wget --no-check-certificate -O install_panel_en.sh $URL;fi;bash install_panel_en.sh ipssl
```
等待约10分钟安装完成。

### 4. 宝塔基础设置
1. 网站 → 安装Nginx（安装了半个世纪）
2. 安全 → 放行端口：43999
3. 网站 → 添加站点（备案域名解析到IX IP）
4. 给站点申请SSL，开启**强制HTTPS**

### 5. 站点Nginx配置（自己看着点修改，不要复制粘贴）
```nginx
server
{
    listen 80;
    listen 443 ssl http2;
    server_name 你的域名;
    index index.php index.html index.htm default.php default.htm default.html;
    root /www/wwwroot/你的域名;
    include /www/server/panel/vhost/nginx/extension/你的域名/*.conf;

    ssl_certificate    /www/server/panel/vhost/cert/你的域名/fullchain.pem;
    ssl_certificate_key    /www/server/panel/vhost/cert/你的域名/privkey.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers EECDH+CHACHA20:不要复制、不要复制！:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_tickets on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    if ($server_port !~ 443){
        rewrite ^(/.*)$ https://$host$1 permanent;
    }

    location /Download {
        grpc_pass grpc://127.0.0.1:43999;
        grpc_set_header Host $host;
    }

    include /www/server/panel/vhost/nginx/well-known/你的域名.conf;
    
    location ~ \.well-known{
        allow all;
    }

    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
    {
        expires 30d;
        error_log /dev/null;
        access_log /dev/null;
    }

    location ~ .*\.(js|css)?$
    {
        expires 12h;
        error_log /dev/null;
        access_log /dev/null; 
    }

    access_log  /www/wwwlogs/你的域名.log;
    error_log  /www/wwwlogs/你的域名.error.log;
}
```

### 6. 伪装网页设置
1. 网站 → 站点后面文件夹，进入根目录
2. 删除所有默认文件
3. 新建 `index.html`
4. 用AI生成云盘/静态伪装页粘贴进去，底部最好加上备案号

## 三、阿里云ESA配置
1. ESA版本：至少**基础版**
2. 添加已备案域名（可租用）
3. DNS添加记录：
   - 类型：CNAME
   - 主机记录：随便填
   - 记录值：填IX宝塔站点域名
   - 回源HOST：跟随源站域名
4. 边缘证书：上传域名SSL证书
5. 缓存设置：
   - 多级缓存：边缘缓存层+区域缓存层
   - 浏览器缓存：不缓存
   - 边缘缓存：不缓存
   - 请求类型：所有传入请求
   - 缓存资格：绕过缓存
6. 网络优化：
   - 速度优化：全部打开
   - 开启IPv6、关闭WebSocket、开启gRPC
   - 智能路由：关闭
7. 其余全部默认不动

## 四、3x-ui 出站+路由分流
### 1. 添加落地出站
Xray设置 → 出站设置 → 添加出站
- 填好落地协议信息
- Tag 命名：最好是ip
- 保存 → 重启Xray

### 2. 路由规则配置
路由规则 → 添加规则
- Use：用户邮箱（入站列表→编辑里查看）
- Inbound tags：选43999
- Outbound tag：选对应落地Tag
- 保存 → 重启Xray

### 3. 新增用户
入站列表 → 对应VLESS节点菜单 → 添加客户端
只加用户，**不新建协议**
不指定出站默认走本地，配路由自动走指定落地

## 五、v2rayN客户端设置
- 地址：ESA备案域名
- 端口：443
- 传输：gRPC
- 安全：TLS
- 跳过证书验证：true
- 保存测试延迟，通即可用

## 六、多落地扩展方法
同一个IX、同一个443端口不变
1. 3x-ui多添加几个出站，每个对应一个落地
2. 新建多条路由规则，不同用户绑定不同出站Tag
3. 实现单入口、多用户、多落地分流

## 七、写在最后
我也是个小白，只是自己一直在用这个方式，有什么意见我也不懂
就这个Guihub发布我都研究了好久 。。。
