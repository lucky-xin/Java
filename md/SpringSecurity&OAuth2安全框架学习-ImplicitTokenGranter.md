```text
ImplicitTokenGranter用于处理请求
uri /oauth/authorize?scope=server&response_type=token&redirect_uri=http://www.baidu.com&client_id=aaa
实现方式为：用户调用/oauth/authorize时因为没有访问权限(/oauth/authorize不能公开访问情况下)，
会被org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint重定向到登录页面进行登录验证生成Authentication
response_type=token则控制org.springframework.security.oauth2.provider.endpoint.AuthorizationEndpoint直接返回该认证成功的Authentication
具体实现,源码如下
```
```java
package org.springframework.security.oauth2.provider.implicit;

/**
 * @author Dave Syer
 * 
 */
public class ImplicitTokenGranter extends AbstractTokenGranter {

	private static final String GRANT_TYPE = "implicit";

	public ImplicitTokenGranter(AuthorizationServerTokenServices tokenServices, ClientDetailsService clientDetailsService, OAuth2RequestFactory requestFactory) {
		this(tokenServices, clientDetailsService, requestFactory, GRANT_TYPE);
	}

	protected ImplicitTokenGranter(AuthorizationServerTokenServices tokenServices, ClientDetailsService clientDetailsService,
			OAuth2RequestFactory requestFactory, String grantType) {
		super(tokenServices, clientDetailsService, requestFactory, grantType);
	}

	@Override
	protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest clientToken) {
        // 从上下文获取认证成功的Authentication并使用OAuth2Authentication封装，然后返回
		Authentication userAuth = SecurityContextHolder.getContext().getAuthentication();
		if (userAuth==null || !userAuth.isAuthenticated()) {
			throw new InsufficientAuthenticationException("There is no currently logged in user");
		}
		Assert.state(clientToken instanceof ImplicitTokenRequest, "An ImplicitTokenRequest is required here. Caller needs to wrap the TokenRequest.");
		
		OAuth2Request requestForStorage = ((ImplicitTokenRequest)clientToken).getOAuth2Request();
		
		return new OAuth2Authentication(requestForStorage, userAuth);

	}
	
	@SuppressWarnings("deprecation")
	public void setImplicitGrantService(ImplicitGrantService service) {
	}

}

```