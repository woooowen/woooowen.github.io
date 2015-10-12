---
layout: post
title: 使用ngx_lua + redis 开发自定义跳转功能
category: [Lua,redis]
tags: [Lua,redis]
---

###使用 ngxin , lua ,redis 开发自定义跳转功能,当网站获取到请求A时,立刻跳转到返回B


#### nginx不必多说,nginx的第三方Lua模块也有很多.redis 也是,譬如: lua-resty-redis,ngx_redis2等等

####相关地址

<https://www.nginx.com/resources/wiki/modules/lua/>

<https://github.com/openresty/lua-resty-redis>

####为了简化整个项目,只描述整体流程

![rewrite](http://www.woowen.com/public/image/rewrite.png)

1.nginx加载lua模块.这个不管是直接使用open-resty,还是使用ngx_lua,都可以.而luajit,速度据说比lua原生效率提高了10倍,这个我也没有测试过.具体的安装gg上面都有不少.这里不细说.安装nginx,ngx_lua,以及lua-resty-redis,如果你使用open-resty,那么里面整合了很多你可能会用到的东西,例如:lua-resty-mysql,lua-resty-redis,lua-resty-memcached等等.

2.在ngxin 的配置文件中,运行lua脚本.

```lua

	-- nginx.conf
 	server {
        listen       80;
        server_name  localhost;

        #这个设置off,会有提醒,每次修改lua脚本都会直接显示最新的结果,方便调试,线上环境需要开启
        lua_code_cache off;                 

		#这里使用的ngx_lua,但是redis模块加载了lua-resty-redis,如果你使用open-resty那么可以不用引入直接使用.链接(1)中有详细说明
        lua_package_path "/Users/james/Tool/redis.lua"; 
        

        #access_log  logs/host.access.log  main;
                        
        location /lua {
                default_type text/plain;
        		content_by_lua 'ngx.say("hello world")';        
        }

        location /dump {
                default_type text/plain;
                content_by_lua_file /usr/local/nginx/conf/lua_script/dump.lua;
        }

        #你可以直接在配置文件中运行lua代码,也可以加载lua文件
    }

```

####链接(1): <https://github.com/openresty/lua-resty-redis#installation>

3.编写lua脚本

```lua
	
	--这只是简单的demo,具体复杂还需要根据业务调整
	--加载第三方redis插件
	local redis = require "resty.redis"
	local rcache = redis:new()

	--redis连接
	local ok,err = rcache:connect("127.0.0.1",6379)

	if not ok then
	        ngx.say("failed to connect redis cache :",err);
	        return
	end
	-- 当前访问地址
	local url = ngx.var.uri
	-- 主机域名
	local host = ngx.var.host
	-- 链接参数
	local args = ngx.var.args

	--获取redis中的值
	local res, err = rcache:hget("rewrite",url)
	if res then
	--如果获取到了对应的链接,那么直接跳转
	        ngx.redirect(res)
	end	
	ngx.req.set_uri(url)

```

####获取redis,如果害怕流量全部落到一个实例上,可以通过LRU,或者随机方式来进行一定的负载均衡,防止cache被击穿,另外我们的redis采用主动缓存,所以不存在lifetime过期,也就不会有Dog-Pile,也可以通过锁的形势.

参考资料: <https://github.com/openresty/lua-resty-lock#for-cache-locks>

####redis采用hash结构,保证get-key的时候复杂度为O(1),避免多次循环,也可以使用ngx.shared.DICT方式,将数据放在内存中,只适合量不大,且不用经常去更新的内容.ngx.shared.DICT只支持简单的key-value.

####然后添加一个后台管理你的redis就可以了.
