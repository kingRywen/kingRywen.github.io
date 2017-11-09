---
title: JS常用原生函数汇总
date: 2017-11-09 09:56:26
tags:
    - 原生函数
    - 实用
---

### 获取url参数

```javascript
function getQueryString(name) {
  var reg = new RegExp("(^|&)" + name + "=([^&]*)(&|$)");
  var r = window.location.search.substr(1).match(reg);
  if (r != null) return unescape(r[2]); return null;
}
```
