## 一、session和cookie

http请求无状态，不能每次发送请求都要将账号密码发送给服务器做判断，所以在用户成功登陆后，服务器会存储用户信息生成sessionID，并将这个sessionID发送给用户的浏览器，这一系列的过程形成会话（session），此时浏览器中会使用cookie来存储sessionID，浏览器会存储这个cookie，在下次请求的时候浏览器将cookie发送给服务器，服务器通过cookie中存储的sessionID，还原用户的操作情形。

cookie是不安全的 在浏览器的调试模式下 通过 ducument.cookie 便可查看。

session的安全性高，但是对服务器的存储压力大。每次认证后，服务器都会记录并存储在内存中，随着用户认证量的增长，服务器的开销会变大。

## 二、jwt

由于session认证的弊端，所以使用token认证机制

**token认证流程**

- 客户端使用账号密码请求服务端
- 服务端通过认证并生成token
- 客户端存储这个token，每次请求的时候将这个token存放在请求头中发送给服务端
- 服务端再验证这个token，完成请求，返回数据

就是比较厉害的token规范

由header，playload，signature三个部分用 `.`连接而成的字符串

#### header

jwt的头部承载两部分信息：

- 声明类型，这里是jwt
- 声明加密的算法 通常直接使用 HMAC SHA256

完整的头部就像下面这样的JSON：

```json
{
  'typ': 'JWT',
  'alg': 'HS256'
}
```

然后将头部进行base64加密（该加密是可以对称解密的),构成了第一部分.

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

#### playload

载荷就是存放有效信息的地方。这个名字像是特指飞机上承载的货品，这些有效信息包含三个部分

- 标准中注册的声明
- 公共的声明
- 私有的声明

**标准中注册的声明** (建议但不强制使用) ：

- **iss**: jwt签发者
- **sub**: jwt所面向的用户
- **aud**: 接收jwt的一方
- **exp**: jwt的过期时间，这个过期时间必须要大于签发时间
- **nbf**: 定义在什么时间之前，该jwt都是不可用的.
- **iat**: jwt的签发时间
- **jti**: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。

**公共的声明** ： 公共的声明可以添加任何的信息，一般添加用户的相关信息或其他业务需要的必要信息.但不建议添加敏感信息，因为该部分在客户端可解密.

**私有的声明** ： 私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64是对称解密的，意味着该部分信息可以归类为明文信息。

定义一个payload:

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

然后将其进行base64加密，得到Jwt的第二部分。

```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
```

#### signature

jwt的第三部分是一个签证信息，这个签证信息由三部分组成：

- header (base64后的)
- payload (base64后的)
- secret

这个部分需要base64加密后的header和base64加密后的payload使用`.`连接组成的字符串，然后通过header中声明的加密方式进行加盐`secret`组合加密，然后就构成了jwt的第三部分。

```js
// javascript
var encodedString = base64UrlEncode(header) + '.' + base64UrlEncode(payload);

var signature = HMACSHA256(encodedString, 'secret'); // TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

将这三部分用`.`连接成一个完整的字符串,构成了最终的jwt:

```
  eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

**注意：secret是保存在服务器端的，jwt的签发生成也是在服务器端的，secret就是用来进行jwt的签发和jwt的验证，所以，它就是你服务端的私钥，在任何场景都不应该流露出去。一旦客户端得知这个secret, 那就意味着客户端是可以自我签发jwt了。**

#### 如何应用

##### 前端

一般是在请求头里加入`Authorization`，并加上`Bearer`标注：

```json
fetch('api/user/1', {
  headers: {
    'Authorization': 'Bearer ' + token
  }
})
```

服务端会验证token，如果验证通过就会返回相应的资源

##### 在Java后端中的应用

1. 引入maven依赖

```xml
<!--引入JWT依赖,由于是基于Java，所以需要的是java-jwt-->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.1</version>
        </dependency>
        <dependency>
            <groupId>com.auth0</groupId>
            <artifactId>java-jwt</artifactId>
            <version>3.4.0</version>
        </dependency>
```



1. 利用aop创建两个注解，用于拦截器区分是否进行拦截和验证

   ```java
   //登陆注解 不会拦截 会执行登陆生成token
   @Target({ElementType.METHOD, ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   public @interface LoginToken {
       boolean required() default true;
   }
   ```

   ```java
   //会进行拦截 验证token
   @Target({ElementType.METHOD, ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   public @interface CheckToken {
       boolean required() default true;
   }
   ```

