---

title: uinapp之request封装

date: 2023-11-05 16:13:35

tags:

---

## 前置

> uniapp内置的request函数



参考官方示例:

``` 
uni.request({
    url: 'https://www.example.com/request', //仅为示例，并非真实接口地址。
    data: {
        text: 'uni.request'
    },
    header: {
        'custom-header': 'hello' //自定义请求头信息
    },
    success: (res) => {
        console.log(res.data);
        this.text = 'request success';
    }
})
```



功能差不多,但是由于uni的这个api的success回调函数,只要有响应(非网络错误),都会走这个函数,无法区分是否正确响应,所以对他进行封装.

## 实现

> 思路:  封装一个http函数,对success之后的逻辑加一层判断.

``` 
utils/request.ts

const http = (options) =>  {
return new Promise((resolve, reject) => {
    uni.request({
      ...options,
      // 响应成功
      success(res) {
        // 状态码 2xx
        if (res.statusCode >= 200 && res.statusCode < 300) {
          // 提取 res.data
          resolve(res.data )
        } else if (res.statusCode === 401) {
          // 401错误   清理用户信息，跳转到登录页
          const memberStore = useMemberStore()
          memberStore.clearProfile()
          uni.navigateTo({ url: '/pages/login/login' })
          reject(res)
        } else {
          // 其他错误
          uni.showToast({
            icon: 'none',
            title: res.data.msg ?? '请求错误',
          })
          reject(res)
        }
      },
      // 响应失败
      fail(err) {
        uni.showToast({
          icon: 'none',
          title: '网络错误，换个网络试试',
        })
        reject(err)
      },
    })
  })
}
```

> 可以再扩展一下,添加拦截器,添加ts类型



``` 
utils/request.ts

const BASE_URL = ''

const httpInterceptor = {
  // 拦截前触发
  invoke(options: UniApp.RequestOptions) {//UniApp 内置的ts类型
    // 非 http 开头需拼接地址
    if (!options.url.startsWith('http')) {
      options.url = BASE_URL + options.url
    }
    // 请求超时
    options.timeout = 10000
    // 添加小程序端请求头标识 小程序必须添加
    options.header = {
      'source-client': 'miniapp',
      ...options.header,
    }
    //  添加 token 请求头标识
    const memberStore = useMemberStore()
    const token = memberStore.profile?.token
    if (token) {
      options.header.Authorization = token
    }
  },
}
uni.addInterceptor('request', httpInterceptor)
uni.addInterceptor('uploadFile', httpInterceptor)
//uni.removeInterceptor('request') 这个用来删除

//封装基本类型
type Data<T> = {
  code: string
  msg: string
  result: T
}
// 支持泛型
export const http = <T>(options: UniApp.RequestOptions) => {
  return new Promise<Data<T>>((resolve, reject) => {
    uni.request({
      ...options,
      success(res) {
        // 状态码 2xx
        if (res.statusCode >= 200 && res.statusCode < 300) {
          //  提取核心数据 res.data
          resolve(res.data as Data<T>)
        } else if (res.statusCode === 401) {
          // 401错误  -> 清理用户信息，跳转到登录页
          const memberStore = useMemberStore()
          memberStore.clearProfile()
          uni.navigateTo({ url: '/pages/login/login' })
          reject(res)
        } else {
          // 其他错误
          uni.showToast({
            icon: 'none',
            title: (res.data as Data<T>).msg || '请求错误',
          })
          reject(res)
        }
      },
      // 响应失败
      fail(err) {
        uni.showToast({
          icon: 'none',
          title: '网络错误，换个网络试试',
        })
        reject(err)
      },
    })
  })
}

```



