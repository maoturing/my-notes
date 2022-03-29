# 0. 学习计划

- 分布式会话
- 拦截器
- 单点登录





# 1. Redis 分布式会话



## 1.1 无状态会话

HTTP请求是无状态的，用户向服务端发起多个请求，服务端并不会知道这多次请求都是来自同一用户，这个就是无状态的。cookie的出现就是为了有状态的记录用户。

常见的，ios与服务端交互，安卓与服务端交互，前后端分离，小程序与服务端交互，他们都是通过发起http来调用接口数据的，每次交互服务端都不会拿到客户端的状态，但是我们可以通过手段去处理，比如每次用户发起请求的时候携带一个userid 或者 user-token，如此一来，就能让服务端根据用户id或token来获得相应的数据。每一次请求都能被服务端识别来自同一个用户。



## 1.2 有状态会话

Tomcat中的会话，就是有状态的，一旦用户和服务端交互，就有会话，会话保存了用户的信息，这样用户就“有状态”了，服务端会和每个客户端都保持着这样的一层关系，这个由容器来管理（也就是tomcat），这个session会话是保存到内存空间里的，如此一来，当不同的用户访问服务端，那么就能通过会话知道谁是谁了。tomcat会话的出现也是为了让http请求变的有状态。如果用户不再和服务端交互，那么会话超时则消失，结束了他的生命周期。如此一来，每个用户其实都会有一个会话被维护，这就是有状态会话。

场景：在传统项目或者jsp项目中是使用的最多的session都是有状态的，session的存在就是为了弥补http的无状态。

注：tomcat会话可以通过手段实现多系统之间的状态同步，但是会损耗一定的时间，一旦发生同步那么用户请求就会等待，这种做法不可取



## 1.3 会话 Session

单个 Tomcat 会话就是有状态的，用户首次访问服务端，服务端生成会话 Session，并将 jsessionid 放入前端 cookie 中，后续每次请求都会携带 jsessionid 以保持用户状态。

![image-20220120173651906](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220120173651906.png)

会话Session代表的是客户端与服务器的一次交互过程，这个过程可以是连续也可以是时断时续的。曾经的Servlet时代（jsp），一旦用户与服务端交互，服务器 tomcat 就会为用户创建一个session，同时前端 Cookie 会保存一个 jsessionid，每次交互都会携带。

如此一来，服务器只要在接到用户请求时候，就可以拿到 jsessionid，并根据这个ID在内存中找到对应的会话 session，当拿到 session 会话后，那么我们就可以操作会话了。

会话存活期间，我们就能认为用户一直处于正在使用着网站的状态，一旦session超期过时，就可以认为用户已经离开网站，停止交互了。用户的身份信息，我们也是通过session来判断的，在session中可以保存不同用户的信息。

session的使用之前在单体部分演示过，代码如下：

```java
@GetMapping("/setSession")
public String setSession(HttpServletRequest request) {
    // 如果前端没有传递 jsessionid, 那么会创建一个新的session会话
    // 如果前端传递了 jsessionid, 那么会根据id查找到这个session会话并返回
    HttpSession session = request.getSession();
    // session返回到前端后, 会自动保存jsessionid到cookie中, 以后每次访问, 都会携带jsessionid来表名身份


    // 打印session-id, 第一次访问会创建新的session, 后面的访问则使用同一个session-id
    log.info("SESSION-ID:"+session.getId()+", 是否为新创建的session:" + (session.isNew()));

    // 这些信息可以传递到前端, 用来显示用户名等信息
    session.setAttribute("userInfo", "new user");
    session.setMaxInactiveInterval(3600);
    session.getAttribute("userInfo");

    return "ok";
}
```



## 1.4 动静分离会话

用户请求服务端，由于动静分离，前端发起 http 请求，不会携带任何状态，当用户第一次请求时，我们手动设置一个 token，作为用户会话，保存到 redis 中，即作为 redis-session，并且将这个 token 保存到前端 token 中。

在后续交互过程中，前端传递 token 给后端，后端就能识别这个请求来自哪个用户了。

![image-20220120174118595](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220120174118595.png)



## 1.5  集群分布式系统会话

