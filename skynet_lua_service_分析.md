# Skynet Lua Service 分析

#### Lua Service 的启动
examples 中， config.start 选项配置 lua service 启动入口，即：examples/main.lua，它完成的功能很简单，把相关服务逐个启动起来，代码如下：  

```lua 
	skynet.start(function()
		print("Server start")
		local service = skynet.newservice("service_mgr")
		skynet.monitor "simplemonitor"
		local console = skynet.newservice("console")
		skynet.newservice("simpledb")
		local watchdog = skynet.newservice("watchdog")
		skynet.call(watchdog, "lua", "start", {
			port = 8888,
			maxclient = max_client,
		})

		skynet.exit()
	end)
```

所有服务都是通过函数 skynet.newservice 启动的，该函数的参数就是包含该服务的 lua 文件名，如 skynet.newservice("simpledb") 对应的 lua 源文件为 examples/simpledb.lua。

***skynet.newservice 函数:***
```lua
function skynet.newservice(name, ...)
    local param =  table.concat({"snlua", name, ...}, " ")
    local handle = skynet.call(".launcher", "text" , param)
    ...
end
```
对于 simpledb，最终的函数调用应该是 skynet.call(".launcher", "text" , "snlua simpledb")

*** skynet.call 函数：***

```lua
function skynet.call(addr, typename, ...)
    local p = proto[typename]
    ...
    local session = c.send(addr, p.id , nil , p.pack(...))
    ...
    return p.unpack(yield_call(addr, session))
end
```
调用 c.send 函数(一个 c 封装)向名字为 ".launcher" 的服务发发送一个消息，消息参数为 snlua simpledb

.lauhcher 是一个 lua service 的管理器，封装了 service 的START、GC、RELOAD等功能；

.launcher 收到参数为 snlua simpledb 的消息时，调用 skynet.launch(service, param) 函数， 最终调用 c.command("LAUNCH", table.concat({...}," ")) 启动服务。

c.command 是一个 skynet 底层函数的 c 语言的封装，详细见 lua-skynet.c 的 _command 函数。

```c
static int
_command(lua_State *L) {
    struct skynet_context * context = lua_touserdata(L, lua_upvalueindex(1));
    const char * cmd = luaL_checkstring(L,1);
    const char * result;
    const char * parm = NULL;
    if (lua_gettop(L) == 2) {
        parm = luaL_checkstring(L,2);
    }    
    ....

    result = skynet_command(context, cmd, parm);
    ...
}
```
***parm：*** 刚才传递进来的参数串 snlua simpledb  
***cmd：*** 字符串 LAUNCH

然后进入 ***skynet_command*** 函数
```c
const char *
skynet_command(struct skynet_context * context, const char * cmd , const char * param) {
  ...
    if (strcmp(cmd,"LAUNCH") == 0) {
        size_t sz = strlen(param);
        char tmp[sz+1];
        strcpy(tmp,param);
        char * args = tmp;
        char * mod = strsep(&args, " \t\r\n");
        args = strsep(&args, "\r\n");
        struct skynet_context * inst = skynet_context_new(mod,args);
        if (inst == NULL) {
            return NULL;
        } else {
            _id_to_hex(context->result, inst->handle);
            return context->result;
        }
    }  
  ...
}

```
函数 skynet_context_new 实际的 lua service 的加载任务，然会返回成功加载服务器的 handle，也就是 context->result。

skynet_context_new  函数比较复杂，待下一步详解。






