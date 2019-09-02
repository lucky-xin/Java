```text
ClientCredentialsTokenGranter(grant_type=client_credentials)模式用于根据client_id和client_secret来获取授权。
请求示例：
1. http://127.0.0.1:1000/oauth/token?grant_type=client_credentials&scope=server&client_id=aaa&client_secret=bbb 可以表单提交
2. http://127.0.0.1:1000/oauth/token?grant_type=client_credentials&scope=server
   请求头 Authorization = Basic YWFhOmJiYg==
    Basic 空格 base64(aaa:bbb) -> Basic YWFhOmJiYg==
具体实现,源码如下
```
```java
package org.springframework.security.oauth2.provider.client;

/**
 * @author Dave Syer
 * 
 */
public class ClientCredentialsTokenGranter extends AbstractTokenGranter {

	private static final String GRANT_TYPE = "client_credentials";
	private boolean allowRefresh = false;

	public ClientCredentialsTokenGranter(AuthorizationServerTokenServices tokenServices,
			ClientDetailsService clientDetailsService, OAuth2RequestFactory requestFactory) {
		this(tokenServices, clientDetailsService, requestFactory, GRANT_TYPE);
	}

	protected ClientCredentialsTokenGranter(AuthorizationServerTokenServices tokenServices,
			ClientDetailsService clientDetailsService, OAuth2RequestFactory requestFactory, String grantType) {
		super(tokenServices, clientDetailsService, requestFactory, grantType);
	}
	
	public void setAllowRefresh(boolean allowRefresh) {
		this.allowRefresh = allowRefresh;
	}

	@Override
	public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
		OAuth2AccessToken token = super.grant(grantType, tokenRequest);
		if (token != null) {
			DefaultOAuth2AccessToken norefresh = new DefaultOAuth2AccessToken(token);
			// The spec says that client credentials should not be allowed to get a refresh token
			if (!allowRefresh) {
				norefresh.setRefreshToken(null);
			}
			token = norefresh;
		}
		return token;
	}

}
```