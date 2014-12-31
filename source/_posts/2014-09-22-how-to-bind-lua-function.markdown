---
layout: post
title: "如何绑定LUA_FUNCTION"
date: 2014-09-22 16:15:34 +0800
comments: true
categories: 
---

今天在项目中尝试在C++代码中调用Lua中的function，实现的逻辑是先在Lua代码中调用C++函数注册自己的handler，然后在需要时由C++调用这个handler，触发Lua中的function。

这个流程可以参照CCLayer中的实现。
{% codeblock lang:c %}
//Step 1:
//注册handler函数，需要由tolua++绑定，在lua代码中调用
void CCLayer::registerScriptTouchHandler(int nHandler, bool bIsMultiTouches, int nPriority, bool bSwallowsTouches);

//Step 2:
//由LuaEngine执行该handler对应的Lua Function
int CCLayer::excuteScriptTouchHandler(int nEventType, CCTouch *pTouch)
{
	return CCScriptEngineManager::sharedManager()->getScriptEngine()->executeLayerTouchEvent(this, nEventType, pTouch);
}

//executeLayerTouchEvent方法中执行handler的代码
m_stack->executeFunctionByHandler(nHandler, 3);

{% endcodeblock %}
<!--more-->
在这个流程中，handler是至关重要的，LuaEngine需要知道这个handler，才可以正确执行对应的Lua Function。那么我们现在的问题就是如何才能获得Lua Function的句柄呢？

如果我们还是按照以前的经验，在pkg文件中，将registerScriptTouchHandler函数的第一个参数声明为int类型，那么由tolua++生成的绑定代码仍然会认为第一个参数是int类型，如果在Lua代码中调用这个函数，第一个参数是Lua Function，就会报错。

那该如何呢？

我们需要在pkg文件中将该函数的第一个参数标记为LUA_FUNCTION类型（而不是int类型），这个LUA_FUNCTION类型在CCLuaValue.h中声明。
{% codeblock lang:c %}
typedef int LUA_FUNCTION;
{% endcodeblock %}
pkg文件中该函数的声明如下：
{% codeblock lang:c %}
void registerScriptTouchHandler(LUA_FUNCTION nHandler, bool bIsMultiTouches = false, int nPriority = 0, bool bSwallowsTouches = false);
{% endcodeblock %}
当pkg文件该参数标记为LUA_FUNCTION类型后，tolua++生成的绑定代码中会把第一个参数认为是Lua Function，会获取该function的句柄，传给registerScriptTouchHandler函数。
{% codeblock lang:c %}
LUA_FUNCTION nHandler = toluafix_ref_function(tolua_S,2,0);
{% endcodeblock %}
下面是registerScriptTouchHandler函数完整的绑定代码：
{% codeblock lang:c++ %}
/* method: registerScriptTouchHandler of class  CCLayer */
#ifndef TOLUA_DISABLE_tolua_Cocos2d_CCLayer_registerScriptTouchHandler00
static int tolua_Cocos2d_CCLayer_registerScriptTouchHandler00(lua_State* tolua_S)
{
#ifndef TOLUA_RELEASE
 tolua_Error tolua_err;
 if (
     !tolua_isusertype(tolua_S,1,"CCLayer",0,&tolua_err) ||
     (tolua_isvaluenil(tolua_S,2,&tolua_err) || !toluafix_isfunction(tolua_S,2,"LUA_FUNCTION",0,&tolua_err)) ||
     !tolua_isboolean(tolua_S,3,1,&tolua_err) ||
     !tolua_isnumber(tolua_S,4,1,&tolua_err) ||
     !tolua_isboolean(tolua_S,5,1,&tolua_err) ||
     !tolua_isnoobj(tolua_S,6,&tolua_err)
 )
  goto tolua_lerror;
 else
#endif
 {
  CCLayer* self = (CCLayer*)  tolua_tousertype(tolua_S,1,0);
  LUA_FUNCTION nHandler = (  toluafix_ref_function(tolua_S,2,0));
  bool bIsMultiTouches = ((bool)  tolua_toboolean(tolua_S,3,false));
  int nPriority = ((int)  tolua_tonumber(tolua_S,4,0));
  bool bSwallowsTouches = ((bool)  tolua_toboolean(tolua_S,5,false));
#ifndef TOLUA_RELEASE
  if (!self) tolua_error(tolua_S,"invalid 'self' in function 'registerScriptTouchHandler'", NULL);
#endif
  {
   self->registerScriptTouchHandler(nHandler,bIsMultiTouches,nPriority,bSwallowsTouches);
  }
 }
 return 0;
#ifndef TOLUA_RELEASE
 tolua_lerror:
 tolua_error(tolua_S,"#ferror in function 'registerScriptTouchHandler'.",&tolua_err);
 return 0;
#endif
}
#endif //#ifndef TOLUA_DISABLE
{% endcodeblock %}

Lua代码中可以以下面形式调用registerScriptTouchHandler这个函数。
{% codeblock lang:lua %}
function handler(target, method)
	return function(…) return method(target, …) end 
end

object: registerScriptTouchHandler(handler(object, object.functionOne))
{% endcodeblock %}