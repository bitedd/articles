---
layout: default
title: Overview
parent: Authentication And Authorization
nav_order: 1
---


# 일반적인 보안 용어

 * 접근 주체(Principal) : 보호된 리소스에 접근하는 대상
 * 인증(Authentication) : 보호된 리소스에 접근한 대상에 대해 이 유저가 누구인지, 애플리케이션의 작업을 수행해도 되는 주체인지 확인하는 과정(ex. Form 기반 Login)
 * 인가(Authorize) : 해당 리소스에 대해 접근 가능한 권한을 가지고 있는지 확인하는 과정(After Authentication, 인증 이후)
 * 권한 : 어떠한 리소스에 대한 접근 제한, 모든 리소스는 접근 제어 권한이 걸려있다. 즉, 인가 과정에서 해당 리소스에 대한 제한된 최소한의 권한을 가졌는지 확인한다.


# 인증

 인증은 authentication provider에게 인증을 받는다는 의미로 다음과 같은 의미이다.

 * DB에서 사용자의 유저명과 암호를 확인
 * OAuth 프로토콜을 이용해서 인증서버에서 Access Token / Rrefresh Token을 받는 것

### 세션 관리
인증에 성공하면, 세션관리가 필요하다.


**세션 관리시 크게 나눴을 때, 2개의 다른 접근**
 * Session or Cookies based approach
   + 다른 용어로 http session-based, session/cookie 방식이라고 하기도 한다.
 * JWT (JSON Web Tokens) based approach
   + 이 방식은 대부분 Access Token / Refresh Token 을 이용한다.


### Session Cookie based approach:

1. Server generates a "sessionId" (signs it using "secret key"), and
(a) saves the sessionId in a sessionDB(MySQL이나 DynamoDB같은 별도 저장소에 저장하는 경우도 있다), and
(b) 쿠키를 응답정보의 SetCookie 헤더에 포함시켜서 클라이언트에 보낸다. 이 정보를 가지고, 사용자를 찾아갈 수 있다.

Sample response with multiple Set-Cookie headers will look like this:
```
HTTP/2.0 200 OK
Content-Type: text/html
Set-Cookie: <cookie-name>=<cookie-value>
Set-Cookie: <cookie-name>=<cookie-value>; Expires=<date>
Set-Cookie: <cookie-name>=<cookie-value>; Max-Age=<number>
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain>
Set-Cookie: <cookie-name>=<cookie-value>; Path=<path>
Set-Cookie: <cookie-name>=<cookie-value>; Secure
Set-Cookie: <cookie-name>=<cookie-value>; HttpOnly

[page content]
```

can also use a single Set-Cookie header to set multiple attributes as well
```
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain>; Secure; HttpOnly
```

2. The browser (client side) receives the "cookie" in the response from server, and saves it in the "cookie" storage.

3. The browser then includes the "cookie" within every subsequent request to the server.



### JWT JSON Web Token approach:

1. Server generates an "accessToken", encrypting the "userId" and "expiresIn", with the ACCESS_TOKEN_SECRET,
and sends the "accessToken" to the browser (client side).

2. The browser (client side) receives the "accessToken" and saves it on the client side. 개발자가 javascript등으로 브라우저(local storage, session storage, cookie storage)에 저장하는 처리를 해줘야 한다.

3. The "accessToken" is included in every subsequent request to the server. 개발자가 javascript등으로 토큰을 헤더에 세팅하는 작업을 해줘야 한다.

```
Authorization: Bearer <token>
```


# opaque token vs jot token

Gooogle / GitLab 은 access token으로 opaque token을 사용하고, Keycloak이나 okta는 jot token(jwt) 를 이용한다. A Big Advantage of the jot token is that the verification of the token can be done by the resource server itself.

Opaque토큰을 이용하면 인증서버로 가서 확인을 해야하기 때문에 리소스 사용량은 jot token이 보다 더 적다.

만약에 access token에 대해서 제어(차단; revoke)을 하는 경우에는 , opaque토큰이 맞을 것이다. 왜냐하면, 이러한 접근제한관련 세팅은 인증 서버에 access해야 알 수 있기 때문이다. Jot token은 리소스 서버에서 authorization처리를 하기 때문에, 이러한 측면에서 access token의 expire time을 짧게 가져가는 것이 낫다.

But using jot token, what if your application is so critical that you can't afford this? Even if it is a jot token, you want every request to the bug tracker microservice to be verified  by calling the introspection endpoint of the KeyCloak identity provider. The introspection endpoint will let the bugtracker microservice know whether the token is active or not, whether it is revoked or not.






# Note

