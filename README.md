# vue-proxy

> 在Vue.js中对跨域请求的基本操作

当我们在路由里面配置成以下代理可以解决跨域问题

```javascript
// config/index.js
module.exports = {
    // ...
    dev: {
        prixyTable: {
            // proxy all requests starting with /api to jsonplaceholder
            '/api/*': {
                target: 'http://jsonplaceholder.typicode.com',
                changeOrigin: true,
                pathRewrite: {
                '^/api' :'/'
                }
            }
        }
    }
}
```

这种配置方式在一定程度上避免了 ``vue-router`` 中会有相同的命名，产生路由冲突,所以我们把所有接口统一规范为一个入口。
其中 ``pathRewrite`` 用于将虚拟的这个api接口去掉，此时真正去后端请求的时候，不会加上api这个前缀，但是，这样我们前台http请求的时候，还必须加上api前缀才能匹配到这个代理。示例代码如下：

```javascript
logout(){
    axios.post( '/api/users/logout')
    .then(result=>{
        let res = result.data;
        this.nickName = '';
        console.log(res);
    })
}
```
所以为了解决这个问题，我们可以利用axios的baseUrl直接默认值是api，这样每次访问的时候，就会自动补上api前缀。

```javascript
// main.js
import axios from 'axios'
axios.defaults.baseUrl = 'api'
```

不过像上面这样配置的话，不会区分生产和开发环境。所以我们可以在config 文件夹里面新建一个 api.config.js 配置文件：

```javascript
const isPro = Object.is(process.env.NODE_ENV, 'production')

module.exports = {
    baseUrl : osPro ? 'http://www.vnshop.cn/api/' : 'api/'
}
```

然后在main.js 里面引入,这样可以保证动态的匹配生产和开发的定义前缀：

```javascript
//main.js
import Vue from 'vue'
import App from './App'
import router from './router'
import axios from "axios"

import apiConfig from '../config/api.config'
axios.defaults.baseURL = apiConfig.baseURL
```

经过上面的配置后，在dom里面可以轻松的访问，也不需要在任何组建里面引入axios模块了。示例代码如下：
```javascript
logout(){
    axios.post( '/users/logout')
    .then(result=>{
        let res = result.data;
        this.nickName = '';
        console.log(res);
    })
}
```

## 跨域请求QQ音乐歌单数据
为了后端代理绕过其referer和host,顺利请求到数据，所以我们需要做一些配置，并且需要安装依赖express，代码如下：
```javascript
//build/webpack.dev.conf.js
const axios = require('axios')
const express = require('express')
const app = express()
const apiRouters = express.Router()
app.use('/api', apiRouters)

const devWebpackConfig = merge(baseWebpackConfig, {
    // ...
    devServer: {
        // ...
        before(app){
        //获取推荐歌单
            app.get('/api/getDiscList', function(req, res) {
                const url = 'https://c.y.qq.com/splcloud/fcgi-bin/fcg_get_diss_by_tag.fcg';
                axios.get(url, {
                    headers: {
                    referer: 'https://c.y.qq.com/',
                    host: 'c.y.qq.com'
                    },
                    params: req.query
                }).then((response) => {
                    res.json(response.data);
                }).catch((e) => {
                    console.log(e);
                });
            });
            //获取歌手列表
            app.get('/api/getSingers', function(req, res) {
                const url = 'https://c.y.qq.com/v8/fcg-bin/v8.fcg';
                axios.get(url, {
                    headers: {
                    referer: 'https://c.y.qq.com/',
                    host: 'c.y.qq.com'
                    },
                    params: req.query
                }).then((response) => {
                    res.json(response.data);
                }).catch((e) => {
                    console.log(e);
                });
            })
        }
    }
}
```
既然是要绕过请求头，那么其余的一些信息也不能放过，我们可以将获取数据封装成一个方法，然后再调用它，代码如下：
```javascript
// src/api/singer.js
import jsonp from '../common/js/jsonp'
import { commonParams, options } from './config'
import axios from 'axios'

export function getSingerList() {
    const url = '/api/getSingers'
    const data = Object.assign({}, commonParams, {
        channel: 'singer',
        page: 'list',
        key: 'all_all_all',
        pagenum: 1,
        pagesize: 100,
        hostUin: 0,
        g_tk: 759984046,
        needNewCode: 0,
        platform: 'yqq'
    })
    return axios.get(url, { params: data })
        .then((res) => {
            return Promise.resolve(res.data)
        })
}
```
这样就能轻松绕过并且顺利拿到数据。
PS.详情请见 JiEr-music 项目。