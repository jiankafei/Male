# 小程序框架的简单封装

  ！不允许修改 frame 文件夹里的文件

## 解决的问题

  1. app onLaunch 等周期函数以及登录请求和 page 的周期函数的执行是异步的，导致两者不能衔接的问题
  2. page 周期函数中 返回页面和进入页面以及挂起后重回页面 onshow 周期函数无法区分的问题
  3. 第三方埋点操作侵入性过强的问题

## App.ready - Promise

  1. 一个 Promise 实例，成功表示登录流程和初始信息加载流程完成；
  2. 一般不需要显式调用，新增的onCreated和onForward生命周期函数在内部使用该 Promise 实例

## Frame

  用于初始化注入，并返回 Main 函数

  ```js
  import Frame from './frame/index';
  const Main = Frame({
    env,
    init: res => {},
    loginToSite: data => {},
  });
  Main({});
  ```

  1. 注入 env, 详见 env
  2. 注入 init 方法，非必填：
    App 登录之后需要执行的初始操作；
    一般可以在这里做初始信息加载等操作；
    每次启动app，该方法都会执行，如果没有登录，该方法会在登录后执行，并接收登录接口的结果作为参数；
    返回一个 Promise 实例；
  3. 注入登录到服务器 loginToSite 方法，非必填：
    返回一个 promise 实例；该方法会接收到一个参数：

  ```js
  data = {
    code,
    encrypted_data, // 授权后才会有
    iv, // 授权后才会有
  }
  ```

## Main

  对 App 函数的封装

### 添加的功能函数

  1. methodCaptured:
    page实例或component实例中任何函数的执行都会触发该函数的执行，并传递参数到该函数，不包含生命周期函数；
    该函数可以实现代码无侵入埋点等操作

  ```js
  options = {
    is, // page 或者 component 的 is 属性
    name, // 函数名
    event, // 事件对象
    result, // 函数执行的返回值
    instance, // page 或者 component 实例
  }
  ```

## App.Page - Method

  对 Page 函数的封装

  ```js
  App.Page({});
  ```

### 删除的生命周期函数

  1. onLoad
  2. onShow

### 添加的生命周期函数

  1. onCreated:
    在内部调用 App.ready，替代 onLoad，App.ready流程完毕后触发，onCreated(query, res) {}
  2. onAppear:
    替代 onShow，如果想有和onCreated一样的触发时机，则可以使用 App.ready 实例
  3. onForward:
    在内部调用 App.ready，页面进入并显示时触发，App.ready流程完毕后触发，onForward(res) {}
  4. onBackward:
    返回到当前页面并显示时触发
  5. onReappear:
    切换到后台又恢复显示时触发

  注：其他配置和原框架一致

## App.Comp - Method

  对 Component 函数的封装，组件需使用 lifetimes 字段来管理生命周期，写在外部的生命周期函数将不起作用，根据微信小程序设计，Component 函数也可以用于实例化页面，用于替代 Page 函数

  ```js
  App.Comp({});
  ```

## Network

  以下方法都进行了 promisify ，除了 App.FC ，其他方法都在 promise 实例上挂载了 task 对象

### App.FC - Object

  小程序请求接口的封装，用法类似 axios，提供拦截器和默认设置等操作

  ```js
  const FC = App.FC;
  FC.defaults = {};
  // 拦截器支持链式调用
  // middleware: async function | common function | common function with promise
  // reqWall callback 参数为 options 请求配置，并且需要返回 options
  // resWall callback 参数为 res 请求结果，并且需要返回 res
  // callback 返回为 promise 时，promise 的 resolve 必须相应的传递 options 或 res
  // callback 绑定了请求实例
  FC.reqWall
    .add(middleware)
    .remove(middleware)
  FC.resWall
    .add(middleware)
    .remove(middleware)
  FC.fetch(options);
  const ins = FC.create(defaults);
  ins.defaults = {}; // 会覆盖 create 方法里的 defaults
  ins.reqWall
    .add(middleware)
    .remove(middleware)
  ins.resWall
    .add(middleware)
    .remove(middleware)
  ins.fetch(options);

  // 选项
  options = {
    baseURL: '',
    url: '',
    data: Object.create(null),
    header: Object.create(null),
    method: 'GET',
    dataType: 'json',
    responseType: 'text',
    validateStatus: status => status >= 200 && status < 300 || status === 304,
  };
  ```