* In case of the JWT approach
  + the accessToken itself contains the encrypted “userId”, and the accessToken is not saved within any sessionDB.
  + Since no DB is required in case of the “jwt approach”, it is sometimes called “stateless” approach to managing sessions, since no “state” or “session” is saved within a DB (it is contained within the JWT token itself).
  + The JWT tokens are sometimes referred to as “Bearer Tokens” since all the information about the user i.e. “bearer” is contained within the token.


* In case of the session cookie based approach
  + the sessionId does not contain any userId information, but is a random string generated and signed by the “secret key”.
  + The sessionId is then saved within a sessionDB. The sessionDB is a database table that maps “sessionId” < — -> “userId”.
  + Since sessionIds are stored in a sessionDB, the “session cookie approach” is sometimes called “stateful” approach to managing sessions, since the “state” or “session” is saved within a DB.


* In both approaches the client side must securely save the “cookie” or the “jwt token”. The main difference is that in case of the JWT approach the server does not need to maintain a DB of sessionId for lookup.




# NodeJS

the following libraries can be used to implement the above,

### Cookie ← using express-session (sessionId string generated by server, and corresponding sessionId saved on server) i.e.


```
app.use( session ({ secret: “secret key”}) )
// a sessionId is created by "express-session" and sent in the response as a set-cookie.
// this sessionId can be stored on the server side to a DB eg. MongoDB or Redis or SQL etc.

NOTE: the "secret key" is used to sign the sessionId to verify its integrity when the cookie (sessionId) is returned back to the server on subsequent requests.
```

### JWT ← using jsonwebtoken (userId encrypted within the JWT token) i.e.

```
const accessToken =
jwt.sign( {userId}, ACCESS_TOKEN_SECRET, {expiresIn: “15m”} )
// Note that the "accessToken" is NOT stored on any server side DB, since there is no need to do so.
```


# To maintain sessions

each subsequent request to the server (in case of session-cookie approach) should include the “cookie (sessionId)”, OR (in case of JWT approach) should include the “accessToken”


### Steps in Session Cookie based approach

1. Get the "sessionId" from the request "cookie".

2. Verify the "sessionId" integrity using the "secret key".
Then look up the "sessionId" within the sessionDB on the server and get the "userId".

3. Look up the "userId" within the shoppingCartDB to add items to cart for that userId, or display cart info for that userId.



### Steps in JWT based approach

1. Get the "accessToken" from the request "header".

2. Decrypt the "accessToken" i.e the JWT, using ACCESS_TOKEN_SECRET and get the "userId" ---> (there is no DB lookup).

3. Look up the "userId" within the shoppingCartDB to add items to cart for that userId, or display cart info for that userId.



 * So notice that in case of the session cookie approach, the server would need to maintain a list of all the sessions in a sessions DB. Also notice that this means that there would be a DB lookup to get the userId . DB look ups are relatively slower than JWT decrypting action.
 * In case of JWT approach, since the JWT token itself contains the (encrypted) userId info, we do not need to perform a “look up” to get the userId. The moment the server decrypts the JWT token, it will get the userId.
(NOTE: only the servers that have the shared ACCESS_TOKEN_SECRET can decrypt the JWT token).
 * This allows for multiple servers to be scaled up separately and without the need to have a current copy of the sessionId DB on all servers, or have all servers access a common sessionDB.
 * As long as a server has the same ACCESS_TOKEN_SECRET that was used to generate the JWT token, that server can decrypt the JWT token and get the userId.


# In Node JS
the above can be implemented as follows,


### Cookie ← using express-session

```
app.get("/", (req, res) => {
   console.log(req.session.id)      // get sessionId
   console.log(req.session.cookie)  // get cookie
})
// "express-session" attaches a "session" object the "req"

NOTE:
To get the userId, the "session.id" needs to be parsed thru the sessionDB (i.e. sessionStore: Mongo or SQL or Redis), to get the associated "userId".
```

### JWT ← using jsonwebtoken

```
jwt.verify( accessToken, ACCESS_TOKEN_SECRET, (err, user) => {
if (err) console.log(err) // eg. invalid token, or expired token
req.user = user
})
// jwt.verify decrypts the accessToken using the ACCESS_TOKEN_SECRET
// the decrypted "user" can be saved in "req.user" to get "req.user.name" or "req.user.email"

NOTE:
Once the "accessToken" is decrypted successfully, the "user" is obtained directly, and there is no separate DB lookup.
```

# 로그아웃시

### Cookie based authnetication

 * 사용자가 로그아웃하면, 서버는 저장되어있던 세션 정보를 지운다(mysql이나 dynamodb같은 별도 저장소에 저장하고 있었다면, 해당 저장소에서 삭제한다.).
 * 브라우저에서 관리되던 쿠키저장소에서도 삭제되게 된다.

