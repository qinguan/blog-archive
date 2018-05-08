---
title: Lua pcall及xpcall函数
date: 2018-04-14 14:51:40
tags: 
    - Lua
---

Lua应用在一般情况下很少使用到异常错误处理，但有时为了防止模块调用异常、函数调用异常、文件读写异常等一些非关键路径(有重试/容错手段)直接抛出异常，中断执行，会封装这些函数的调用，进行异常捕获。

Lua的异常捕获主要基于[pcall](https://www.lua.org/pil/8.4.html)及[xpcall函数](https://www.lua.org/pil/8.5.html)。

[pcall函数](https://www.gammon.com.au/scripts/doc.php?lua=pcall)
--

```Lua
Summary
Calls a function in protected mode
Prototype
ok, result [ , result2 ...] = pcall (f, arg1, arg2, ...)

Description
Calls function f with the supplied arguments in protected mode. Catches errors and returns:

On success:
true
function result(s) - may be more than one

On failure:
false
error message
```

举个简单的例子:
```Lua
--- 求和
function sum(a,b,c)
    d = a + b + c
    return d
end

local e = sum(10, 20, 30)
print ("e:", e)
local h = sum("ten", "forty", "nine")
print ("h:", h)

--output:
e:      60
lua: src/pcall_test.lua:12: attempt to perform arithmetic on local 'a' (a string value)
stack traceback:
        src/pcall_test.lua:12: in function 'sum'
        src/pcall_test.lua:26: in main chunk
        [C]: in ?
```
如上述代码所示，当sum函数碰到无法处理的字符串输出时，抛出了一个异常，中止了程序运行。

如果我们期望捕获这种异常，做处理，并继续运行程序，可以如下这样调用：
```Lua
local f, vrf = pcall(sum, "ten", "twenty", "thirty")
if f then
    print(vrf)
else
    print("failed to call sum function:" .. vrf)
end

--output:
failed to call sum function:src/pcall_test.lua:12: attempt to perform arithmetic on local 'a' (a string value)
```

[xpcall函数](https://www.gammon.com.au/scripts/doc.php?lua=xpcall)
--
```Lua
Summary
Calls a function with a custom error handler

Prototype
ok, result = xpcall (f, err)

If an error occurs in f it is caught and the error-handler 'err' is called. Then xpcall returns false, and whatever the error handler returned.

If there is no error in f, then xpcall returns true, followed by the function results from f.
```

举个简单的例子:
```Lua
local function err_handle(x)
    print("err_handle info:" .. x)
end

local f, res = xpcall(function ()
    return sum(10, 20, "a")
end , err_handle)
print(f, res)

--output:
err_handle info:src/pcall_test.lua:12: attempt to perform arithmetic on local 'c' (a string value)
false   nil
```
上面的err_handle就是定义的一个错误处理函数，当然也可以直接改成debug自带的相关函数，如下debug.traceback：
```Lua
local f, res = xpcall(function ()
    return sum(10, 20, "a")
end , debug.traceback)
print(f, res)

-- output:
false   src/pcall_test.lua:12: attempt to perform arithmetic on local 'c' (a string value)
stack traceback:
        src/pcall_test.lua:12: in function <src/pcall_test.lua:11>
        (...tail calls...)
        [C]: in function 'xpcall'
        src/pcall_test.lua:41: in main chunk
        [C]: in ?
```
