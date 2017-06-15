# frp_application
使用示例

根据对应的操作系统及架构，从 <a href="https://github.com/fatedier/frp/releases">Release</a> 页面下载最新版本的程序。

将 frps 及 frps.ini 放到具有公网 IP 的机器上。

将 frpc 及 frpc.ini 放到处于内网环境的机器上。

通过 ssh 访问公司内网机器

修改 frps.ini 文件，这里使用了最简化的配置：
# frps.ini

[common]

bind_port = 7000

启动 frps：

./frps -c ./frps.ini

修改 frpc.ini 文件，假设 frps 所在服务器的公网 IP 为 x.x.x.x；

# frpc.ini

[common]

server_addr = x.x.x.x

server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
启动 frpc：
./frpc -c ./frpc.ini

通过 ssh 访问内网机器，假设用户名为 test：
ssh -oPort=6000 test@x.x.x.x

通过自定义域名访问部署于内网的 web 服务

有时想要让其他人通过域名访问或者测试我们在本地搭建的 web 服务，但是由于本地机器没有公网 IP，无法将域名解析到本地的机器，通过 frp 就可以实现这一功能，以下示例为 http 服务，https 服务配置方法相同， vhost_http_port 替换为 vhost_https_port， type 设置为 https 即可。

修改 frps.ini 文件，设置 http 访问端口为 8080：
# frps.ini
[common]
bind_port = 7000
vhost_http_port = 8080
启动 frps；
./frps -c ./frps.ini

修改 frpc.ini 文件，假设 frps 所在的服务器的 IP 为 x.x.x.x，local_port 为本地机器上 web 服务对应的端口, 绑定自定义域名 www.yourdomain.com:
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000

[web]
type = http
local_port = 80
custom_domains = www.yourdomain.com
启动 frpc：
./frpc -c ./frpc.ini

将 www.yourdomain.com 的域名 A 记录解析到 IP x.x.x.x，如果服务器已经有对应的域名，也可以将 CNAME 记录解析到服务器原先的域名。

通过浏览器访问 http://www.yourdomain.com:8080 即可访问到处于内网机器上的 web 服务。

转发 DNS 查询请求

DNS 查询请求通常使用 UDP 协议，frp 支持对内网 UDP 服务的穿透，配置方式和 TCP 基本一致。

修改 frps.ini 文件：
# frps.ini
[common]
bind_port = 7000
启动 frps：
./frps -c ./frps.ini

修改 frpc.ini 文件，设置 frps 所在服务器的 IP 为 x.x.x.x，转发到 Google 的 DNS 查询服务器 8.8.8.8 的 udp 53 端口：
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000

[dns]
type = udp
local_ip = 8.8.8.8
local_port = 53
remote_port = 6000
启动 frpc：
./frpc -c ./frpc.ini

通过 dig 测试 UDP 包转发是否成功，预期会返回 www.google.com 域名的解析结果：
dig @x.x.x.x -p 6000 www.goolge.com
加密与压缩

这两个功能默认是不开启的，需要在 frpc.ini 中通过配置来为指定的代理启用加密与压缩的功能，压缩算法使用 snappy：

# frpc.ini
[ssh]
type = tcp
local_port = 22
remote_port = 6000
use_encryption = true
use_compression = true
如果公司内网防火墙对外网访问进行了流量识别与屏蔽，例如禁止了 ssh 协议等，通过设置 use_encryption = true，将 frpc 与 frps 之间的通信内容加密传输，将会有效防止流量被拦截。

如果传输的报文长度较长，通过设置 use_compression = true 对传输内容进行压缩，可以有效减小 frpc 与 frps 之间的网络流量，加快流量转发速度，但是会额外消耗一些 cpu 资源。

服务器端热加载配置文件

由于从 v0.10.0 版本开始，所有 proxy 都在客户端配置，这个功能暂时移除。

特权模式

由于从 v0.10.0 版本开始，所有 proxy 都在客户端配置，原先的特权模式是目前唯一支持的模式。

端口白名单

为了防止端口被滥用，可以手动指定允许哪些端口被使用，在 frps.ini 中通过 privilege_allow_ports 来指定：

# frps.ini
[common]
privilege_allow_ports = 2000-3000,3001,3003,4000-50000
privilege_allow_ports 可以配置允许使用的某个指定端口或者是一个范围内的所有端口，以 , 分隔，指定的范围以 - 分隔。

TCP 多路复用

从 v0.10.0 版本开始，客户端和服务器端之间的连接支持多路复用，不再需要为每一个用户请求创建一个连接，使连接建立的延迟降低，并且避免了大量文件描述符的占用，使 frp 可以承载更高的并发数。

