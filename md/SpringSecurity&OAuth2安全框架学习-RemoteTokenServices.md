```text
RemoteTokenServices用于其它微服务模块向权限认证中心校验接口调用者是否有权限访问，具体实现为使用org.springframework.web.client.RestTemplate
封装client-id,client-secret,请求参数为，请求头值为Basic 开头的请求头 -> Authorization=Basic " + new String(Base64.encode(creds.getBytes("UTF-8")) 
并请求http://ip:port/oauth/check_token校验token,ip为认证中心的ip,port为认证中心port,从配置文件获取，例如某个微服务的配置文件application.yml
配置了如下的权限认证中心
```
```yaml
## spring security 配置
security:
  oauth2:
    client:
      client-id: ENC(ltJPpR50wT0oIY9kfOe1Iw==)
      client-secret: ENC(ltJPpR50wT0oIY9kfOe1Iw==)
      scope: server
      # 默认放行url,如果子模块重写这里的配置就会被覆盖
      ignore-urls:
        - /actuator/**
        - /v2/api-docs
        - /webjars/springfox-swagger-ui/**
    resource:
      loadBalanced: true
      token-info-uri: http://${AUTH2_ENDPOINT:datainsights-auth:6000}/oauth/check_token
```
### http://ip:port/oauth/check_token 请求发送到权限认证中心的
[CheckTokenEndpoint](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-CheckTokenEndpoint.md)
成功访问认证中心之后如果token有权限则调用[DefaultAccessTokenConverter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-DefaultAccessTokenConverter.md)转换token信息
```java
 public class RemoteTokenServices implements ResourceServerTokenServices {
 
 	protected final Log logger = LogFactory.getLog(getClass());
 
 	private RestOperations restTemplate;
 
 	private String checkTokenEndpointUrl;
 
 	private String clientId;
 
 	private String clientSecret;
 
 	private String tokenName = "token";
 
 	private AccessTokenConverter tokenConverter = new DefaultAccessTokenConverter();
 
 	public RemoteTokenServices() {
 		restTemplate = new RestTemplate();
 		((RestTemplate) restTemplate).setErrorHandler(new DefaultResponseErrorHandler() {
 			@Override
 			// Ignore 400
 			public void handleError(ClientHttpResponse response) throws IOException {
 				if (response.getRawStatusCode() != 400) {
 					super.handleError(response);
 				}
 			}
 		});
 	}
 	
 	@Override
 	public OAuth2Authentication loadAuthentication(String accessToken) throws AuthenticationException, InvalidTokenException {
 
 		MultiValueMap<String, String> formData = new LinkedMultiValueMap<String, String>();
 		formData.add(tokenName, accessToken);
 		HttpHeaders headers = new HttpHeaders();
 		headers.set("Authorization", getAuthorizationHeader(clientId, clientSecret));
        // 封装请求体，调用认证中心接口校验token
 		Map<String, Object> map = postForMap(checkTokenEndpointUrl, formData, headers);
 
 		if (map.containsKey("error")) {
 			if (logger.isDebugEnabled()) {
 				logger.debug("check_token returned error: " + map.get("error"));
 			}
 			throw new InvalidTokenException(accessToken);
 		}
 
 		// gh-838
 		if (map.containsKey("active") && !"true".equals(String.valueOf(map.get("active")))) {
 			logger.debug("check_token returned active attribute: " + map.get("active"));
 			throw new InvalidTokenException(accessToken);
 		}
 
 		return tokenConverter.extractAuthentication(map);
 	}
 
 	@Override
 	public OAuth2AccessToken readAccessToken(String accessToken) {
 		throw new UnsupportedOperationException("Not supported: read access token");
 	}
 
 	private String getAuthorizationHeader(String clientId, String clientSecret) {
 
 		if(clientId == null || clientSecret == null) {
 			logger.warn("Null Client ID or Client Secret detected. Endpoint that requires authentication will reject request with 401 error.");
 		}
 
 		String creds = String.format("%s:%s", clientId, clientSecret);
 		try {
 			return "Basic " + new String(Base64.encode(creds.getBytes("UTF-8")));
 		}
 		catch (UnsupportedEncodingException e) {
 			throw new IllegalStateException("Could not convert String");
 		}
 	}
 
 	private Map<String, Object> postForMap(String path, MultiValueMap<String, String> formData, HttpHeaders headers) {
 		if (headers.getContentType() == null) {
 			headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
 		}
 		@SuppressWarnings("rawtypes")
 		Map map = restTemplate.exchange(path, HttpMethod.POST,
 				new HttpEntity<MultiValueMap<String, String>>(formData, headers), Map.class).getBody();
 		@SuppressWarnings("unchecked")
 		Map<String, Object> result = map;
 		return result;
 	}
 
 }

```