2. 编写JwtUtil类，用于生成token，解析token，验证token

   ```java
   public class JwtUtil {
       /**
        * 用户登录成功后生成Jwt
        * 使用Hs256算法  私匙使用用户密码
        *
        * @param ttlMillis jwt过期时间
        * @param user      登录成功的user对象
        * @return
        */
       public static String createJWT(long ttlMillis, User user) {
           //指定签名的时候使用的签名算法，也就是header那部分，jjwt已经将这部分内容封装好了。
           SignatureAlgorithm signatureAlgorithm = SignatureAlgorithm.HS256;
   
           //生成JWT的时间
           long nowMillis = System.currentTimeMillis();
           Date now = new Date(nowMillis);
   
           //创建payload的私有声明（根据特定的业务需要添加，如果要拿这个做验证，一般是需要和jwt的接收方提前沟通好验证方式的）
           Map<String, Object> claims = new HashMap<String, Object>();
           claims.put("id", user.getId());
           claims.put("username", user.getUsername());
           claims.put("password", user.getPassword());
   
           //生成签名的时候使用的秘钥secret,这个方法本地封装了的，一般可以从本地配置文件中读取，切记这个秘钥不能外露哦。它就是你服务端的私钥，在任何场景都不应该流露出去。一旦客户端得知这个secret, 那就意味着客户端是可以自我签发jwt了。
           String key = user.getPassword();
   
           //生成签发人
           String subject = user.getUsername();
   
   
   
           //下面就是在为payload添加各种标准声明和私有声明了
           //这里其实就是new一个JwtBuilder，设置jwt的body
           JwtBuilder builder = Jwts.builder()
                   //如果有私有声明，一定要先设置这个自己创建的私有的声明，这个是给builder的claim赋值，一旦写在标准的声明赋值之后，就是覆盖了那些标准的声明的
                   .setClaims(claims)
                   //设置jti(JWT ID)：是JWT的唯一标识，根据业务需要，这个可以设置为一个不重复的值，主要用来作为一次性token,从而回避重放攻击。
                   .setId(UUID.randomUUID().toString())
                   //iat: jwt的签发时间
                   .setIssuedAt(now)
                   //代表这个JWT的主体，即它的所有人，这个是一个json格式的字符串，可以存放什么userid，roldid之类的，作为什么用户的唯一标志。
                   .setSubject(subject)
                   //设置签名使用的签名算法和签名使用的秘钥
                   .signWith(signatureAlgorithm, key);
           if (ttlMillis >= 0) {
               long expMillis = nowMillis + ttlMillis;
               Date exp = new Date(expMillis);
               //设置过期时间
               builder.setExpiration(exp);
           }
           return builder.compact();
       }
   
   
       /**
        * Token的解密
        * @param token 加密后的token
        * @param user  用户的对象
        * @return
        */
       public static Claims parseJWT(String token, User user) {
           //签名秘钥，和生成的签名的秘钥一模一样
           String key = user.getPassword();
   
           //得到DefaultJwtParser
           Claims claims = Jwts.parser()
                   //设置签名的秘钥
                   .setSigningKey(key)
                   //设置需要解析的jwt
                   .parseClaimsJws(token).getBody();
           return claims;
       }
   
   
       /**
        * 校验token
        * 在这里可以使用官方的校验，我这里校验的是token中携带的密码于数据库一致的话就校验通过
        * @param token
        * @param user
        * @return
        */
       public static Boolean isVerify(String token, User user) {
           //签名秘钥，和生成的签名的秘钥一模一样
           String key = user.getPassword();
   
           //得到DefaultJwtParser
           Claims claims = Jwts.parser()
                   //设置签名的秘钥
                   .setSigningKey(key)
                   //设置需要解析的jwt
                   .parseClaimsJws(token).getBody();
   
           if (claims.get("password").equals(user.getPassword())) {
               return true;
           }
   
           return false;
       }
   
   }
   ```

3. 编写拦截器对请求进行拦截(拦截所以请求，并判断注解)

   ```java
   public class AuthenticationInterceptor implements HandlerInterceptor {
   
       @Autowired
       UserService userService;
   
       //在controller调用之前执行
       @Override
       public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object object) throws Exception {
   
           // 从 http 请求头中取出 token
           String token = httpServletRequest.getHeader("token");
           // 如果不是映射到方法直接通过
           if (!(object instanceof HandlerMethod)) {
               return true;
           }
   
           HandlerMethod handlerMethod = (HandlerMethod) object;
           Method method = handlerMethod.getMethod();
           //检查是否有LoginToken注释，有则跳过认证
           if (method.isAnnotationPresent(LoginToken.class)) {
               LoginToken loginToken = method.getAnnotation(LoginToken.class);
               if (loginToken.required()) {
                   return true;
               }
           }
   
           //检查有没有需要用户权限的注解
           if (method.isAnnotationPresent(CheckToken.class)) {
               CheckToken checkToken = method.getAnnotation(CheckToken.class);
               if (checkToken.required()) {
                   // 执行认证
                   if (token == null) {
                       throw new RuntimeException("无token，请重新登录");
                   }
                   // 获取 token 中的 user id
                   String userId;
                   try {
                       userId = JWT.decode(token).getClaim("id").asString();
                   } catch (JWTDecodeException j) {
                       throw new RuntimeException("访问异常！");
                   }
                   User user = userService.findUserById(userId);
                   if (user == null) {
                       throw new RuntimeException("用户不存在，请重新登录");
                   }
                   Boolean verify = JwtUtil.isVerify(token, user);
                   if (!verify) {
                       throw new RuntimeException("非法访问！");
                   }
                   return true;
               }
           }
           return true;
       }
   
       //controller调用之后，dispatcherServlet视图渲染之前执行，post，get都可以处理
       @Override
       public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
   
       }
   
       //dispatcherServlet视图渲染之后执行
       @Override
       public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
   
       }
   
   
   }
   ```

4. controller演示

   ```java
     @RestController
     @RequestMapping("/api")
     public class UserController {
     
      @Autowired
      private UserService userService;
     
      //登录
      @PostMapping("/login")
      @LoginToken
      public Object login(@RequestBody @Valid com.pjb.springbootjjwt.jimisun.User user) {
     
          JSONObject jsonObject = new JSONObject();
          User userForBase = userService.findByUsername(user);
          if (userForBase == null) {
              jsonObject.put("message", "登录失败,用户不存在");
              return jsonObject;
          } else {
              if (!userForBase.getPassword().equals(user.getPassword())) {
                  jsonObject.put("message", "登录失败,密码错误");
                  return jsonObject;
              } else {
                  String token = JwtUtil.createJWT(6000000, userForBase);
                  jsonObject.put("token", token);
                  jsonObject.put("user", userForBase);
                  return jsonObject;
              }
          }
      }
     
      //查看个人信息
      @CheckToken
      @GetMapping("/getMessage")
      public String getMessage() {
          return "你已通过验证";
      }
     }
   ```
