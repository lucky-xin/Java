```text
令牌端点具体处理登录权限uri /oauth/token查看方法org.springframework.security.oauth2.provider.endpoint.TokenEndpoint#postAccessToken。
方法可知在调用时必须传入认证Authentication，否则报错。
SpringSecurity构建Authentication思路为先验证客户端（client_id,client_secret）是否正确，如果正确就创建一个对象Authentication，并存入上下文，
SecurityContextHolder.getContext().setAuthentication(authResult);
有两种请求方式对应两个不同的Filter
```
[BasicAuthenticationFilter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-BasicAuthenticationFilter.md)
```text
拦截如下请求url:
http://127.0.0.1:1000/oauth/token?username=aaa&password=123456&grant_type=password&scope=read-write
请求头: Authorization= Basic aaaaaaaadad
base64加密client_id:client_secret字符串
aaaaaaaadad = base64(client_id:client_secret) 
```
与[ClientCredentialsTokenEndpointFilter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ClientCredentialTokenEndpointFilter.md)
```text
拦截如下请求url:
http://127.0.0.1:1000/oauth/token?username=aaa&password=123456&grant_type=password&scope=read-write&client_id=aaa&client_secret=bbb
```
TokenEndpoint用到的用户信息定义接口[UserDetails](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-UserDetails.md)
和[客户端定义接口](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ClientDetails.md)
客户基本信息表对应ClientDetails
```sql
CREATE TABLE `oauth_client_details`  (
  `client_id` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `resource_ids` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `client_secret` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `scope` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `authorized_grant_types` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `web_server_redirect_uri` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `authorities` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `access_token_validity` int(11) NULL DEFAULT NULL,
  `refresh_token_validity` int(11) NULL DEFAULT NULL,
  `additional_information` varchar(4096) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `autoapprove` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL
  PRIMARY KEY (`client_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '终端信息表' ROW_FORMAT = Dynamic;
```

TokenEndpoint 源码如下：
```java
package org.springframework.security.oauth2.provider.endpoint;
/**
 * <p>
 * Endpoint for token requests as described in the OAuth2 spec. Clients post requests with a <code>grant_type</code>
 * parameter (e.g. "authorization_code") and other parameters as determined by the grant type. Supported grant types are
 * handled by the provided {@link #setTokenGranter(org.springframework.security.oauth2.provider.TokenGranter) token
 * granter}.
 * </p>
 * 
 * <p>
 * Clients must be authenticated using a Spring Security {@link Authentication} to access this endpoint, and the client
 * id is extracted from the authentication token. The best way to arrange this (as per the OAuth2 spec) is to use HTTP
 * basic authentication for this endpoint with standard Spring Security support.
 * </p>
 * 
 * @author Dave Syer
 * 
 */
@FrameworkEndpoint
public class TokenEndpoint extends AbstractEndpoint {

	private OAuth2RequestValidator oAuth2RequestValidator = new DefaultOAuth2RequestValidator();

	private Set<HttpMethod> allowedRequestMethods = new HashSet<HttpMethod>(Arrays.asList(HttpMethod.POST));

	@RequestMapping(value = "/oauth/token", method=RequestMethod.GET)
	public ResponseEntity<OAuth2AccessToken> getAccessToken(Principal principal, @RequestParam
	Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
		if (!allowedRequestMethods.contains(HttpMethod.GET)) {
			throw new HttpRequestMethodNotSupportedException("GET");
		}
		return postAccessToken(principal, parameters);
	}
	
	@RequestMapping(value = "/oauth/token", method=RequestMethod.POST)
	public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam
	Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
        // 先认证ClientDetails，拿到Authentication。不然抛出异常
		if (!(principal instanceof Authentication)) {
			throw new InsufficientAuthenticationException(
					"There is no client authentication. Try adding an appropriate authentication filter.");
		}
        // 从Authentication获取ClientDetails
		String clientId = getClientId(principal);
		ClientDetails authenticatedClient = getClientDetailsService().loadClientByClientId(clientId);

		TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(parameters, authenticatedClient);

		if (clientId != null && !clientId.equals("")) {
			// Only validate the client details if a client authenticated during this
			// request.
			if (!clientId.equals(tokenRequest.getClientId())) {
				// double check to make sure that the client ID in the token request is the same as that in the
				// authenticated client
				throw new InvalidClientException("Given client ID does not match authenticated client");
			}
		}
        // 校验Scope对应oauth_client_details表的scope
		if (authenticatedClient != null) {
			oAuth2RequestValidator.validateScope(tokenRequest, authenticatedClient);
		}
        // 校验ClientDetails的grant type对应oauth_client_details表的authorized_grant_types
		if (!StringUtils.hasText(tokenRequest.getGrantType())) {
			throw new InvalidRequestException("Missing grant type");
		}
		if (tokenRequest.getGrantType().equals("implicit")) {
			throw new InvalidGrantException("Implicit grant type not supported from token endpoint");
		}
        // 如果是授权登录模式authorization_code，设置scope为空
		if (isAuthCodeRequest(parameters)) {
			// The scope was requested or determined during the authorization step
			if (!tokenRequest.getScope().isEmpty()) {
				logger.debug("Clearing scope of incoming token request");
				tokenRequest.setScope(Collections.<String> emptySet());
			}
		}
	
