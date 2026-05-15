# 阿里云ESA-IX
# 单入口N用户对接N个落地
## 一、整体架构
**ESA边缘入口（443端口）→ 宝塔Nginx反向代理gRPC（xhttp） → 3x-ui入站 → 自定义路由分流 → 多个独立落地节点**
全程**宝塔面板 + 3x-ui面板**可视化操作，支持**单域名单443端口、多用户、多落地独立分流**

---

## 二、IX中转服务器
### 1. 一键安装 3x-ui
直接复制到服务器终端执行：
```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```
安装完记住面板端口、账号密码

### 2. 3x-ui 创建 VLESS-gRPC（xhttp） 入站节点
1. 登录3x-ui → 入站列表 → 新建入站
2. 参数按下面填：
   - 协议：**VLESS**
   - 监听端口：**43999**
   - 传输方式：**gRPC**（xhttp）
   - Service Name：`Download`
   - **关闭TLS、不配置证书、不开启加密**
   - 其余保持默认，直接保存

### 3. 一键安装 宝塔国际版
直接复制到服务器终端执行：
```bash
URL=https://www.aapanel.com/script/install_panel_en.sh && if [ -f /usr/bin/curl ];then curl -ksSO $URL ;else wget --no-check-certificate -O install_panel_en.sh $URL;fi;bash install_panel_en.sh ipssl
```
等待10分钟左右安装完成，保存宝塔登录地址、账号密码。

### 4. 宝塔设置
1. 网站 → 安装 **Nginx**（耗时半个世纪）
2. 安全 → 防火墙 → **放行全部端口**
3. 网站 → 添加站点
   - 域名：解析到IX服务器IP的域名（可用非备案域名做源站）
   - 运行目录默认，PHP版本选纯静态即可
4. 站点设置 → SSL → 申请免费Let’s Encrypt证书，**开启强制HTTPS**

### 5. Nginx 配置
进入宝塔站点 → 配置文件 **添加核心配置：VLESS 反向代理**
```nginx
server
{
    listen 80;
    listen 443 ssl http2; # 必须开启 http2
    server_name 你的域名;
    index index.php index.html index.htm default.php default.htm default.html;
    root /www/wwwroot/你的域名;
    include /www/server/panel/vhost/nginx/extension/你的域名/*.conf;

    # SSL 证书路径（保持你原有的不变）
    ssl_certificate    /www/server/panel/vhost/cert/你的域名/fullchain.pem;
    ssl_certificate_key    /www/server/panel/vhost/cert/你的域名/privkey.pem;
    
    # 协议优化：TikTok 直播建议只留 TLS 1.2 和 1.3
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers EECDH+CHACHA20:这里保留你自己的:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_tickets on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # 强制 HTTPS 跳转
    if ($server_port !~ 443){
        rewrite ^(/.*)$ https://$host$1 permanent;
    }

    # ----------------------------------------------------------------
    # 核心配置：VLESS 反向代理
    # ----------------------------------------------------------------
   location /Download { # 此路径必须与 3X-UI 中填写的 serviceName 完全一致
        grpc_pass grpc://127.0.0.1:43999; # 转发到 3X-UI 监听的本地端口
        grpc_set_header Host $host;
        # ... 其他可选 headers
    }
    # ----------------------------------------------------------------

    # 以下保持你原有的安全与静态资源配置
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
```
xhttp使用：
```
    # ----------------------------------------------------------------
    # 核心配置：VLESS 反向代理
    # ----------------------------------------------------------------
   location /Download { # 此路径必须与 3X-UI 中填写的 serviceName 完全一致
        proxy_pass http://127.0.0.1:43999; # 转发到 3X-UI 监听的本地端口
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        gproxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
```
改完**重载Nginx**

### 6. 伪装静态网页部署
1. 宝塔网站 → 对应站点 → 进入根目录文件夹
2. **删除系统默认所有文件**
3. 新建 `index.html`
4. AI一个云盘/博客/企业/视频站伪装页代码
5. 在网页底部加添加备案信息

---

## 三、阿里云 ESA 边缘加速配置（逐行照做）
1. ESA 版本要求：**基础版及以上**
2. 域名管理：添加**已备案域名**
3. DNS → 记录：
   - 添加记录
   - 记录类型：**CNAME**
   - 主机记录：自定义（比如 `ix`）
   - 记录值/源站：选择**域名**
   - 域名：**IX宝塔站点域名**
   - 回源HOST：跟随源站域名
   - 接入方式：选 **API加速** 完成接入
   - 备案域名解析到ESA提供的CNAME地址
4. SSL/TLS → 边缘证书：
   - 申请免费证书
   - 开启强制HTTPS
   - TLS 加密套件与协议版本 TLSv1.2 TLSv1.3
5. 缓存：
   - 配置 → 
   - 浏览器缓存过期时间：**不缓存**
   - 边缘缓存过期时间：**不缓存**
   - 多级缓存 → 
   - 边缘缓存层+区域缓存层
6. 规则 → 缓存规则：
   - 新建规则 → 所有传入请求
   - 缓存资格：**绕过缓存**
7. 速度和网络 → 优化：
   - 速度优化：全部开关打开
   - 网络优化：开启IPv6、关闭WebSocket、开启gRPC（xhttp可关闭）
   - 最大上传大小：改为 **500M**
8. 流量 → 智能路由：**直接关闭**
9. 其余默认不动，保存配置等待生效

---

## 四、3x-ui 出站添加 + 路由分流
### 1. 添加落地节点出站
1. 3x-ui → Xray设置 → 出站设置 → 添加出站
2. 填入落地节点协议、地址、端口、密钥等信息
3. **Tag 自定义命名**（建议用落地IP方便区分）
4. 保存 → 重启Xray服务

> 有多少个落地，就添加多少条出站

### 2. 路由规则绑定用户与落地
1. 路由规则 → 添加规则
2. 关键参数：
   - Use：用户邮箱（入站客户端里查看）
   - Inbound tags：选中 `43999`
   - Outbound tag：选中要绑定的落地出站Tag
3. 保存 → 重启Xray

### 3. 新增客户端用户
1. 入站列表 → VLESS:43999节点 → 添加客户端
2. 只新增用户，**不用新建协议端口**
3. 不指定路由默认走本地，配置规则后自动走绑定落地

---

## 五、v2rayN / Shadowrocket 客户端配置
### v2rayN 参数
- 地址：**ESA 备案域名**
- 端口：`443`
- 协议：VLESS
- 传输：**gRPC**
- ServiceName：`Download`
- 安全：TLS
- 跳过证书验证：**开启true**
- 其他默认，保存后测延真延迟即可

### Shadowrocket
直接在3x-ui客户端右键**分享**，扫码自动导入即可使用。

---

## 六、协议转换会增加延迟 以下可选
### 多用户单落地可选
IX的443端口转发到落地的443端口，宝塔和3x-ui在落地机完成。
### 多用户多落地可选
不添加出站，直接IX落地，其他协议使用代理通过。
