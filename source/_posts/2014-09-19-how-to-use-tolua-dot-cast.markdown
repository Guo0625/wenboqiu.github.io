---
layout: post
title: "何时使用tolua.cast"
date: 2014-09-19 10:45:15 +0800
comments: true
categories: 
---

之前一直没去关心这个问题，觉得只要按照代码示例照做就好。今天暂时梳理了下这个函数。

下面是tolua 手册中对这个函数的定义：
tolua.cast (var, type)：Changes the metatable of var in order to make it of type type. type needs to be a string with the complete C type of the object (including namespaces, etc).

它所对应的C语言实现在tolua_map.c中
<!--more-->
{% codeblock lang:c %}
/* Type casting
*/
static int tolua_bnd_cast (lua_State* L)
{
    void* v;
    const char* s;
    if (lua_islightuserdata(L, 1)) {
        v = tolua_touserdata(L, 1, NULL);
    } else {
        v = tolua_tousertype(L, 1, 0);
    };

    s = tolua_tostring(L,2,NULL);
    if (v && s)
        tolua_pushusertype(L,v,s);
    else
        lua_pushnil(L);
    return 1;
}
{% endcodeblock %}

我们可以看到，tolua.cast实际上是把var的类型转成目标类型，那什么时候需要用到它呢。如果我们需要在Lua中使用C++代码中传过来的CCArray对象，我们直接在Lua中调用该对象的方法即可，因为tolua已经设置该对象的metatable为CCArray类型。但是如果想要使用CCArray对象中存储的对象，比如说该数组中有2个CCString对象，我们获取其中一个对象时，tolua会把该对象设置为CCObject类型，这时候就需要我们把该对象转为CCString类型，才能使用CCString的方法。tolua.cast其实就像C++的dynamic_cast，用于类层次间的上行转换和下行转换，在执行期决定真正的类型。

下面是Lua代码示例，希望有助于理解。
{% codeblock lang:lua %}
--commands 是 一个CCArray对象，里面存着CCString对象
function object:onStart( commands )
	local count = commands:count()

	if count > 0 then 
		local command0 = commands:objectAtIndex(0)
		local length = tolua.cast(command0, "CCString"):length()
		print("the first string length is " .. length)

	end 
end
{% endcodeblock %}