        //是否为刷新token，具体判断为 "refresh_token".equals(parameters.get("grant_type")) && parameters.get("refresh_token") != null;
		if (isRefreshTokenRequest(parameters)) {
			// A refresh token has its own default scopes, so we should ignore any added by the factory here.
			tokenRequest.setScope(OAuth2Utils.parseParameterList(parameters.get(OAuth2Utils.SCOPE)));
		}
        //getTokenGranter()返回CompositeTokenGranter
		// CompositeTokenGranter 遍历所有的TokenGranter实现类如果某个类处理之后返回OAuth2AccessToken不为null,则直接返回此OAuth2AccessToken
        OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
		if (token == null) {
			throw new UnsupportedGrantTypeException("Unsupported grant type: " + tokenRequest.getGrantType());
		}

		return getResponse(token);
	}

	/**
	 * @param principal the currently authentication principal
	 * @return a client id if there is one in the principal
	 */
	protected String getClientId(Principal principal) {
		Authentication client = (Authentication) principal;
		if (!client.isAuthenticated()) {
			throw new InsufficientAuthenticationException("The client is not authenticated.");
		}
		String clientId = client.getName();
		if (client instanceof OAuth2Authentication) {
			// Might be a client and user combined authentication
			clientId = ((OAuth2Authentication) client).getOAuth2Request().getClientId();
		}
		return clientId;
	}
}

```
# 委派到CompositeTokenGranter遍历所有持有的TokenGranter实现类
```java
package org.springframework.security.oauth2.provider;

/**
 * @author Dave Syer
 * 
 */
public class CompositeTokenGranter implements TokenGranter {

	private final List<TokenGranter> tokenGranters;

	public CompositeTokenGranter(List<TokenGranter> tokenGranters) {
		this.tokenGranters = new ArrayList<TokenGranter>(tokenGranters);
	}
	
	public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {
		for (TokenGranter granter : tokenGranters) {
			OAuth2AccessToken grant = granter.grant(grantType, tokenRequest);
			if (grant!=null) {
				return grant;
			}
		}
		return null;
	}
	
	public void addTokenGranter(TokenGranter tokenGranter) {
		if (tokenGranter == null) {
			throw new IllegalArgumentException("Token granter is null");
		}
		tokenGranters.add(tokenGranter);
	}

}
```
# TokenGranter实现类
* [AuthorizationCodeTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-AuthorizationCodeTokenGranter.md)授权码模式（用于三方登录）,对应uri请求为/oauth/token,grant_type为authorization_code时使用该模式
* [ResourceOwnerPasswordTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ResourceOwnerPasswordTokenGranter.md)账号密码模式,对应uri请求/oauth/token，grant_type为password时使用该模式
* [RefreshTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-RefreshTokenGranter.md),刷新token模式,对应uri请求/oauth/token,grant_type为refresh_token时使用该模式
* [ClientCredentialsTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ClientCredentialsTokenGranter.md)模式用于用client_id和client_secret来获取授权,grant_type为client_credentials时使用该模式
* [ImplicitTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ImplicitTokenGranter.md)该模式用于处理请求uri /oauth/authorize?scope=server&response_type=token&redirect_uri=http://www.baidu.com&client_id=aaa
* [AbstractTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-AbstractTokenGranter.md)
# tokenGranters初始化添加如下
```java
	private List<TokenGranter> getDefaultTokenGranters() {
		ClientDetailsService clientDetails = clientDetailsService();
		AuthorizationServerTokenServices tokenServices = tokenServices();
		AuthorizationCodeServices authorizationCodeServices = authorizationCodeServices();
		OAuth2RequestFactory requestFactory = requestFactory();

		List<TokenGranter> tokenGranters = new ArrayList<TokenGranter>();
		tokenGranters.add(new AuthorizationCodeTokenGranter(tokenServices, authorizationCodeServices, clientDetails,
				requestFactory));
		tokenGranters.add(new RefreshTokenGranter(tokenServices, clientDetails, requestFactory));
		ImplicitTokenGranter implicit = new ImplicitTokenGranter(tokenServices, clientDetails, requestFactory);
		tokenGranters.add(implicit);
		tokenGranters.add(new ClientCredentialsTokenGranter(tokenServices, clientDetails, requestFactory));
		if (authenticationManager != null) {
			tokenGranters.add(new ResourceOwnerPasswordTokenGranter(authenticationManager, tokenServices,
					clientDetails, requestFactory));
		}
		return tokenGranters;
	}
```

