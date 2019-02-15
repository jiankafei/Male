# 小程序框架的简单封装

  ！不允许修改 frame 文件夹里的文件

## App.ready - Promise

  1. 一个 Promise 实例，成功表示登录流程和初始信息加载流程完成；
  2. 一般不需要显式调用，新增的onCreated和onForward生命周期函数在内容使用该 Promise 实例

## Frame

  import Frame from './frame/index';
  用于初始化注入，并返回 Main 函数

  1. 注入 config，详见 config
  2. 注入 store，详见 store
  3. 注入 init 方法，非必填：
    App 登录之后需要执行的初始操作；
    一般可以在这里做初始信息加载等操作；
    每次启动app，该方法都会执行，如果没有登录，该方法会在登录后执行，并接收登录接口的结果作为参数；
    返回一个 Promise 实例；
  4. 注入登录到服务器 loginToSite 方法，非必填：
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

  1. catchMethods
    page实例或component实例中任何函数的执行都会触发该函数的执行，并传递参数到该函数，不包含生命周期函数；
    该函数可以实现代码无侵入埋点等操作

    ```js
    options = {
      route, // page 里为 this.route comp 里为 this.is
      name, // 函数名
      event, // 事件对象
      result, // 函数执行的返回值
      scope, // page 或者 component 实例
    }
    ```

## App.Page - Method

  对 Page 函数的封装

### 删除的生命周期函数

  1. onLoad
  2. onShow

### 添加的生命周期函数：

  1. onCreated:
    替代 onLoad，小程序登陆流程和初始信息加载流程完毕后触发
  2. onAppear:
    替代 onShow，如果想有和onCreated一样的触发时机，则可以使用 App.ready 实例
  3. onForward:
    页面进入并显示时触发，但触发时机和 onCreated 一样
  4. onBackward:
    返回到当前页面并显示时触发
  5. onReappear:
    切换到后台又恢复显示时触发
  注：其他配置和原框架一致

## App.Comp - Method

  替代 Component 函数，组件需使用 lifetimes 字段来管理生命周期，因为在外部写的生命周期函数不会起作用

## App.Fetch - Method

  小程序请求接口的封装，返回一个请求实例

## config

  配置
  APP_STORE_KEY: '',
  BASE_TOKEN: '',
  LOGIN_URL: '',
  LOGIN_TYPE: '', // both_login | auth_login | silent_login 默认 both_login
  INDEX_ROUTE: '', // 主页路径，没有前置/，默认 pages/index/index
  CHECK_SESSION_TYPE: '', // store | api 默认 store
  NAV_BAR_MODE: '', // dark | light 自定义组件nav-bar 默认 dark

## store

  app.store // 相当于 app.globalData

  默认有：

  1. systemInfo,
  2. navBarInfo,
  3. userInfo, // 默认为空对象
  4. indexRoute, // 来源于 config
  5. navBarMode, // 来源于 config

## async await 支持

  在需要使用 async 函数的文件里顶部添加如下代码：

  ```js
    const regeneratorRuntime = App.regeneratorRuntime;
  ```

## Login

  在 config 文件里配置登录类型，除 login 方法外的其他三个方法已挂载到 app 实例上

  1. authLogin: 授权登录，只有授权后才会调用登录流程，并获取用户微信信息
  2. silentLogin: 静默登录，不需要用户授权，执行静默登录
  3. bothLogin: 兼容登录，授权情况下，调用authLogin，没有授权，调用silentLogin
  4. login: 一般只需调用该方法，传入要调用的方法名 (both_login | auth_login | silent_login) 该方法在内部做了是否登录的判断，并默认调用 both_login，除非需要显式调用登录方法

  注：在做授权登录时，如果需要自行调用上述任何一个登录函数，则需要按照下面的方式操作：

  ```js
    App.ready = app.login(name);
    // or
    App.ready = app.authLogin();
    // or
    App.ready = app.silentLogin();
    // or
    App.ready = app.bothLogin();
    App.ready
      .then()
      .catch();
  ```

## Methods

  各种工具函数，已挂载到 app 实例上

  getStorage,
  setStorage,
  removeStorage,
  clearStorage,

  getKey, // 获取app存储的某一个数据
  setKey, // 设置app存储的某一个数据
  removeKey, // 删除app存储的某一个数据

  getSession, // 获取 token
  setSession, // 设置token
  removeSession, // 删除token

  storeCheckSession, // 基于是否存储 token 判断是否过期
  wxCheckSession, // 基于 wx.checkSession 接口判断token是否过期
  checkSession, // 上两个方法的封装

  updateApp, // 更新app
  getSystemInfo, // 获取系统信息
  getPage, // 获取当前page
  rpx2px, // rpx 转换到 px

  getUserInfo, // promisify 'wx.getUserInfo' api
  checkAuth, // promisify 权限鉴定
  loginToWx, // promisify 登陆到 微信服务器
  loginToSite, // promisify 登陆到自家服务器

## 注意

  1. 登录流程和初始信息加载流程在 App 启动时，只会执行一次，因此除了初始显式的页面，其他页面的 onCreated 等有类似启动时机的周期函数将会很快被执行，因此无需担心新的周期函数会增加页面打开时间；
  2. 对于全局数据，建议只在app启动时调用一次获取接口，并把数据放到全局数据中，然后其他接口更新相关数据后，也一并更新全局数据即可，好处是不需要每个页面都要获取数据，加快页面显示；