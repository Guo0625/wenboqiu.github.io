---
layout: post
title: "Lua绑定对C++函数不同类型形参的处理"
date: 2014-09-12 11:31:18 +0800
comments: true
categories: 
---

在Lua中调用C++函数时，产生过一个疑问，C++函数的形参有可能是指针，也可能是引用，或者是按值传递的，在Lua里面，需要针对这些参数进行特殊处理吗？

今天看了下tolua++绑定后的代码，发现绑定代码已经进行转化过了，在Lua中只要传入Lua value就好。

举例来说，bool CCScale9Sprite::initWithFile(const char* file, CCRect rect) 这个方法形参既有指针，也有按值传递，通过tolua++绑定后，lua实际调用的函数会是怎么样的呢？
<!--more-->
{% codeblock lang:c %}
/* method: initWithFile of class  CCScale9Sprite */
#ifndef TOLUA_DISABLE_tolua_Cocos2d_CCScale9Sprite_initWithFile01
static int tolua_Cocos2d_CCScale9Sprite_initWithFile01(lua_State* tolua_S)
{
 tolua_Error tolua_err;
 if (
     !tolua_isusertype(tolua_S,1,"CCScale9Sprite",0,&tolua_err) ||
     !tolua_isstring(tolua_S,2,0,&tolua_err) ||
     (tolua_isvaluenil(tolua_S,3,&tolua_err) || !tolua_isusertype(tolua_S,3,"CCRect",0,&tolua_err)) ||
     !tolua_isnoobj(tolua_S,4,&tolua_err)
 )
  goto tolua_lerror;
 else
 {
  CCScale9Sprite* self = (CCScale9Sprite*)  tolua_tousertype(tolua_S,1,0);
  const char* file = ((const char*)  tolua_tostring(tolua_S,2,0));
  CCRect rect = *((CCRect*)  tolua_tousertype(tolua_S,3,0));
#ifndef TOLUA_RELEASE
  if (!self) tolua_error(tolua_S,"invalid 'self' in function 'initWithFile'", NULL);
#endif
  {
   bool tolua_ret = (bool)  self->initWithFile(file,rect);
   tolua_pushboolean(tolua_S,(bool)tolua_ret);
  }
 }
 return 1;
tolua_lerror:
 return tolua_Cocos2d_CCScale9Sprite_initWithFile00(tolua_S);
}
#endif //#ifndef TOLUA_DISABLE
{% endcodeblock %}
从上面代码，可以看到，tolua_tostring 和 tolua_tousertype 返回的其实都是指针，第一个参数需要的是指针，因此直接将file传入就好，第二个参数需要的是值，因此第17行代码对指针做了 解引用 处理。因此在Lua代码中，只需要传入Lua value就好，不需要关心实际C++函数需要的是指针还是值，或者引用。