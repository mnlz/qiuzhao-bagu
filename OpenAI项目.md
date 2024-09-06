# OpenAI项目

这个项目的核心功能包括：**微信公众号认证，通过关注微信公众号获得验证码**、**对接各类大模型的OpenAI-sdk：例如chatGPT、文心引言、智谱AI、同时自定义注解和AOP的方式设计限流组件、**对模型生成的内容进行各种**规则校验和敏感词校验**、最后通过sse模型实现

模型的**流式应答、**项目中使用策略模式、工厂模式、模板模式大量的简化代码代码拥有较好的扩展性

## 1.单Token

### 1.1 JWT简介

**JWT token的令牌格式：**

- Header采用的加密算法和类型

  - 使用HS256加密算法

- Payload ==**有效载荷-任何人可以看到-不能放敏感信息**==

  - **本次项目中使用 openId作为有效载荷**

- 签发人，也就是JWT是给谁的（逻辑上一般都是username或者userId）

- Signature-签名

  - 保证上面信息不会被篡改

  - **HMACSHA256( base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)**
  - **对称加密算法，自定义一个加密的密钥，别人就无法伪造token信息**

### **1.2 验证过程**

**Java服务端可以安全地验证JWT的合法性，根据Payload中的字段信息进行用户身份认证**

**拦截JWT**：通过Spring的`HandlerInterceptor`接口，编写自定义拦截器，从请求头中提取JWT。

**验证JWT**：拦截器解析并验证JWT的签名和过期时间，确保令牌合法。

**注册拦截器**：通过Spring的`WebMvcConfigurer`配置拦截器，确保其能拦截指定的请求。

**使用JWT信息**：在通过拦截器的请求中，可以通过`HttpServletRequest`获取用户的身份信息。

<img src="./OpenAI项目.assets/image-20240906111144146.png" alt="image-20240906111144146" style="zoom: 50%;" />

### 1.3 cookie、session和token

http协议是无状态的，服务端是不知道请求是不是同一个客户端，三种方案

cookie：用户登录成功之后，服务端生成一个cookie返回客户端，客户端下次请求携带cookie，Cookie 存储在客户端，可能会被窃取或篡改，因此对敏感信息的存储需要进行加密处理；

session：用户登录成功之后，服务端根据用户的数据，生成一个sessionId，session本质上是通过sessionId来关联用户的敏感信息，因为用户的敏感信息不能存储在浏览器端。session是存放在服务端，客户端每次请求通过cookie携带相应的seesionID

Token机制：Token 机制不需要在服务器上保存任何关于用户的状态信息，只需要在登录成功时，服务器端通过某种算法生成一个唯一的 Token 值，之后再将此 Token 发送给客户端存储（存储在 **localStorage 或 sessionStorage** 中）并且服务端是不存储这个token的，服务端只进行校验。

Cookie 是应用程序通过在客户端存储临时数据，用于实现状态管理的一种机制；Session 是服务器端记录用户状态的方式，服务器会为每个会话分配一个唯一的 Session ID，并将其与用户状态相关联；Token 是一种用于认证和授权的一种机制，通常表示用户的身份信息和权限信息。

## 2.双Token机制

长短token三验证

两个token：长短token的意义是过期时间的长短，不是字符串的长度

- access_token:用来验证，正常的业务请求 30分钟
- refresh_token：用来续约 3天

具体的流程：

1.正常用户登录之后，生成双token，将refresh_token的md5存入redis

2.前端访问业务接口的时候携带access_token进行访问，==业务系统验证token的有效性（时间+解析用户）==

3.验证成功，执行业务代码

4.验证失败，如果是token过期，使用refresh_token访问续约的接口

5.==验证refresh_token的有效性==&&再次和redis中存储的md5值进行比对

6.生成一对新的access_token 和 refres_token，把新的refresh_token的md5值放到redis中

7.前端携新的access_token进行访问

**`重点在于refresh_token的一次性`**



<img src="./OpenAI项目.assets/image-20240906163127617.png" alt="image-20240906163127617" style="zoom:50%;" />

## 3.AOP和限流算法

SpringAOP的底层原理：

AOP面向切面编程，实现方法的解耦，在不改变源码的基础上，扩充方法。

常见的有两种方式实现AOP，
