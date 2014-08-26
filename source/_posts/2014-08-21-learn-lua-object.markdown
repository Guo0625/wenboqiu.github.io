---
layout: post
title: "Lua 面向对象实现(Part I)"
date: 2014-08-21 19:51:24 +0800
comments: true
categories: 
---

接触Lua已经有一年的，但是一直对Lua的面向对象实现一知半解，全赖wp同学的工作，我们可以不用管具体的细节，把项目顺利完成。不过有东西一直一知半解，总觉得难受，于是趁这几天有空，把这块知识点好好梳理了一番。

一、Lua Table 模拟面向对象实现

Lua本身并不是面向对象的语言，没有面向对象语言的类、继承、多态等特性，但是凭借其自身强大的table特性，类、继承、多态等特性能够在Lua中被模拟出来。

Table是Lua八种基础类型中的一种，也是最重要的特性之一，其他语言中的数组，字典，在Lua中都可以通过Table来实现。
<!--more-->
在Lua中，每一种类型自身都有一个metatable，中文翻译成元表，它的作用在于提供了一种改变对象默认行为的途径，Lua中每种类型都有默认的操作方式，比如说数字的加减乘除，字符串的连接等，当两个数字相加，比如a+b，Lua会先查看a的metatable，看里面__add有没有定义，如果没有定义，则调用默认的+规则，否则会调用_add来执行+的操作。又比如两个table值默认是不支持+操作的，但是通过定义metable中的_add，我们可以实现表的相加。

{% codeblock lang:lua %}
--定义2个表
a = {5, 6}
b = {7, 8}
--用c来做Metatable
c = {}
--重定义加法操作
c.__add = function(op1, op2)
	for _, item in ipairs(op2) do
		table.insert(op1, item)
	end
	return op1
end
--将a的Metatable设置为c
setmetatable(a, c)
--d现在的样子是{5,6,7,8}
d = a + b
{% endcodeblock %}

Metatable不光提供__add域，还提供了其他操作域，如下面所列的这些。

add, sub, mul, div, mod, pow, unm, concat, len, eq, lt, le, tostring, gc, index, newindex, call…

在上述这些操作域中，index是Lua能够成功模拟面向对象特性的关键，index提供了表的索引值入口，当表要索引一个值时如table[key], Lua会首先在table本身中查找key的值, 如果没有，且这个table被定义过Metatable，且Metatable中index域也被定义过，则Lua会按照index所定义的来查找，__index可以是函数也可以是一个table。这种行为不正与面向对象的继承很类似吗，调用一个对象的方法，如果这个方法在对象所属的类中没有定义，就自动去父类查找。

通过这种方式，也可以模拟实现类的概念，Lua中每个table都是一个对象，但是通过metatable，Lua可以把一个对象实现类的功能，下面是Programming In Lua中的例子。

{% codeblock lang:lua %}
local Bird = {CanFly = true}
Bird.__index = Bird

function Bird:New()
	local b = {}
	setmetatable(b, self)
	return b
end

local aBird = Bird:New()
if aBird.CanFly then 
	print(“a bird can fly”) 
end
{% endcodeblock %}

例子中的Lua对象Bird就被模拟成一个类。通过Bird:New()，每次生成一个新的lua对象（空的table），这个新的对象拥有Bird的行为。

那么如何实现继承呢，请看下面的例子。

{% codeblock lang:lua %}
local Crow = {Color = “black”}
Crow.__index = Crow
setmetatable(Crow, Bird)

function Crow:New()
	local c = {}
	setmetatable(c, self)
	return c
end
{% endcodeblock %}

通过设置setmetatable，将Crow的metatable设置为Bird，实现了继承。

二、Lua 对象如何模拟继承C++对象

在具体的Cocos2dx项目中，不同的框架对Lua对象化的实现方式也有不同。在实际项目中，Lua对象不光可能模拟“继承”自另外一个Lua对象，亦可能需要模拟“继承”自一个C++对象，这个该如何实现呢。

首先还得说说Metatable，因为实现模拟“继承”自C++对象，还得依赖Metatable强大的特性。在Lua Reference Manual中有提到，用户可以通过getmetatable这个方法获得任何一种Lua类型的值的Metatable，但是用户要想替换Lua值的Metatable，却是有限制的，只有table类型的值可以通过setmetatable来设置metatable，而其他类型的值要想设置metatable，通过Lua的API恐怕就无法办到（除了用debug库），必须使用C的API才能办到。

在Cocos2dx项目中，自定义的C++类是通过tolua来生成绑定代码，实现Lua访问C++类的属性和方法。tolua自动将C/C++ basic类型map到Lua的 basic类型中去。char, int, float, and double are mapped to the Lua type number; char* is mapped to string; void* is mapped to userdata.至于C++中的用户自定义类型，比如说struct，自定义类，在Lua中，会被map成userdata类型。

那么如何让Lua对象模拟继承自一个C++类呢，即模拟继承自一个userdata类型呢？在上文中有提到userdata类型的metatable是无法通过Lua API来实现的，那就没办法像模拟继承一个Lua object一样来实现。

幸好tolua提供了一个机制，允许我们store any additional field attached to these objects，因为这个机制，这些objects也可以被当做成传统的Lua table。

{% codeblock lang:lua %}
obj = ClassName:new()
obj.myfield = 1 --即使ClassName这个类中没有定义myfield，左边这样赋值也是成立的。
{% endcodeblock %}

上述赋值之所以能够成立，是因为当需要时，tolua自动创建一个Lua table关联到这个对象上。因此，obj这个对象之所以能够存储C++类中没有定义过的字段，是因为这些新的字段被存到了一个伴生的table里。Lua程序员可以使用统一的方式访问C++中定义的字段和这些在Lua中新定义的字段，如果两个字段名一样（C++字段和Lua新定义字段），Lua新定义字段（方法）会覆盖掉C++中的字段（方法）(注：这个覆盖只在Lua层有效)。

tolua提供了一个方法让程序员可以设置这个伴生的table，即the object's peer table（仅支持Lua 5.1）。这个方法是 tolua.setpeer (object, peer_table) 。tolua++将这个peer table设置为object's envirnment table，使用lua_gettable/settable来获得或者设置这个peer table中的字段值。这个机制允许我们对自己的Lua table实现我们自己的对象系统，实现对userdata对象的继承。下面请看tolua手册中的例子。

{% codeblock lang:lua %}
-- a 'LuaWidget' class
LuaWidget = {}
LuaWidget.__index = LuaWidget

function LuaWidget:add_button(caption)
-- add a button to our widget here. 'self' will be the userdata Widget
end

local w = Widget()
local t = {}
setmetatable(t, LuaWidget) -- make 't' an instance of LuaWidget

tolua.setpeer(w, t) --make 't' the peer table of 'w'

set_parent(w) --we use 'w' as the object now

w:show() --a method from 'Widget'
w:add_button(“Quit”) --a method from LuaWidget (but we still use 'w' to call it)
{% endcodeblock %}

上面Widget是C++类，LuaWidget是Lua类，通过tolua.setpeer，将LuaWidget的一个实例设置为w（Widget的一个实例）的peer table, 如此一来，LuaWidget可以被认为是Widget的子类，因为每次访问w的方法时，首先会去查看它的peer table有没有这个方法，如果有，就执行peer table中的方法，如果没有，才执行Widget中定义的方法。这样，就实现了Lua对象继承C++对象的“模拟”。
