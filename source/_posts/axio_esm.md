---
title: axios的ES module (esm)
date: 2020/06/05
categories: 
- 前端
---
##### 背景
* 刚好遇到某个场景需要用到`es module`, axios用习惯了, 不过axios官方没有esm版本 [https://github.com/axios/axios/issues/1879](https://github.com/axios/axios/issues/1879)


##### 解决
* 可以用第三方: [https://github.com/bundled-es-modules/axios](https://github.com/bundled-es-modules/axios)
* 使用起来很简单, 直接用里面的`axios.js`即可
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>ESM-test</title>
</head>
<body>
<script type="module">
    import axios from './axios.js'
    
    axios.get('http://127.0.0.1:8888/shuogg').then(resp => {
        console.log(resp.data)
    }).catch(e => {
        console.log(e)
    })
</script>
</body>
</html>

```