### Token based authentication
 * 브라우저에 저장했던 토큰을 삭제처리 해줘야 한다.


# Session Cookies vs. JSON Web tokens — Advantages and Disadvantages

### Session Cookies approach:

##### Advantages
 * 브라우저가 쿠키를 관리해주고, 모든 리퀘스트에 쿠키를 자동으로 추가해주는 편리함(인증이 필요없는 경우에도 보내기는 하지만...).
 * On logout, since the sessionId is removed from the database, we have a precision logout i.e. logout occurs at the moment the sessionId is removed from the sessionDB

##### Disadvantages
 * XSS나 CSRF 공격에 취약하다.
 * Needs an additional DB lookup within the sessionId table to get the userId, DB look ups are relatively slower than JWT decrypting action.
 * Individual servers cannot be scaled separately since they would need to share the sessionDB.


### JWT (JSON Web token) approach:

##### Advantages
 * Since userId is got by decrypting the JWT token, no DB call is required to get userId, so somewhat faster that session approach.
 * Servers can be scaled separately, without the need share sessionDB. This makes the JWT approach a great option for micro-services architecture.
 * The same token can be used to authenticate on different servers (as long as server has the access_token_secret), without the need to share sessionDB,
 * This also allows for a completely separate authentication server, that can be solely responsible to issue “accessTokens” and “refreshTokens”.



##### Disadvantages
 * Since JWT tokens cannot be “invalidated” (without maintaining them in a shared db), in JWT approach the logout length precision is set by the expiration length of the access_token. However the “access_token” lifespan can be kept short (typically 10 to 15 mins), so that tokens are automatically “invalidated” after the duration.
 * Anti-pattern: Sometimes additional and unnecessary information is stored in the JWT. The JWT token should primarily contain user information, and the data authorized to be accessed by that user should be provisioned and managed as a separate service on that respective server.


# Combining both approaches?

 * Some programmers have attempted to combine the Session Cookie based and JWT based approach, and use one for authentication and the other for looking up authorized resources.

 * Session based or JWT based approaches are complete in themselves to effectively manage session management individually, and there is no clear advantage in combining them together (unless there is a specific use case that this solves).


* Session management can be implemented either using the “Session-Cookie” based approach (and is still used in many web applications), or can be implemented using the JWT approach (which is a newer approach, and is frequently used in mobile and web apps)

# 중요사항(기억해둘 내용들)

### Cookie based authentication

 * 쿠키헤더를 세팅할 떄, httpOnly 속성을 추가해서 XSS공격에 대응할 수 있다.
 ```
 Set-Cookie: <cookie-name>=<cookie-value>; Secure
 Set-Cookie: <cookie-name>=<cookie-value>; HttpOnly
 ```

 * SameSite 속성을 사용해서 CSRF 공격을 효과적으로 방지할 수 있다.
   * SmaeSite=Lax <= 디폴트 동작, 브라우저는 쿠키를 cross-site request로 보내지 않는다.
   * SameSite=Strict 브라우저는 samesite request인 경우면 전송한다.
   * SameSite=None cross-site와 same-site에 보내는 것을 허용한다.

```
Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Lax
```

 * 쿠키는 기본적으로 single domain에서 사용된다. 별도로 세팅을 해주지 않는 이상 사용이 어렵다
 * 다른 도메인을 사용하는 사이트간에 통신을 위해서는 쿠키에 whitelist설정을 해줘야 쿠키 전송이 사용이 가능하다.

 * 쿠키를 사용하는 경우는 클라이언트가 브라우저가 아니고, 모바일 앱같은 경우에는 쿠키 관리가 토큰에 비해서 상대적으로 까다롭다.

 * redis같은 메모리디비를 사용하는 방법이 많이 이용되지만, 세션을 관리하는 것은 복잡도가 늘어나는 단점이다.

 * 별도 저장소(mysql이나 dynamodb)에 저장하는 경우가 많기 때문에, 세션에 대한 많은 정보를 저장할 수 있어서, 사용자의 요청에 이용할 수 있다. 토큰의 경에도 (JWT의 claims) 저장이 가능하지만, 네트워크 사용량이 늘어나게 된다.
   + user personalization
   + access control

### Token based authentication
 * 서버의 저장소에는 저장되지 않는다. 서버는 토큰을 만드는 일만 함.

 * 외부 javascript가 embed되는(library같은 것도?) 경우에는 XSS공격에 취약해진다.

 * 노출되게 되면, expire될 때까지 기다려야 한다. 따라서, 되도록 expire되는 시간을 짧게 가져가는게 중요하다.
