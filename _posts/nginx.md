NGINX关于TCP的优化基本上都是修改系统内核提供的配置项，所以跟具体的Linux版本和系统配置有关。

### sendfile

syscall, 不需要先read再write，没有context切换的代价，0拷贝技术

如果是作为静态文件服务器,sendfile可以明显加速；如果是作为反向代理，那可以不激活它。

### tcp_nodelay

强制socket发送缓冲区的数据，不管包的大小。

为了避免网络拥塞，TCP栈实现了一个机制，为避免发送的包太小，等待0.2s再发送。（Nagle's算法——在发出去的数据还未被确认之前，新生成的小数据先存起来，凑满一个MSS或者等到收到确认后再发送）

Unix是200ms

是一个socket选项，启用后会禁用Nagle算法。

NGINX只会针对处于keepalive状态的TCP连接才会启用tcp_nodelay。

### tcp_nopush

优化每次传输的数据包大小，数据包会累计到一定大小后才会发送，减小了额外开销，提高网络效率。

是FreeBSD的一个socket选项，对应Linux的TCP_CORK

必须设置sendfile

### keepalive_timeout

服务端为每个TCP连接最多可以保持多长时间，NGINX的默认值是75秒，有些浏览器最多只保持60s，所以可以统一设置为60s。

### 组合使用

1. sendfile和tcp_nopush结合，满包发送，减少网络负担，加速文件传输。
2. 可以看到TCP_NOPUSH是要等到数据包积累到一定大小才发送，TCP_NODELAY是要尽快发送，二者相互矛盾，实际上，它们可以一起使用，达到的效果是先填满包，再尽快发送。

