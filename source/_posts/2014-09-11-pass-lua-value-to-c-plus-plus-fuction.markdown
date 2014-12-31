---
layout: post
title: "在Lua中如何调用带字符参数的C++函数"
date: 2014-09-11 16:40:20 +0800
comments: true
categories: 
---

之前一直有一个疑问，在Lua中调用一个C++函数，如果函数的形参是const char* 或者 std::string 类型，能否直接传入一个Lua的string值呢？

今天仔细看了下通过tolua++绑定后的代码，发现绑定代码会将lua的string值转成C++函数指定的字符类型，不管是const char* 还是std::string。

比如说CCDictionary中的CCObject* objectForKey(const std::string& key)，这个成员函数需要const std::string的引用。那么经过tolua++绑定后，lua中调用这个函数实际又会执行什么样的代码呢。
<!--more-->
下面就是绑定后的代码：
{% codeblock lang:c %}
/* method: objectForKey of class  CCDictionary */
#ifndef TOLUA_DISABLE_tolua_Cocos2d_CCDictionary_objectForKey00
static int tolua_Cocos2d_CCDictionary_objectForKey00(lua_State* tolua_S)
{
#ifndef TOLUA_RELEASE
 tolua_Error tolua_err;
 if (
     !tolua_isusertype(tolua_S,1,"CCDictionary",0,&tolua_err) ||
     !tolua_iscppstring(tolua_S,2,0,&tolua_err) ||
     !tolua_isnoobj(tolua_S,3,&tolua_err)
 )
  goto tolua_lerror;
 else
#endif
 {
  CCDictionary* self = (CCDictionary*)  tolua_tousertype(tolua_S,1,0);
  const std::string key = ((const std::string)  tolua_tocppstring(tolua_S,2,0));
#ifndef TOLUA_RELEASE
  if (!self) tolua_error(tolua_S,"invalid 'self' in function 'objectForKey'", NULL);
#endif
  {
   CCObject* tolua_ret = (CCObject*)  self->objectForKey(key);
    int nID = (tolua_ret) ? (int)tolua_ret->m_uID : -1;
    int* pLuaID = (tolua_ret) ? &tolua_ret->m_nLuaID : NULL;
    toluafix_pushusertype_ccobject(tolua_S, nID, pLuaID, (void*)tolua_ret,"CCObject");
   tolua_pushcppstring(tolua_S,(const char*)key);
  }
 }
 return 2;
#ifndef TOLUA_RELEASE
 tolua_lerror:
 tolua_error(tolua_S,"#ferror in function 'objectForKey'.",&tolua_err);
 return 0;
#endif
}
#endif //#ifndef TOLUA_DISABLE
{% endcodeblock %}
从上面代码中，我们可以看到,第17行tolua_tocppstring这个方法会把lua stack中的lua value 转成std::string。

再比如CCLabelTTF中的virtual void setString(const char label)，这个成员函数需要一个const char的值，那我们看看tolua++绑定后的实现。
{% codeblock lang:c %}
/* method: setString of class  CCLabelTTF */
#ifndef TOLUA_DISABLE_tolua_Cocos2d_CCLabelTTF_setString00
static int tolua_Cocos2d_CCLabelTTF_setString00(lua_State* tolua_S)
{
#ifndef TOLUA_RELEASE
 tolua_Error tolua_err;
 if (
     !tolua_isusertype(tolua_S,1,"CCLabelTTF",0,&tolua_err) ||
     !tolua_isstring(tolua_S,2,0,&tolua_err) ||
     !tolua_isnoobj(tolua_S,3,&tolua_err)
 )
  goto tolua_lerror;
 else
#endif
 {
  CCLabelTTF* self = (CCLabelTTF*)  tolua_tousertype(tolua_S,1,0);
  const char* label = ((const char*)  tolua_tostring(tolua_S,2,0));
#ifndef TOLUA_RELEASE
  if (!self) tolua_error(tolua_S,"invalid 'self' in function 'setString'", NULL);
#endif
  {
   self->setString(label);
  }
 }
 return 0;
#ifndef TOLUA_RELEASE
 tolua_lerror:
 tolua_error(tolua_S,"#ferror in function 'setString'.",&tolua_err);
 return 0;
#endif
}
#endif //#ifndef TOLUA_DISABLE
{% endcodeblock %}
从上面代码中，我们可以看到，第17行tolua_tostring这个方法会把lua stack中的lua value 转成const char*。

因此可以得出结论：Lua中调用C++函数，不管参数需要的std::string 还是 const char*类型，只要传入Lua string值即可。

在当前基础上，再稍微拓展一下，因为Lua在运行期会对string、number值进行自动转换，因此即使传入一个lua number值，也会被转换成Lua string值。所以，上述的结论可以改为：Lua中调用C++函数，不管参数需要的std::string 还是 const char*类型，只要传入Lua string值或number值即可。

另外如果C++函数需要的是CCString*，我们还是老老实实的在Lua中创建一个CCString对象，然后把对象的指针传入这个C++函数，因为CCString对象在Lua中被认为是userdata类型，跟一般的C++对象是一致的。

