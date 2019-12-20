```text
CheckTokenEndpoint用于校验token是否合法。在SpringCloud微服务之中使用CheckTokenEndpoint就特别方便
请求示例：
http://127.0.0.1:1000/oauth/check_token?token=ce63c22c-7f5d-4102-91f5-4a23174daace&client_id=aaa&client_secret=bbb
为了安全一定要在继承自org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter添加如下配置
```
```java
	@Override
	public void configure(AuthorizationServerSecurityConfigurer oauthServer) {
		oauthServer
				.allowFormAuthenticationForClients()
				// /oauth/check_token 需验证才能访问
				.checkTokenAccess("isAuthenticated()");
	}	

```
```java
package org.springframework.security.oauth2.provider.endpoint;

/**
 * Controller which decodes access tokens for clients who are not able to do so (or where opaque token values are used).
 * 
 * @author Luke Taylor
 * @author Joel D'sa
 */
@FrameworkEndpoint
public class CheckTokenEndpoint {

	private ResourceServerTokenServices resourceServerTokenServices;

	private AccessTokenConverter accessTokenConverter = new CheckTokenAccessTokenConverter();

	protected final Log logger = LogFactory.getLog(getClass());

	private WebResponseExceptionTranslator<OAuth2Exception> exceptionTranslator = new DefaultWebResponseExceptionTranslator();

	public CheckTokenEndpoint(ResourceServerTokenServices resourceServerTokenServices) {
		this.resourceServerTokenServices = resourceServerTokenServices;
	}
	
	/**
	 * @param exceptionTranslator the exception translator to set
	 */
	public void setExceptionTranslator(WebResponseExceptionTranslator<OAuth2Exception> exceptionTranslator) {
		this.exceptionTranslator = exceptionTranslator;
	}

	/**
	 * @param accessTokenConverter the accessTokenConverter to set
	 */
	public void setAccessTokenConverter(AccessTokenConverter accessTokenConverter) {
		this.accessTokenConverter = accessTokenConverter;
	}

	@RequestMapping(value = "/oauth/check_token")
	@ResponseBody
	public Map<String, ?> checkToken(@RequestParam("token") String value) {

		OAuth2AccessToken token = resourceServerTokenServices.readAccessToken(value);
		if (token == null) {
			throw new InvalidTokenException("Token was not recognised");
		}

		if (token.isExpired()) {
			throw new InvalidTokenException("Token has expired");
		}
        //根据token值查询缓存获取token绑定的权限信息
		OAuth2Authentication authentication = resourceServerTokenServices.loadAuthentication(token.getValue());

		return accessTokenConverter.convertAccessToken(token, authentication);
	}

	@ExceptionHandler(InvalidTokenException.class)
	public ResponseEntity<OAuth2Exception> handleException(Exception e) throws Exception {
		logger.info("Handling error: " + e.getClass().getSimpleName() + ", " + e.getMessage());
		// This isn't an oauth resource, so we don't want to send an
		// unauthorized code here. The client has already authenticated
		// successfully with basic auth and should just
		// get back the invalid token error.
		@SuppressWarnings("serial")
		InvalidTokenException e400 = new InvalidTokenException(e.getMessage()) {
			@Override
			public int getHttpErrorCode() {
				return 400;
			}
		};
		return exceptionTranslator.translate(e400);
	}

	static class CheckTokenAccessTokenConverter implements AccessTokenConverter {
		private final AccessTokenConverter accessTokenConverter;

		CheckTokenAccessTokenConverter() {
			this(new DefaultAccessTokenConverter());
		}

		CheckTokenAccessTokenConverter(AccessTokenConverter accessTokenConverter) {
			this.accessTokenConverter = accessTokenConverter;
		}

		@Override
		public Map<String, ?> convertAccessToken(OAuth2AccessToken token, OAuth2Authentication authentication) {
			Map<String, Object> claims = (Map<String, Object>) this.accessTokenConverter.convertAccessToken(token, authentication);

			// gh-1070
			claims.put("active", true);		// Always true if token exists and not expired

			return claims;
		}

		@Override
		public OAuth2AccessToken extractAccessToken(String value, Map<String, ?> map) {
			return this.accessTokenConverter.extractAccessToken(value, map);
		}

		@Override
		public OAuth2Authentication extractAuthentication(Map<String, ?> map) {
			return this.accessTokenConverter.extractAuthentication(map);
		}
	}
}

```