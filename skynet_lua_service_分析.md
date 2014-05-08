# Skynet Lua Service 运行分析

下面以 SIMPLEDB 这个服务(examples/simpledb.lua)为例，分析 lua service 的运行原理和数据流向。  
simpledb.lua 的源码非常简单：

<pre>
local command = {}

function command.GET(key)
    return db[key]
end

function command.SET(key, value)
    local last = db[key]
    db[key] = value
    return last
end

skynet.start(function()
    skynet.dispatch("lua", function(session, address, cmd, ...)
        local f = command[string.upper(cmd)]
        skynet.ret(skynet.pack(f(...)))
    end)
    skynet.register "SIMPLEDB"
end)

</pre>


该模块启动时会运行 skynet.start 函数，参数是一个匿名函数，匿名函数中：

* skynet.dispatch 设置当前服务的消息处理函数，比较明了
* skynet.register 该服务注册个名字，方便其它应用向该服务发消息。  

### skynet.start 函数

<pre>
function skynet.start(start_func)
    c.callback(dispatch_message)
    skynet.timeout(0, function()
        init_service(start_func)
    end)
end
</pre>

c.callback 是一个 c 接口，将当前服务的消息处理函数设置为 dispatch_message；实际处理消息时，dispatch_message 会调用上面 skynet.dispatch 设置的消息处理函数。dispatch_message 对 coroutine 的处理进行了一些封装，稍微有些复杂，后面会详细讲。

init_service 调用最终会把传进来的匿名函数执行一次。



#### c.callback 函数####
<pre>
_callback(lua_State *L) {
    struct skynet_context * context = lua_touserdata(L, lua_upvalueindex(1));
	// 检测参数类型是否是 lua 函数
    luaL_checktype(L,1,LUA_TFUNCTION);
    // 忽略掉多余的参数
    lua_settop(L,1);
    // 将该函数指针存储在 LUA_REGISTRYINDEX 位置
    lua_rawsetp(L, LUA_REGISTRYINDEX, _cb);
	
	// 与热更新有关
    lua_rawgeti(L, LUA_REGISTRYINDEX, LUA_RIDX_MAINTHREAD);
    lua_State *gL = lua_tothread(L,-1);
	
	// 设置 c 语言里面的消息处理函数
    skynet_callback(context, gL, _cb);

    return 0;
}
// _cb 函数

static int
_cb(struct skynet_context * context, void * ud, int type, int session, uint32_t source, const void * msg, size_t sz) {
    lua_State *L = ud;
    int trace = 1;
    int r;
    int top = lua_gettop(L);
    if (top == 1) {
    	// 取出之前设置好的 lua 消息处理函数
        lua_rawgetp(L, LUA_REGISTRYINDEX, _cb);
    } else {
        assert(top == 2);
        lua_pushvalue(L,2);
    }
	// 设置函数参数
    lua_pushinteger(L, type);
    lua_pushlightuserdata(L, (void *)msg);
    lua_pushinteger(L,sz);
    lua_pushinteger(L, session);
    lua_pushnumber(L, source);
	// 调用 lua 的消息处理函数，也就是 dispatch_message
    r = lua_pcall(L, 5, 0 , trace);

    if (r == LUA_OK) {
        return 0;
    }
    ...
}
</pre>

一个新消息进来，函数调用次序应该是这样 _cb --> dispatch_message --> skynet.dispatch 设置的处理函数


#### dispatch_message 函数 ####
<pre>
local function dispatch_message(...)
	// 这里跳转到 raw_dispatch_message
    local succ, err = pcall(raw_dispatch_message,...)
	while true do
		-- 处理 coroutine 队列，暂时忽略
		…  
end

local function raw_dispatch_message(prototype, msg, sz, session, source, ...)
    -- skynet.PTYPE_RESPONSE = 1, read skynet.h
    if prototype == 1 then
 		…
 		-- 回应报，忽略相关处理
    else
        local p = assert(proto[prototype], prototype)
        local f = p.dispatch
        if f then
            -- 对于新来的消息，创建一个新的 coroutine
            local co = co_create(f)
            session_coroutine_id[co] = session
            session_coroutine_address[co] = source
            -- 执行 coroutine.resume 开始消息处理
            suspend(co, coroutine.resume(co, session,source, p.unpack(msg,sz, ...)))
        else
            unknown_request(session, source, msg, sz)
        end
    end
end

local function co_create(f)
    -- 从 coroutine 池中取出一个空闲的 coroutine，如果没有，则创建一个新的
    local co = table.remove(coroutine_pool)
    if co == nil then
        co = coroutine.create(function(...)
            // 执行实际的消息处理，skynet.dispatch 指定的函数
            f(...)
            // 下面用于 countinue 的回收
            while true do
                f = nil
                coroutine_pool[#coroutine_pool+1] = co
                f = coroutine_yield "EXIT"
                f(coroutine_yield())
            end
        end)
    else
        coroutine.resume(co, f)
    end
    return co
end
</pre>
简单来说，服务收到的每一个消息，都会单独分配一个 coroutine 来处理。  

这块 coroutine 分配以及运行的设计方案比较复杂，基本方法都是通过 coroutine.yield 返回一些状态字符串来控制 coroutine 的运行、sleep 等控制功能。