集群或分布式系统本质都是多个系统，将用户状态保存到第三方 redis 中，在 A 系统中登录后，请求 B 系统，也可以通过 redis 识别用户会话。



![image-20220120174142436](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220120174142436.png)





## 1.6 使用 Redis 实现分布式会话

1. 为用户生成唯一 token，并将`userId` 作为 key，`token`作为 value 保存到 Redis中

```java
// 成用户token, 存入redis
private UsersVO generateUserToken(Users userResult) {
    String uniqueToken = UUID.randomUUID().toString().trim();
    // 这里可以设置过期时间, 即过一段时间用户登录失效需要重新登录
    redisOperator.set(CommonConstant.REDIS_USER_TOKEN + ":" + userResult.getId(), uniqueToken);

    UsersVO usersVO = new UsersVO();
    // 去除敏感信息
    BeanUtils.copyProperties(userResult, usersVO);
    usersVO.setUserUniqueToken(uniqueToken);
    return usersVO;
}
```



2. 登录时将 token 保存到前端 cookie 中，还有用户id，用户名，用户昵称，用户头像等信息

```java
@PostMapping("/login")
public IMOOCJSONResult login(@RequestBody UserBO userBO,
                             HttpServletRequest request,
                             HttpServletResponse response) throws Exception {
    //......
    // 去数据库查询用户信息，检查密码是否正确
    Users userResult = userService.queryUserForLogin(username, MD5Utils.getMD5Str(password));
    if (userResult == null) {
        return IMOOCJSONResult.errorMsg("用户名或密码错误");
    }
    
    // 1.生成用户token, 存入redis会话, 并去除敏感信息
    UsersVO usersVO = generateUserToken(userResult);

    // 2.将用户信息包括token保存到cookie返回到前台
    CookieUtils.setCookie(request, response, "user",
                          JsonUtils.objectToJson(usersVO), true);

    // ......
    return IMOOCJSONResult.ok(userResult);
}
```



3. 退出登录时删除 token

```java
@PostMapping("/logout")
public IMOOCJSONResult logout(@RequestParam("userId") String userId,
                              HttpServletRequest request,
                              HttpServletResponse response) {
    // 1.清除用户的相关信息的cookie
    CookieUtils.deleteCookie(request, response, "user");
    // 2.清楚redis中保存的用户token
    redisOperator.del(CommonConstant.REDIS_USER_TOKEN + ":" + userId);

    // 用户退出登录, 需要清空购物车
    CookieUtils.deleteCookie(request, response, CommonConstant.SHOPCART_COOKIE_NAME);

    return IMOOCJSONResult.ok();
}
```





## 1.7 拦截器

上一节，为每个会话创建了 token，这一节我们使用 token 来拦截非法请求。

1. 创建拦截器

   写这种代码前, 先不要着急写逻辑, 先把流程测试通, 等测试后确认请求可以被当前拦截器拦截, 再写业务逻辑

```java
public class UserTokenInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        // 写这种代码前, 先不要着急写逻辑, 先把流程测试通, 等测试后确认请求可以被当前拦截器拦截, 再写业务逻辑

        logger.info("请求进入到拦截器, 请求不通过");
        return false;
    }
```

2. 注册拦截器，配置拦截器拦截的请求`/hello`

```java
@Configuration      // 让Spring容器扫描到
public class WebMvcConfig implements WebMvcConfigurer {

    @Bean
    public UserTokenInterceptor userTokenInterceptor() {
        return new UserTokenInterceptor();
    }
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 请求http://localhost:8088/hello
        // 测试是否会拦截 HelloController 的 /hello, debug UserTokenInterceptor#preHandler
        registry.addInterceptor(userTokenInterceptor())
                .addPathPatterns("/hello")
            
        // WebMvcConfigurer.super.addInterceptors(registry); 
    }
```