### App.DL - Method

  小程序下载接口的封装

### App.UL - Method

  小程序上传接口的封装

### App.WS - Method

  小程序双工通讯接口的封装

## env

  ```js
  STORE_KEY: '', // 存储app信息的 localStorege key
  BASE_TOKEN: '', // 基础 token
  LOGIN_URL: '',  // 登录到站点的url
  LOGIN_TYPE: '', // 登录类型 smart | auth | silent 默认 smart
  INDEX_ROUTE: '', // 主页路径，没有前置/，默认 pages/index/index
  CHECK_SESSION_TYPE: '', // 检测session的方式 api | store 默认 api
  NAV_BAR_MODE: '', // 自定义组件nav-bar模式 dark | light  默认 dark
  HAS_USERINFO: , // 是否已经获取过用户信息，用于判断是否需要获取用户信息
  ```

## store

  app.store 相当于 app.globalData

  默认有：

  ```js
  systemInfo, // wx.getSystemInfoSync 的结果
  navBarInfo, // 导航栏相关布局信息
  userInfo, // 默认为空对象
  ```

## Login

  在 config 文件里配置登录类型，除 login 方法外的其他三个方法已挂载到 app 实例上

  1. authLogin: 授权登录，只有授权后才会调用登录流程，并获取用户微信信息
  2. silentLogin: 静默登录，不需要用户授权，执行静默登录
  3. smartLogin: 兼容登录，授权情况下，调用 authLogin，没有授权，调用 silentLogin
  4. login: 该方法在内部调用，做了是否登录的判断。通过 config 来配置登录方式 (smart | auth | silent)，并默认调用 smart

### 登录注意

  1. 自定义登陆态下，当前页面请求时，session_key 过期，则会重新登录并重新加载页面
  2. 在做授权登录时，如果需要自行调用上述三个登录函数，则需要按照下面的方式操作：

  ```js
    App.ready = app.authLogin();
    // or
    App.ready = app.silentLogin();
    // or
    App.ready = app.smartLogin();
    App.ready
      .then()
      .catch();
  ```

## Methods

  各种工具函数，已挂载到 app 实例上

  ```js
  // 操作 querystring
  qs
    parse
    stringify
  // 操作 localStorage
  local
    get
    set
    remove
    clear
  // 操作 localStorage 存储里的某一个 key
  localKey
    get
    set
    remove
  // 操作 localStorage 存储里的 access_token
  session
    get
    set
    remove

  storeCheckSession, // 基于是否存储 token 判断是否过期
  apiCheckSession, // 基于 wx.checkSession 接口判断 token 是否过期
  checkSession, // 上两个方法的封装

  updateApp, // 更新app
  getSystemInfo, // 获取系统信息
  getPage, // 获取当前page
  rpx2px, // rpx 转换到 px

  getUserProfile, // promisify 'wx.getUserProfile' api
  checkAuth, // promisify 权限鉴定
  loginToWx, // promisify 登陆到 微信服务器
  loginToSite, // promisify 登陆到自家服务器
  ```

## 内部组件

  以下两个组件均已添加到全局组件

  nav-bar: 自定义导航组件

  注：请自行设置不使用原生导航

    @props title // 非必填，标题
    @props color // 非必填，标题颜色
    @props background // 非必填，导航栏背景
    @props fill // 非必填，是否占据空间
    @props back // 非必填，是否显示 back 按钮
    @props home // 非必填，是否显示 home 按钮
    @props mode // 非必填，按钮样式，dark | light

  user-info: 授权 getUserProfile 组件

    @props visibility // 非必填，外部控制是否显示
    @event userinfo // 点击授权按钮事件
    @event success // 授权成功事件
    @event fail // 授权失败事件

## 注意

1. 登录流程和初始信息加载流程在 App 启动时，只会执行一次，因此除了初始显式的页面，其他页面的 onCreated 等有类似启动时机的周期函数将会很快被执行，因此无需担心新的周期函数会增加页面打开时间；
2. 对于全局数据，建议只在app启动时调用一次获取接口，并把数据放到全局数据中，然后其他接口更新相关数据后，也一并更新全局数据即可，好处是不需要每个页面都要获取数据，加快页面显示；

## ChangeLog

最新的微信小程序支持增强编译，因此去掉了 runtime 对 async function, Promise.prototype.finally 的原有支持。
