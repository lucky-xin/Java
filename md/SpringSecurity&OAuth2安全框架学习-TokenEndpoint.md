```text
权限登录处理类TokenEndpoint详情如下，具体处理登录权限uri /oauth/token在方法postAccessToken之中。
查看postAccessToken方法可知在调用时必须传入认证已经认证的Authentication，否则报错。SpringSecurity构建Authentication思路为先验证客户端
是否正确，如果正确就创建一个对象Authentication，并存入上下文，SecurityContextHolder.getContext().setAuthentication(authResult);
有两种请求方式对应两个不同的Filter
```
[BasicAuthenticationFilter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-BasicAuthenticationFilter.md)
与[ClientCredentialsTokenEndpointFilter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ClientCredentialTokenEndpointFilter.md)
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
        // 需要拿到Authentication。先认证ClientDetails,不然抛出异常
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
		//private boolean isAuthCodeRequest(Map<String, String> parameters) {
		//    return "authorization_code".equals(parameters.get("grant_type")) && parameters.get("code") != null;
		//}
        //是否为刷新token
		if (isRefreshTokenRequest(parameters)) {
			// A refresh token has its own default scopes, so we should ignore any added by the factory here.
			tokenRequest.setScope(OAuth2Utils.parseParameterList(parameters.get(OAuth2Utils.SCOPE)));
		}
        //private boolean isRefreshTokenRequest(Map<String, String> parameters) {
		//    return "refresh_token".equals(parameters.get("grant_type")) && parameters.get("refresh_token") != null;
		//}
        //getTokenGranter()返回CompositeTokenGranter
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

	@ExceptionHandler(HttpRequestMethodNotSupportedException.class)
	public ResponseEntity<OAuth2Exception> handleHttpRequestMethodNotSupportedException(HttpRequestMethodNotSupportedException e) throws Exception {
		if (logger.isInfoEnabled()) {
			logger.info("Handling error: " + e.getClass().getSimpleName() + ", " + e.getMessage());
		}
	    return getExceptionTranslator().translate(e);
	}
	
	@ExceptionHandler(Exception.class)
	public ResponseEntity<OAuth2Exception> handleException(Exception e) throws Exception {
		if (logger.isErrorEnabled()) {
			logger.error("Handling error: " + e.getClass().getSimpleName() + ", " + e.getMessage(), e);
		}
		return getExceptionTranslator().translate(e);
	}
	
	@ExceptionHandler(ClientRegistrationException.class)
	public ResponseEntity<OAuth2Exception> handleClientRegistrationException(Exception e) throws Exception {
		if (logger.isWarnEnabled()) {
			logger.warn("Handling error: " + e.getClass().getSimpleName() + ", " + e.getMessage());
		}
		return getExceptionTranslator().translate(new BadClientCredentialsException());
	}

	@ExceptionHandler(OAuth2Exception.class)
	public ResponseEntity<OAuth2Exception> handleException(OAuth2Exception e) throws Exception {
		if (logger.isWarnEnabled()) {
			logger.warn("Handling error: " + e.getClass().getSimpleName() + ", " + e.getMessage());
		}
		return getExceptionTranslator().translate(e);
	}

	private ResponseEntity<OAuth2AccessToken> getResponse(OAuth2AccessToken accessToken) {
		HttpHeaders headers = new HttpHeaders();
		headers.set("Cache-Control", "no-store");
		headers.set("Pragma", "no-cache");
		headers.set("Content-Type", "application/json;charset=UTF-8");
		return new ResponseEntity<OAuth2AccessToken>(accessToken, headers, HttpStatus.OK);
	}

	private boolean isRefreshTokenRequest(Map<String, String> parameters) {
		return "refresh_token".equals(parameters.get("grant_type")) && parameters.get("refresh_token") != null;
	}

	private boolean isAuthCodeRequest(Map<String, String> parameters) {
		return "authorization_code".equals(parameters.get("grant_type")) && parameters.get("code") != null;
	}

	public void setOAuth2RequestValidator(OAuth2RequestValidator oAuth2RequestValidator) {
		this.oAuth2RequestValidator = oAuth2RequestValidator;
	}

	public void setAllowedRequestMethods(Set<HttpMethod> allowedRequestMethods) {
		this.allowedRequestMethods = allowedRequestMethods;
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
```text
tokenGranters初始化添加如下TokenGranter实现类：
```
* [AuthorizationCodeTokenGranter]()授权码模式（用于三方登录）,对应uri请求为/oauth/authorize,grant_type为authorization_code时使用该模式
* [ResourceOwnerPasswordTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ResourceOwnerPasswordTokenGranter.md)账号密码模式,对应uri请求/oauth/token，grant_type为password时使用该模式
* [RefreshTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-RefreshTokenGranter.md),刷新token模式,对应uri请求/oauth/token,grant_type为refresh_token时使用该模式
* [ClientCredentialsTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ClientCredentialsTokenGranter.md)模式用于用client_id和client_secret来获取授权,grant_type为client_credentials时使用该模式

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

