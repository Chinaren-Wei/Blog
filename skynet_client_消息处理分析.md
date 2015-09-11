# PTYPE_CLIENT 消息处理流程分析

### gate 网关服务开启端口监听:
<pre>
        function CMD.open( source, conf )
                assert(not socket)
                local address = conf.address or "0.0.0.0"
                local port = assert(conf.port)
                maxclient = conf.maxclient or 1024
                nodelay = conf.nodelay
                skynet.error(string.format("Listen on %s:%d", address, port))
                socket = socketdriver.listen(address, port)
                socketdriver.start(socket)
                if handler.open then
                        return handler.open(source, conf)
                end
        end
</pre>	

socketdriver.listen 开启监听指定 IP 和 端口，内部实现是本服务内打开监听端口，然后把 fd 扔给 skynet 的 socket server 服务代为处理，socket server 独立占用一个操作系统线程。

socketdriver.start 设置由当前 service 来处理 socket 消息，开启 socket server 向本服务(gate)的 socket 消息投递。

### gate 网关对 socket 消息的处理

对于玩家客户端的连接或者发送数据，socket server 接收到后都会生成一个 socket 类型的消息投递到 gate 服务的信箱里面，gate 服务需要定义怎么处理这种 socket 消息。

<pre>
       skynet.register_protocol {
                name = "socket",
                id = skynet.PTYPE_SOCKET,       -- PTYPE_SOCKET = 6
                unpack = function ( msg, sz )
                        return netpack.filter( queue, msg, sz)
                end,
                dispatch = function (_, _, q, type, ...)
                        queue = q
                        if type then
                                MSG[type](...)
                        end
                end
        }
</pre>
这里 name 和 id 标识了要处理的消息类型；unpack 是发送过来的数据包的解压预处理；

dispatch 设置了 socket 消息处理函数，最终的消息处理函数，是由 socket 消息的类型来决定的 MSG\[type\](...)

共有 6 种类型的消息
<pre>
#define TYPE_DATA 1
#define TYPE_MORE 2
#define TYPE_ERROR 3
#define TYPE_OPEN 4
#define TYPE_CLOSE 5
#define TYPE_WARNING 6
</pre>

gate 网关对应的 socket 消息处理函数是:
	
	MSG.data 数据包
	MSG.more 数据包 queue 
	MSG.open 连接建立消息
	MSG.close 连接关闭消息
	MSG.error 错误消息
	MSG.warning 警告(可能数据太长，截断之类的)


### gate 服务器完成的功能
gate 服务器核心工作是管理大量的外部 tcp 长连接，比如端口监听，连接进入，连接断开，转发数据包给 agent 处理。

##### MSG.open 指令处理流程
socket server 接收到一个外部连接时，调用 MSG.open 记录当前连接的 socket fd，然后向 watchdog 发送 socket open 消息，并把 socket fd 和 socket addr 发送给 watch dog
<pre>
function handler.connect(fd, addr)
        local c = {
                fd = fd,
                ip = addr,
        }
        connection[fd] = c
        skynet.send(watchdog, "lua", "socket", "open", fd, addr)
end
</pre>

watchdog 收到 open 消息后，会根据需要创建一个 agent 服务.
<pre>
function SOCKET.open(fd, addr)
        skynet.error("New client from : " .. addr)
        -- 创建一个 agent 服务
        agent[fd] = skynet.newservice("agent")
        -- 启动 agent 服务
        skynet.call(agent[fd], "lua", "start", { gate = gate, client = fd, watchdog = skynet.self() })
end
</pre>
然后向 agent 发送 start 命令，并把 socket fd、gate 服务地址和watchdog服务地址 这 3 参数传递过去。

agent start 指令处理流程
<pre>
function CMD.start(conf)
        local fd = conf.client
        local gate = conf.gate
        WATCHDOG = conf.watchdog
		...
        client_fd = fd
        skynet.call(gate, "lua", "forward", fd)
end
</pre>
保存传递过来的 socket fd，gate和 watchdog，然后向 gate 服务发送 forward 指令，并把当前用户的 fd 作为参数回传过去。


gate forward 指令处理流程
<pre>
function CMD.forward(source, fd, client, address)
        local c = assert(connection[fd])
        unforward(c)
        c.client = client or 0
        c.agent = address or source
        forwarding[c.agent] = c
        gateserver.openclient(fd)
end
</pre>
记录 fd 和 agent 的对应关系，后续 gate 转发数据包时，根据 fd 找到对应的 agent 服务地址，将数据包转发给 agent。

gateserver.openclient 调用 socketdriver.start 开启用户 socket 数据接收。


总结：open 一个连接过程中，gate 记录连接信息，watchdog 负责 agent service 的创建和启动，agent service 启动完成后，向 gate 发送 forward 指令，让 gate 将数据包直接转发给它。


##### MSG.data 消息处理流程
这个最简单，及时转发数据包给 agent
<pre>
function handler.message(fd, msg, sz)
        -- recv a package, forward it
        local c = connection[fd]
        local agent = c.agent
        if agent then
                skynet.redirect(agent, c.client, "client", 0, msg, sz)
        else
                skynet.send(watchdog, "lua", "socket", "data", fd, netpack.tostring(msg, sz))
        end
end
</pre>
根据 fd 找到负责处理的 agent，然后把数据包 redirect 过去，消息类型为 "client"(这里是字符串，redirect 内部实现会转换为 PTYPE_CLIENT 整数类型)

### agent PTYPE_CLIENT 消息的处理
注册协议处理 PTYPE_CLIENT 类型的消息
<pre>
skynet.register_protocol {
        name = "client",
        id = skynet.PTYPE_CLIENT,
        unpack = function (msg, sz)
                return host:dispatch(msg, sz)
        end,
        dispatch = function (_, _, type, ...)
                if type == "REQUEST" then
                        local ok, result  = pcall(request, ...)
                        if ok then
                                if result then
                                        send_package(result)
                                end
                        else
                                skynet.error(result)
                        end
                else
                        assert(type == "RESPONSE")
                        error "This example doesn't support request client"
                end
        end
}
</pre>

这里 unpack 用了 sproto，unpack 后把参数传递给对应指令处理。如

get 命令处理函数(这里的 self 用法还想不太明白)
<pre>
function REQUEST:get()
        print("get", self.what)
        local r = skynet.call("SIMPLEDB", "lua", "get", self.what)
        return { result = r }
end
</pre>


### gate msg.close 处理流程
处理外部 socket 关闭消息

<pre>
-- gate 服务取消 fd 的消息转发，并向 watchdog 发送 socket close 指令
function MSG.close(fd)
        if handler.disconnect then
                handler.disconnect(fd)
        end
        close_fd(fd)
end
 
function handler.disconnect(fd)
        close_fd(fd)
        skynet.send(watchdog, "lua", "socket", "close", fd)
end


-- watchdog 给 gate 回传 kick 指令踢脚连接，关闭 socket fd；给 agent 发送 disconnect 指令，让 agent 退出(exit)。

function SOCKET.close(fd)
        print("socket close",fd)
        close_agent(fd)
end

local function close_agent(fd)
        local a = agent[fd]
        agent[fd] = nil
        if a then
                skynet.call(gate, "lua", "kick", fd)
                -- disconnect never return
                skynet.send(a, "lua", "disconnect")
        end
end
</pre>





	