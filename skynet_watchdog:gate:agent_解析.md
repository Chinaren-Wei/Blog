# Skynet watchdog/gate/agent 解析

按照官方概述，watchdog、gate、agent 大概是下面这个意思：
> gate 服务。它的特征是监听一个 TCP 端口，接受连入的 TCP 连接，并把连接上获得的数据转发到 skynet 内部。Gate 可以用来消除外部数据包和 skynet 内部消息包的不一致性。外部 TCP 流的分包问题，是 Gate 实现上的约定。

> watchdog 模式，由 gate 加上包头，同时处理控制信息和数据信息的所有数据;  

> agent 模式，让每个 agent 处理独立连接;  

> broker 模式，由一个 broker 服务处理不同连接上的所有数据包。  

> 无论是哪种模式，控制信息都是交给 watchdog 去处理的，而数据包如果不发给 watchdog 而是发送给 agent 或 broker 的话，则不会有额外的数据头（也减少了数据拷贝）。识别这些包是从外部发送进来的方法是检查消息包的类型是否为 PTYPE_CLIENT 。当然，你也可以自己定制消息类型让 gate 通知你。



### watchdog 模块  
watchdog 的主要功能是管理用户和 agent 服务的链接。

用一个 table 保存了用户和 agent 服务的连接信息
<pre>
local agent = {
	socket1 -> agent_service1,
	socket2 -> agent_service2,
	…
}
</pre>

主要提供了 2 个方法：  

1) 创建一个 agent service 来响应新进入的请求

<pre>
function SOCKET.open(fd, addr)
    agent[fd] = skynet.newservice("agent")
    skynet.call(agent[fd], "lua", "start", gate, fd)
end
</pre>

2) 关闭连接
<pre>
function SOCKET.close(fd)
    print("socket close",fd)
    close_agent(fd)
end
</pre>

examples 里面的 watchdog 还提供了启动 gate 的方法，但核心功能也就上面这些。

### gate 模块
gate 模块的功能大概可以分为 2 部分：  
1）通过 CMD 接口初始化 gate 以及gate 上的连接的管理  

gate bind 某个端口，监听 tcp 链接
<pre>
function CMD.open( source , conf )
    local address = conf.address or "0.0.0.0"
    local port = assert(conf.port)
    ...
    socket = socketdriver.listen(address, port)
    socketdriver.start(socket)
end	
</pre>

gate 用一个 table 保存连接信息
<pre>
	local connection = {}   -- fd -> connection : { fd , client, agent , ip, mode }
	local forwarding = {}   -- agent -> connection
</pre>

gate 只负责初始化服务链接，相关数据全部以转发(forward)给相应的 agent 服务来处理。
<pre>
function CMD.forward(source, fd, client, address)
   	...
    forwarding[c.agent] = c
end
</pre>

提供接口，踢掉相关连接
<pre>
function CMD.kick(source, fd)
    local c
    if fd then
        c = connection[fd]
    else
        c = forwarding[source]
    end

   	...
    if c.mode ~= "close" then
        c.mode = "close"
        socketdriver.close(c.fd)
    end
end
</pre>

2) 通过 MSG 相关接口来创建 tcp 连接和转发连接上的数据。  
创建一个新的 tcp 连接
<pre>
function MSG.open(fd, msg)
	...
    local c = {
        fd = fd,
        ip = msg,
    }
    connection[fd] = c
	...
    skynet.send(watchdog, "lua", "socket", "open", fd, msg)
end
</pre>

转发 tcp 数据给 agent 服务
<pre>
function MSG.data(fd, msg, sz)
    -- recv a package, forward it
    local c = connection[fd]
    local agent = c.agent
    if agent then
        skynet.redirect(agent, c.client, "client", 0, msg, sz)
    else
        skynet.send(watchdog, "lua", "socket", "data", netpack.tostring(msg, sz))
    end
end
</pre>

### agent 模块
agent 模块就是实际的业务处理模块，examples 里面的仅仅是把参数转发给 simpledb 服务，然后返回结果给最终用户。


### 接收一个 tcp 的流程
gate 启动并绑定服务端口后，此时 client 发一个 tcp 连接到 gate。

1) socket_server 收到 tcp 连接后，会发送一个连接消息给 gate。

2) gate 通过 MSG.open 函数处理该消息，然后给 watchdog 发送一个 open 消息
<pre>
function MSG.open(fd, msg)
    if client_number >= maxclient then
        socketdriver.close(fd)
        return
    end
    local c = {
        fd = fd,
        ip = msg,
    }
    connection[fd] = c
    client_number = client_number + 1
    skynet.send(watchdog, "lua", "socket", "open", fd, msg)
end
</pre>

3) watchdog 收到 open 消息，调用 SOCKET.open 创建一个 agent 服务器节点来处理该 tcp 连接。
<pre>
function SOCKET.open(fd, addr)
    agent[fd] = skynet.newservice("agent")
    skynet.call(agent[fd], "lua", "start", gate, fd)
end
</pre>




