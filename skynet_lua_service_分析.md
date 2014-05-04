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

#### snlua 模块解析
整个模块由 service_snlua.c 实现，编译完成后，以 so 的形式存在运行目录中，和 so 类型的服务接口一样。

*** snlua_create 接口***
```c
struct snlua *
snlua_create(void) {
    struct snlua * l = skynet_malloc(sizeof(*l));
    memset(l,0,sizeof(*l));
    l->L = lua_newstate(skynet_lalloc, NULL);
    l->init = _init;
    return l;
}

```
创建一个新的 luastate，返回 snlua 结构。

snlua_init 接口
```c
int
snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args) {
    int sz = strlen(args);
    char * tmp = skynet_malloc(sz+1);
    memcpy(tmp, args, sz+1);
    skynet_callback(ctx, l , _launch);
    const char * self = skynet_command(ctx, "REG", NULL);
    uint32_t handle_id = strtoul(self+1, NULL, 16);
    // it must be first message
    skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz+1);
    return 0;
}
```
很多行，但其实只干了一件事，调用 skynet_send 给自己发个消息，让函数 _launch 处理该消息。

而 _launch 函数又调用了 _init 函数

_init 函数 
```c
static int
_init(struct snlua *l, struct skynet_context *ctx, const char * args) {
    lua_State *L = l->L;
    ...
    // 初始化 lua 环境，设置 codecache 等
    lua_gc(L, LUA_GCSTOP, 0);
    luaL_openlibs(L);
    lua_pushlightuserdata(L, l);
    ...
    ...
    // 调用 _load 函数，完成实际 lua 代码的加载
    int r = _load(L, &parm);
    
    ...
    // 执行加载完成的 lua 文件
    r = lua_pcall(L,n,0,traceback_index);
    ...

```
_load 处理下 lua service path 的问题，再调用 _try_load 

_try_load 函数
```c
static int
_try_load(lua_State *L, const char * path, int pathlen, const char * name) {
    ...
    int r = luaL_loadfile(L,tmp);
    if (r == LUA_OK) {
	...
        luaL_Buffer b;
        luaL_buffinit(L, &b);
        ...
    }
}

```
调用 luaL_loadfile 加载 lua 源文件并设置相关环境。

#### 利用 snlua 创建 lua service
以 simpledb 为例，创建一个 simpledb 的服务：skynet_context_new("snlua", "simpledb")  

skynet_context_new 函数:
```c
skynet_context_new(const char * name, const char *param) {
    /* 利用 dlopen 加载 snlua.so ，创建一个新 mod 结构。
    	mod->create   -->  snlua_create
    	mod->init     -->  snlua_init
    	mod->release  --> snlua_release
    */
    struct skynet_module * mod = skynet_module_query(name);
    ...
    
    // 本质上是调用 snlua_create
    void *inst = skynet_module_instance_create(mod);
    ...
    // 创建新的 ctx 对象
    struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));
    ...
    ctx->mod = mod;
    ctx->instance = inst;
    ...
    // 注册 service，设置对应的 service handle
    ctx->handle = skynet_handle_register(ctx);

    //创建该 service 的私有消息队列
    struct message_queue * queue = ctx->queue = skynet_mq_create(ctx->handle);
    ...
    // 本质上调用 snlua_init，完成 simpledb 服务加载
    int r = skynet_module_instance_init(mod, inst, ctx, param);
    ...   
}
```