3. 访问`http://localhost:8088/hello`，测试拦截器能否拦截该请求。至此，拦截器流程跑通，配置拦截器需要拦截的请求路径。包括订单，用户信息，购物车，都是需要登录后才能访问的

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    // 添加拦截的url
    registry.addInterceptor(userTokenInterceptor())
        .addPathPatterns("/hello")
        .addPathPatterns("/shopcart/*")
        .addPathPatterns("/address/*")
        .addPathPatterns("/orders/*")
        .addPathPatterns("/center/*")
        .addPathPatterns("/userInfo/*")
        .addPathPatterns("/myorders/*")
        .addPathPatterns("/mycomments/*")
        .excludePathPatterns("/myorders/deliver")
        .excludePathPatterns("/orders/notifyMerchantOrderPaid");

    //  WebMvcConfigurer.super.addInterceptors(registry);     // 经过测试, 这行代码没必要
}
```

4. 编写拦截器的业务逻辑，检查拦截到的请求，检查是否携带了`headerUserId`和`headerUserToken`参数，根据`userId`去 redis 中查找当前用户的 token，若与请求中的 headerUserToken 一致，则说明用户已经登录了，放行返回 true。

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    // 写这种代码前, 先不要着急写逻辑, 先把流程测试通, 等测试后确认请求可以被当前拦截器拦截, 再写业务逻辑

    log.info("请求进入到拦截器, 开始校验请求");

    // userinfo.html 中请求后端时, 会将userId和userToken添加到请求头header中
    String userId = request.getHeader("headerUserId");
    String userToken = request.getHeader("headerUserToken");

    if (StringUtils.isBlank(userId) || StringUtils.isBlank(userToken)) {
        log.info("用户未登录, 找不到指定token, 请求被拦截...");
        // 设置返回信息, 将"请登录"作为响应返回.
        returnErrorResponse(response, IMOOCJSONResult.errorMsg("请登录"));
        return false;
    }

    String uniqueToken = redisOperator.get(CommonConstant.REDIS_USER_TOKEN + ":" + userId);
    if (StringUtils.isBlank(uniqueToken)) {
        log.info("redlis中用户token为空, 用户未登录, 请登录");
        returnErrorResponse(response, IMOOCJSONResult.errorMsg("请登录"));
        return false;
    }

    if (userToken.equals(uniqueToken)) {
        log.info("请求携带的用户token与redis中保存的用户token一致, 校验通过");
        return true;
    } else {
        log.info("请求携带的用户token与redis中保存的用户token不一致, 账号在异地登录, 请重新登录");
        returnErrorResponse(response, IMOOCJSONResult.errorMsg("请登录"));
        return false;
    }
}
```

5. 这是个人信息页面前端判断用户是否登录的代码，1.6.2 登录时会将用户信息 userInfo 保存到 cookie 中，若不为空则说明用户已经登录了，并解析用户信息保存到变量`this.userInfo`，其中包括用户id，token，username 等信息。

```javascript
judgeUserLoginStatus() {
    // 从cookie中取出登录时保存的用户信息(包括token)
    var userCookie = app.getCookie("user");
    if (
        userCookie != null &&
        userCookie != undefined &&
        userCookie != ""
    ) {
        var userInfoStr = decodeURIComponent(userCookie);
        // console.log(userInfoStr);
        if (
            userInfoStr != null &&
            userInfoStr != undefined &&
            userInfoStr != ""
        ) {
            var userInfo = JSON.parse(userInfoStr);
            // 判断是否是一个对象
            if ( typeof(userInfo)  == "object" ) {
                this.userIsLogin = true;
                // console.log(userInfo);
                this.userInfo = userInfo;
            } else {
                this.userIsLogin = false;
                this.userInfo = {};
            }
        }
    } else {
        this.userIsLogin = false;
        this.userInfo = {};
    }
}

```

6. 这是获取用户信息的前端代码，获取用户信息需要登录，这里我们携带上一步解析出的用户id `userInfo.id`和`userInfo.userUniqueToken`来访问后端，这样 1.7.4 就能拦截到该请求，用 `userInfo.id`查询 redis 得到 token，再与请求中的 token 对比，一致则说明用户一登录，放行。