该功能默认启用，如需关闭，可以在 frps.ini 和 frpc.ini 中配置，该配置项在服务端和客户端必须一致：

# frps.ini 和 frpc.ini 中
[common]
tcp_mux = false
连接池

默认情况下，当用户请求建立连接后，frps 才会请求 frpc 主动与后端服务建立一个连接。当为指定的代理启用连接池后，frp 会预先和后端服务建立起指定数量的连接，每次接收到用户请求后，会从连接池中取出一个连接和用户连接关联起来，避免了等待与后端服务建立连接以及 frpc 和 frps 之间传递控制信息的时间。

这一功能比较适合有大量短连接请求时开启。

首先可以在 frps.ini 中设置每个代理可以创建的连接池上限，避免大量资源占用，客户端设置超过此配置后会被调整到当前值：
# frps.ini
[common]
max_pool_count = 5
在 frpc.ini 中为客户端启用连接池，指定预创建连接的数量：
# frpc.ini
[common]
pool_count = 1
修改 Host Header

通常情况下 frp 不会修改转发的任何数据。但有一些后端服务会根据 http 请求 header 中的 host 字段来展现不同的网站，例如 nginx 的虚拟主机服务，启用 host-header 的修改功能可以动态修改 http 请求中的 host 字段。该功能仅限于 http 类型的代理。

# frpc.ini
[web]
type = http
local_port = 80
custom_domains = test.yourdomain.com
host_header_rewrite = dev.yourdomain.com
原来 http 请求中的 host 字段 test.yourdomain.com 转发到后端服务时会被替换为 dev.yourdomain.com。

通过密码保护你的 web 服务

由于所有客户端共用一个 frps 的 http 服务端口，任何知道你的域名和 url 的人都能访问到你部署在内网的 web 服务，但是在某些场景下需要确保只有限定的用户才能访问。

frp 支持通过 HTTP Basic Auth 来保护你的 web 服务，使用户需要通过用户名和密码才能访问到你的服务。

该功能目前仅限于 http 类型的代理，需要在 frpc 的代理配置中添加用户名和密码的设置。

# frpc.ini
[web]
type = http
local_port = 80
custom_domains = test.yourdomain.com
http_user = abc
http_pwd = abc
通过浏览器访问 http://test.yourdomain.com，需要输入配置的用户名和密码才能访问。

自定义二级域名

在多人同时使用一个 frps 时，通过自定义二级域名的方式来使用会更加方便。

通过在 frps 的配置文件中配置 subdomain_host，就可以启用该特性。之后在 frpc 的 http、https 类型的代理中可以不配置 custom_domains，而是配置一个 subdomain 参数。

只需要将 *.{subdomain_host} 解析到 frps 所在服务器。之后用户可以通过 subdomain 自行指定自己的 web 服务所需要使用的二级域名，通过 {subdomain}.{subdomain_host} 来访问自己的 web 服务。

# frps.ini
[common]
subdomain_host = frps.com
将泛域名 *.frps.com 解析到 frps 所在服务器的 IP 地址。

# frpc.ini
[web]
type = http
local_port = 80
subdomain = test
frps 和 fprc 都启动成功后，通过 test.frps.com 就可以访问到内网的 web 服务。

需要注意的是如果 frps 配置了 subdomain_host，则 custom_domains 中不能是属于 subdomain_host 的子域名或者泛域名。

同一个 http 或 https 类型的代理中 custom_domains 和 subdomain 可以同时配置。

URL 路由

frp 支持根据请求的 URL 路径路由转发到不同的后端服务。

通过配置文件中的 locations 字段指定一个或多个 proxy 能够匹配的 URL 前缀(目前仅支持最大前缀匹配，之后会考虑正则匹配)。例如指定 locations = /news，则所有 URL 以 /news 开头的请求都会被转发到这个服务。

# frpc.ini
[web01]
type = http
local_port = 80
custom_domains = web.yourdomain.com
locations = /

[web02]
type = http
local_port = 81
custom_domains = web.yourdomain.com
locations = /news,/about
按照上述的示例配置后，web.yourdomain.com 这个域名下所有以 /news 以及 /about 作为前缀的 URL 请求都会被转发到 web02，其余的请求会被转发到 web01。

通过代理连接 frps

在只能通过代理访问外网的环境内，frpc 支持通过 HTTP PROXY 和 frps 进行通信。

可以通过设置 HTTP_PROXY 系统环境变量或者通过在 frpc 的配置文件中设置 http_proxy 参数来使用此功能。

# frpc.ini
server_addr = x.x.x.x
server_port = 7000
http_proxy = http://user:pwd@192.168.1.128:8080
