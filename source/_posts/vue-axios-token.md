---
title: VUE+AXIOS实现登录拦截Demo
date: 2017-04-09 22:00
tags: 
    - js
    - vue
---

> 一个项目学会vue全家桶+axios实现登录、拦截、登出功能，以及利用axios的http拦截器拦截请求和响应。

## 前言
通过这个项目学习如何实现一个前端项目中所需要的
登录及拦截、登出、token失效的拦截及对应 axios 拦截器的使用。
<!--more-->
**准备**
你需要先熟悉axios文档（https://github.com/mzabriskie/axios）。
* 发起http请求的方法
* http 请求成功时返回的数据及其类型
* http请求失败的处理
* 拦截器的使用
* http的配置


## src下项目结构
**此项目demo是由vue-cli脚手架生成**
```
  |__ api  // 配置api接口文件
    |__ account.js  
  |__ App.vue
  |__ assets  // 资源文件
    |__ css
      |__ base.scss
      |__ min.scss
      |__ personal.scss
    |__ js  // 公共的function
      |__ pubFunc.js
  |__ components
    |__ Spinner.vue
  |__ main.js
  |__ page
    |__ center
      |__ index.vue
    |__ market
      |__ index.vue
    |__ personal  // 登陆组件
      |__ login.vue
      |__ register.vue
      |__ resetPassword.vue
    |__ routerview
      |__ index.vue
    |__ tool
      |__ index.vue
    |__ viewing
      |__ index.vue
  |__ router  
    |__ index.js  // 配置路由
    |__ router.js // 路由定义
  |__ store  // 状态管理
    |__ global
      |__ actions.js
      |__ getters.js
      |__ index.js
      |__ mutations.js
      |__ mutations_types.js
      |__ state.js
    |__ index.js
  |__ utils
    |__ httpconfig.js  // 二次封装axios请求及http 拦截器配置文件
    |__ localstorage.js  //利用class封装本地存储方法
```

### 技术栈
* Vue 2.0
* vue-router
* vuex
* axios
* sass

### 登录拦截逻辑

#### 第一步：路由拦截
本项目是每次进来要强制先登陆的，所以路由定义不用添加自定义字段用于判断该路由是否要先登陆

定义完路由后，我们主要是利用`vue-router`提供的钩子函数`beforeEach()`对路由进行判断。
```javascript
router.beforeEach((to, from, next) => {
  if (!window.localStorage.getItem('token') && to.path !== '/' && to.path !== '/register' && to.path !== '/resetpassword') { // 判断没有本地存储token,如果不是跳转登陆注册修改密码，则跳转到登陆
    next('/');
  } else {
    if (window.localStorage.getItem('token') && to.path === '/') { //判断如果有本地存储token，用户打开应用则跳转到首页
      next({
        path: '/routerview'
      });
    } else {
      next();
    }
  }
})
```
每个钩子方法接收三个参数：
* to: Route: 即将要进入的目标 路由对象
* from: Route: 当前导航正要离开的路由
* next: Function: 一定要调用该方法来 resolve 这个钩子。执行效果依赖 next 方法的调用参数。
  * next(): 进行管道中的下一个钩子。如果全部钩子执行完了，则导航的状态就是 confirmed （确认的）。
  * next(false): 中断当前的导航。如果浏览器的 URL 改变了（可能是用户手动或者浏览器后退按钮），那么 URL 地址会重置到 from 路由对应的地址。
  * next('/') 或者 next({ path: '/' }): 跳转到一个不同的地址。当前的导航被中断，然后进行一个新的导航。

**确保要调用 next 方法，否则钩子就不会被 resolved。**
> 完整的方法见`/src/router.js`



登录拦截到这里就结束了吗？并没有。这种方式只是简单的前端路由控制，并不能真正阻止用户访问需要登录权限的路由。还有一种情况便是：当前token失效了，但是token依然保存在本地。这时候你去访问需要登录权限的路由时，实际上应该让用户重新登录。
这时候就需要结合 http 拦截器 + 后端接口返回的http 状态码来判断。

#### 第二步：拦截器
要想统一处理所有http请求和响应，就得用上 axios 的拦截器。通过配置`http response inteceptor`，当后端接口返回`401 Unauthorized（未授权）`，让用户重新登录。
```javascript
// 添加请求拦截器
axios.interceptors.request.use(function (config) {
  store.dispatch('show_loading');
  if(store.state.global.token){
    config.headers.Authorization = 'Bearer '+ `${store.state.global.token}`;
  };
  return config;
}, function (error) {
  store.dispatch('hide_loading');
  return Promise.reject(error);
});


// 添加响应拦截器
axios.interceptors.response.use(function (response) {
  store.dispatch('hide_loading');
  return response;
}, function (error) {

  if (error.response) {
    switch (error.response.status) {
      case 401:
        // 返回 401 清除token信息并跳转到登录页面
        store.dispatch('remove_token');
        router.replace({
          path: '/',
          query: {redirect: router.currentRoute.fullPath}
        })
    }
  }
  store.dispatch('hide_loading');
  return Promise.reject(error)   // 返回接口返回的错误信息
});
```
>完整的方法见`/src/utils/httpconfig.js`.

通过上面这两步，就可以在前端实现登录拦截了。`登出`功能也就很简单，只需要把当前token清除，再跳转到首页即可。

```

另一种情况是应用一开始不是强制用户先登陆，而是进入应用之后，跳转路由才去判断用户需不需要用户先登陆才能进行下一步操作
[这个业务情景我也帮你们找到demo了](https://github.com/superman66/vue-axios-github?utm_source=tuicool&utm_medium=referral)