```js
renderUserInfo() {
    var userInfo = this.userInfo;
    // console.log(userInfo);
    // 请求后端获得最新数据
    var serverUrl = app.serverUrl;
    axios.defaults.withCredentials = true;
    axios.get(
        // 请求个人信息, 需要验证是否登录
        serverUrl + '/userInfo/get?userId=' + userInfo.id,
        {
            headers: {
                // 请求用户个人信息时携带用户id和token
                'headerUserId': userInfo.id,
                'headerUserToken': userInfo.userUniqueToken
            }
        })
        .then(res => {
        if (res.data.status == 200) {
            var userInfoMore = res.data.data;
            // console.log(userInfoMore);
            this.userInfoMore = userInfoMore;

            var datepicker = moment(userInfoMore.birthday).format('YYYY-MM-DD');
            $("#datepicker").attr("value", datepicker);
        } else {
            // 若返回值不是200, 则弹出返回信息, 这里是"请登录"
            alert(res.data.msg);
            console.log(res.data.msg);
        }
    });
},
```



## 3. 单点登录 SSO

在面试过程中有时候会被问到单点登录，单点登录又称之为Single Sign On，简称SSO，单点登录可以通过基于用户会话的共享，分为两种，先来看第一种，那就是他的原理是分布式会话来实现。

比如说现在有个一级域名为 www.imooc.com，是教育类网站，但是慕课网有其他的产品线，可以通过构建二级域名提供服务给用户访问，比如：music.imooc.com，shop.imooc.com， blog.imoo c.com等等，分别为慕课音乐，慕课电商以及慕课博客等，用户只需要在其中一个站点登录，那么其他站点也会随之而登录。

也就是说，用户自始至终只在某一个网站下登录后，那么他所产生的会话，就共享给了其他的网站，实现了单点网站登录后，同时间接登录了其他的网站，那么这个其实就是单点登录，他们的会话是共享的，都是同一个用户会话。



1. Cookie＋Redis 实现 SSO

之前我们所实现的分布式会话后端是基于redis的，如此会话可以流窜在后端的任意系统，都能获取到缓存中的用户数据信息，前端通过使用cookie，可以保证在同域名的一级二级下获取，那么这样一来，cookie中的信息userid和token是可以在发送请求的时候携带上的，这样从前端请求后端后是可以获取拿到的，这样一来，其实用户在某一端登录注册以后，其实cookie和redis中都会带有用户信息，只要用户不退出，那么就能在任意一个站点实现登录了。

- 那么这个原理主要也是cookie和网站的依赖关系，顶级域名`www.imooc.com`和 `*.imooc.com`的cookie值是可以共享的 ，可以被携带至后端的，比如设置为`.imooc.com`，`.t.mukewang.com`，如此是OK的。

- 二级域名自己的独立cookie是不能共享的，不能被其他二级域名获取，比如：`music.imooc.com`的cookie是不能被`mtv.imooc.com`共享，两者互不影响，要共享必须设置为`.imooc.com`。

2. Cookie共享测试

找到前端项目app.js，开启如下代码，设置你的对应域名，需要和SwitchHosts相互对应：

```js
cookieDomain: ".t.mukewang.com;"
```

```nginx
127.0.0.1	shop.t.mukewang.com
127.0.0.1	center.t.mukewang.com
```



如下图，可以看到，不论是在shop或是center中，两个站点都能够在用户登录后共享用户信息。



如此一来，cookie中的信息被携带至后端，而后端又实现了分布式会话，那么如此一来，单点登录就实现了，用户无需再跨站点登录了。上述过程我们通过下图来更加具象化的展示，只要前端网页都在同一个顶级域名下，就能实现cookie与session的共享：









![image-20220124230118665](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220124230118665.png)



![image-20220124230137611](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220124230137611.png)



> **小技巧：** 

# 待补充




# 进阶学习



# 推荐阅读

[前端鉴权的兄弟们：cookie、session、token、jwt、单点登录 - 掘金 (juejin.cn)](https://juejin.cn/post/6898630134530752520)

[Cookie 的 SameSite 属性 - 阮一峰的网络日志 (ruanyifeng.com)](https://www.ruanyifeng.com/blog/2019/09/cookie-samesite.html)

[Springboot应用中设置Cookie的SameSite属性 - 博客园 (cnblogs.com)](https://www.cnblogs.com/kevinblandy/p/13589864.html)

# 参考文档




# 学习记录

2022年1月14日17:28:17
