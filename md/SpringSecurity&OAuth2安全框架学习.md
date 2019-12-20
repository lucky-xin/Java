# SpringSecurity&OAuth2安全框架源码学习

# [获取token端点 TokenEndpoint 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-TokenEndpoint.md)
#### [ResourceOwnerPasswordTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ResourceOwnerPasswordTokenGranter.md)账号密码模式,对应uri请求/oauth/token，grant_type为password时使用该模式
#### [AuthorizationCodeTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-AuthorizationCodeTokenGranter.md)授权码模式（用于三方登录）,对应uri请求为/oauth/token,grant_type为authorization_code时使用该模式
#### [RefreshTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-RefreshTokenGranter.md),刷新token模式,对应uri请求/oauth/token,grant_type为refresh_token时使用该模式
#### [ClientCredentialsTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ClientCredentialsTokenGranter.md)模式用于用client_id和client_secret来获取授权,grant_type为client_credentials时使用该模式
#### [ImplicitTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ImplicitTokenGranter.md)该模式用于处理请求uri /oauth/authorize?scope=server&response_type=token&redirect_uri=http://www.baidu.com&client_id=aaa
#### [AbstractTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-AbstractTokenGranter.md)

# [BasicAuthenticationFilter 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-BasicAuthenticationFilter.md)
# [ClientCredentialTokenEndpointFilter 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ClientCredentialTokenEndpointFilter.md)

# [授权端点 AuthorizationEndpoint 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-AuthorizationEndpoint.md)

# [权限过滤器 OAuth2AuthenticationProcessingFilter 源码解析]()
# [校验token端点 CheckTokenEndpoint 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-CheckTokenEndpoint.md)

# [AbstractAuthenticationProcessingFilter 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-AbstractAuthenticationProcessingFilter.md)    

# [DefaultTokenServices 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-DefaultTokenServices.md)
# [ExceptionTranslationFilter 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ExceptionTanslationFilter.md)
# [HttpSessionRequestCache 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-HttpSessionRequestCache.md)
# [LoginUrlAuthenticationEntryPoint 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-LoginUrlAuthenticationEntryPoint.md)
# [LoginUrlAuthenticationEntryPoint 源码解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-OAuth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE.md)
