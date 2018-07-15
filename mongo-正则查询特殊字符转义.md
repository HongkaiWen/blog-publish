---
title: mongo 正则查询特殊字符转义
date: 2016-04-28 12:46:32
tags:
---

**mongo 模糊查询通过正则表达式来实现。**

eg:
*模糊查询
Pattern pattern = Pattern.compile(String.format("^.*%s.*$", groupName), Pattern.CASE_INSENSITIVE);
query.append("name", pattern);
同样正则表达式查询有较大的灵活性，不仅仅可以实现模糊匹配
不区分大小写查询
Pattern pattern = Pattern.compile(String.format("^%s$", groupName), Pattern.CASE_INSENSITIVE);*

**但是通过正则来做查询功能，前端输入的参数不可以直接作为参数放到上面两个例子的输入参数里，因为前端输入的参数有可能含有正则的元字符： ^ | ( $ 等。**

所以此处需要转义元字符，使其不具备特殊含义。
像下面这种查询条件即可使输入的元字符时区特殊含义：
{ "fieldName" : { "$regex" : "^.*\Q$\E.*$" , "$options" : "i"}}

java 代码实现：

    String displayName = Pattern.quote(input);
    Pattern pattern = Pattern.compile("^.*"+input+".*$", Pattern.CASE_INSENSITIVE);
    query.append("fieldName", pattern);

