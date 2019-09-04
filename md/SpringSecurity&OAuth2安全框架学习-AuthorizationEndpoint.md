```text
AuthorizationEndpoint为授权码授权认证端点，以及授权三方登录有两种模式code和token,response_type必须为code或者token,必须传入client_id
并且必须进行登录认证成功之后才能进行授权
实现方式：用户调用/oauth/authorize时因为没有访问权限(/oauth/authorize不能公开访问情况下)，
抛出异常被
```
[org.springframework.security.web.access.ExceptionTranslationFilter]()
```text
拦截处理，使用org.springframework.security.web.savedrequest.RequestCache保存当前的request以及response对象，使用
org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint重定向到登录页面进行登录验证生成Authentication，
认证成功之后从RequestCache之中获取认证之前的请求url,在进行重定向此时请求上下文已经有Authentication，已经被授权访问。

后面调用AuthorizationEndpoint的authorize方法以及approveOrDeny方法 的参数Principal 就是已经认证成功的Authentication，
1.token 模式。获取授权的token用于授权登录（/oauth/token?grant_type=authorization_code进行登录认证）
2.code 模式。授权之后生成一个code并重定向到redirect_uri(格式为redirect_uri?code=生成的code)
请求示例：
请求方法：GET
http://127.0.0.1:1000/oauth/authorize?scope=server&response_type=code&redirect_uri=www.baidu.com&client_id=aaa
```
```java
package org.springframework.security.oauth2.provider.endpoint;

/**
 * <p>
 * Implementation of the Authorization Endpoint from the OAuth2 specification. Accepts authorization requests, and
 * handles user approval if the grant type is authorization code. The tokens themselves are obtained from the
 * {@link TokenEndpoint Token Endpoint}, except in the implicit grant type (where they come from the Authorization
 * Endpoint via <code>response_type=token</code>.
 * </p>
 * 
 * <p>
 * This endpoint should be secured so that it is only accessible to fully authenticated users (as a minimum requirement)
 * since it represents a request from a valid user to act on his or her behalf.
 * </p>
 * 
 * @author Dave Syer
 * @author Vladimir Kryachko
 * 
 */
@FrameworkEndpoint
@SessionAttributes({AuthorizationEndpoint.AUTHORIZATION_REQUEST_ATTR_NAME, AuthorizationEndpoint.ORIGINAL_AUTHORIZATION_REQUEST_ATTR_NAME})
public class AuthorizationEndpoint extends AbstractEndpoint {
	static final String AUTHORIZATION_REQUEST_ATTR_NAME = "authorizationRequest";

	static final String ORIGINAL_AUTHORIZATION_REQUEST_ATTR_NAME = "org.springframework.security.oauth2.provider.endpoint.AuthorizationEndpoint.ORIGINAL_AUTHORIZATION_REQUEST";

	private AuthorizationCodeServices authorizationCodeServices = new InMemoryAuthorizationCodeServices();

	private RedirectResolver redirectResolver = new DefaultRedirectResolver();

	private UserApprovalHandler userApprovalHandler = new DefaultUserApprovalHandler();

	private SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();

	private OAuth2RequestValidator oauth2RequestValidator = new DefaultOAuth2RequestValidator();

	private String userApprovalPage = "forward:/oauth/confirm_access";

	private String errorPage = "forward:/oauth/error";

	private Object implicitLock = new Object();

	public void setSessionAttributeStore(SessionAttributeStore sessionAttributeStore) {
		this.sessionAttributeStore = sessionAttributeStore;
	}

	public void setErrorPage(String errorPage) {
		this.errorPage = errorPage;
	}

	@RequestMapping(value = "/oauth/authorize")
	public ModelAndView authorize(Map<String, Object> model, @RequestParam Map<String, String> parameters,
			SessionStatus sessionStatus, Principal principal) {

		// Pull out the authorization request first, using the OAuth2RequestFactory. All further logic should
		// query off of the authorization request instead of referring back to the parameters map. The contents of the
		// parameters map will be stored without change in the AuthorizationRequest object once it is created.
		AuthorizationRequest authorizationRequest = getOAuth2RequestFactory().createAuthorizationRequest(parameters);

		Set<String> responseTypes = authorizationRequest.getResponseTypes();
		// 校验response_type 必须为token或者code
		if (!responseTypes.contains("token") && !responseTypes.contains("code")) {
			throw new UnsupportedResponseTypeException("Unsupported response types: " + responseTypes);
		}
	    // 必须传入client_id
		if (authorizationRequest.getClientId() == null) {
			throw new InvalidClientException("A client id must be provided");
		}

		try {
		    //必须先认证拿到Authentication
			if (!(principal instanceof Authentication) || !((Authentication) principal).isAuthenticated()) {
				throw new InsufficientAuthenticationException(
						"User must be authenticated with Spring Security before authorization can be completed.");
			}

			ClientDetails client = getClientDetailsService().loadClientByClientId(authorizationRequest.getClientId());
			// 获取redirect_uri并使用redirectResolver来匹配该客户端所配置的redirect_uri如果不匹配则报错
			// The resolved redirect URI is either the redirect_uri from the parameters or the one from
			// clientDetails. Either way we need to store it on the AuthorizationRequest.
			String redirectUriParameter = authorizationRequest.getRequestParameters().get(OAuth2Utils.REDIRECT_URI);
			String resolvedRedirect = redirectResolver.resolveRedirect(redirectUriParameter, client);
			if (!StringUtils.hasText(resolvedRedirect)) {
				throw new RedirectMismatchException(
						"A redirectUri must be either supplied or preconfigured in the ClientDetails");
			}
			authorizationRequest.setRedirectUri(resolvedRedirect);

			// We intentionally only validate the parameters requested by the client (ignoring any data that may have
			// been added to the request by the manager).
			oauth2RequestValidator.validateScope(authorizationRequest, client);

			// Some systems may allow for approval decisions to be remembered or approved by default. Check for
			// such logic here, and set the approved flag on the authorization request accordingly.
			authorizationRequest = userApprovalHandler.checkForPreApproval(authorizationRequest,
					(Authentication) principal);
			// TODO: is this call necessary?
			boolean approved = userApprovalHandler.isApproved(authorizationRequest, (Authentication) principal);
			authorizationRequest.setApproved(approved);

			// Validation is all done, so we can check for auto approval...
			if (authorizationRequest.isApproved()) {
			    //获取登录使用的token
				if (responseTypes.contains("token")) {
					return getImplicitGrantResponse(authorizationRequest);
				}
				// 获取授权码code
				if (responseTypes.contains("code")) {
					return new ModelAndView(getAuthorizationCodeResponse(authorizationRequest,
							(Authentication) principal));
				}
			}

			// Store authorizationRequest AND an immutable Map of authorizationRequest in session
			// which will be used to validate against in approveOrDeny()
			model.put(AUTHORIZATION_REQUEST_ATTR_NAME, authorizationRequest);
			model.put(ORIGINAL_AUTHORIZATION_REQUEST_ATTR_NAME, unmodifiableMap(authorizationRequest));

			return getUserApprovalPageResponse(model, authorizationRequest, (Authentication) principal);

		}
		catch (RuntimeException e) {
			sessionStatus.setComplete();
			throw e;
		}

	}

	@RequestMapping(value = "/oauth/authorize", method = RequestMethod.POST, params = OAuth2Utils.USER_OAUTH_APPROVAL)
	public View approveOrDeny(@RequestParam Map<String, String> approvalParameters, Map<String, ?> model,
			SessionStatus sessionStatus, Principal principal) {

		if (!(principal instanceof Authentication)) {
			sessionStatus.setComplete();
			throw new InsufficientAuthenticationException(
					"User must be authenticated with Spring Security before authorizing an access token.");
		}

		AuthorizationRequest authorizationRequest = (AuthorizationRequest) model.get(AUTHORIZATION_REQUEST_ATTR_NAME);

		if (authorizationRequest == null) {
			sessionStatus.setComplete();
			throw new InvalidRequestException("Cannot approve uninitialized authorization request.");
		}

		// Check to ensure the Authorization Request was not modified during the user approval step
		@SuppressWarnings("unchecked")
		Map<String, Object> originalAuthorizationRequest = (Map<String, Object>) model.get(ORIGINAL_AUTHORIZATION_REQUEST_ATTR_NAME);
		if (isAuthorizationRequestModified(authorizationRequest, originalAuthorizationRequest)) {
			throw new InvalidRequestException("Changes were detected from the original authorization request.");
		}

		try {
			Set<String> responseTypes = authorizationRequest.getResponseTypes();

			authorizationRequest.setApprovalParameters(approvalParameters);
			authorizationRequest = userApprovalHandler.updateAfterApproval(authorizationRequest,
					(Authentication) principal);
			boolean approved = userApprovalHandler.isApproved(authorizationRequest, (Authentication) principal);
			authorizationRequest.setApproved(approved);

			if (authorizationRequest.getRedirectUri() == null) {
				sessionStatus.setComplete();
				throw new InvalidRequestException("Cannot approve request when no redirect URI is provided.");
			}

			if (!authorizationRequest.isApproved()) {
				return new RedirectView(getUnsuccessfulRedirect(authorizationRequest,
						new UserDeniedAuthorizationException("User denied access"), responseTypes.contains("token")),
						false, true, false);
			}

			if (responseTypes.contains("token")) {
				return getImplicitGrantResponse(authorizationRequest).getView();
			}

			return getAuthorizationCodeResponse(authorizationRequest, (Authentication) principal);
		}
		finally {
			sessionStatus.setComplete();
		}

	}


	// We need explicit approval from the user.
	private ModelAndView getUserApprovalPageResponse(Map<String, Object> model,
			AuthorizationRequest authorizationRequest, Authentication principal) {
		if (logger.isDebugEnabled()) {
			logger.debug("Loading user approval page: " + userApprovalPage);
		}
		model.putAll(userApprovalHandler.getUserApprovalRequest(authorizationRequest, principal));
		return new ModelAndView(userApprovalPage, model);
	}

	// We can grant a token and return it with implicit approval.
	private ModelAndView getImplicitGrantResponse(AuthorizationRequest authorizationRequest) {
		try {
			TokenRequest tokenRequest = getOAuth2RequestFactory().createTokenRequest(authorizationRequest, "implicit");
			OAuth2Request storedOAuth2Request = getOAuth2RequestFactory().createOAuth2Request(authorizationRequest);
			OAuth2AccessToken accessToken = getAccessTokenForImplicitGrant(tokenRequest, storedOAuth2Request);
			if (accessToken == null) {
				throw new UnsupportedResponseTypeException("Unsupported response type: token");
			}
			return new ModelAndView(new RedirectView(appendAccessToken(authorizationRequest, accessToken), false, true,
					false));
		}
		catch (OAuth2Exception e) {
			return new ModelAndView(new RedirectView(getUnsuccessfulRedirect(authorizationRequest, e, true), false,
					true, false));
		}
	}

	private OAuth2AccessToken getAccessTokenForImplicitGrant(TokenRequest tokenRequest,
			OAuth2Request storedOAuth2Request) {
		OAuth2AccessToken accessToken = null;
		// These 1 method calls have to be atomic, otherwise the ImplicitGrantService can have a race condition where
		// one thread removes the token request before another has a chance to redeem it.
		synchronized (this.implicitLock) {
			accessToken = getTokenGranter().grant("implicit",
					new ImplicitTokenRequest(tokenRequest, storedOAuth2Request));
		}
		return accessToken;
	}

	private View getAuthorizationCodeResponse(AuthorizationRequest authorizationRequest, Authentication authUser) {
		try {
			return new RedirectView(getSuccessfulRedirect(authorizationRequest,
					generateCode(authorizationRequest, authUser)), false, true, false);
		}
		catch (OAuth2Exception e) {
			return new RedirectView(getUnsuccessfulRedirect(authorizationRequest, e, false), false, true, false);
		}
	}

	private String appendAccessToken(AuthorizationRequest authorizationRequest, OAuth2AccessToken accessToken) {

		Map<String, Object> vars = new LinkedHashMap<String, Object>();
		Map<String, String> keys = new HashMap<String, String>();

		if (accessToken == null) {
			throw new InvalidRequestException("An implicit grant could not be made");
		}

		vars.put("access_token", accessToken.getValue());
		vars.put("token_type", accessToken.getTokenType());
		String state = authorizationRequest.getState();

		if (state != null) {
			vars.put("state", state);
		}
		Date expiration = accessToken.getExpiration();
		if (expiration != null) {
			long expires_in = (expiration.getTime() - System.currentTimeMillis()) / 1000;
			vars.put("expires_in", expires_in);
		}
		String originalScope = authorizationRequest.getRequestParameters().get(OAuth2Utils.SCOPE);
		if (originalScope == null || !OAuth2Utils.parseParameterList(originalScope).equals(accessToken.getScope())) {
			vars.put("scope", OAuth2Utils.formatParameterList(accessToken.getScope()));
		}
		Map<String, Object> additionalInformation = accessToken.getAdditionalInformation();
		for (String key : additionalInformation.keySet()) {
			Object value = additionalInformation.get(key);
			if (value != null) {
				keys.put("extra_" + key, key);
				vars.put("extra_" + key, value);
			}
		}
		// Do not include the refresh token (even if there is one)
		return append(authorizationRequest.getRedirectUri(), vars, keys, true);
	}

	private String generateCode(AuthorizationRequest authorizationRequest, Authentication authentication)
			throws AuthenticationException {

		try {

			OAuth2Request storedOAuth2Request = getOAuth2RequestFactory().createOAuth2Request(authorizationRequest);

			OAuth2Authentication combinedAuth = new OAuth2Authentication(storedOAuth2Request, authentication);
			String code = authorizationCodeServices.createAuthorizationCode(combinedAuth);

			return code;

		}
		catch (OAuth2Exception e) {

			if (authorizationRequest.getState() != null) {
				e.addAdditionalInformation("state", authorizationRequest.getState());
			}

			throw e;

		}
	}

	private String getSuccessfulRedirect(AuthorizationRequest authorizationRequest, String authorizationCode) {

		if (authorizationCode == null) {
			throw new IllegalStateException("No authorization code found in the current request scope.");
		}

		Map<String, String> query = new LinkedHashMap<String, String>();
		query.put("code", authorizationCode);

		String state = authorizationRequest.getState();
		if (state != null) {
			query.put("state", state);
		}

		return append(authorizationRequest.getRedirectUri(), query, false);
	}

	private String getUnsuccessfulRedirect(AuthorizationRequest authorizationRequest, OAuth2Exception failure,
			boolean fragment) {

		if (authorizationRequest == null || authorizationRequest.getRedirectUri() == null) {
			// we have no redirect for the user. very sad.
			throw new UnapprovedClientAuthenticationException("Authorization failure, and no redirect URI.", failure);
		}

		Map<String, String> query = new LinkedHashMap<String, String>();

		query.put("error", failure.getOAuth2ErrorCode());
		query.put("error_description", failure.getMessage());

		if (authorizationRequest.getState() != null) {
			query.put("state", authorizationRequest.getState());
		}

		if (failure.getAdditionalInformation() != null) {
			for (Map.Entry<String, String> additionalInfo : failure.getAdditionalInformation().entrySet()) {
				query.put(additionalInfo.getKey(), additionalInfo.getValue());
			}
		}

		return append(authorizationRequest.getRedirectUri(), query, fragment);

	}

	private String append(String base, Map<String, ?> query, boolean fragment) {
		return append(base, query, null, fragment);
	}

	private String append(String base, Map<String, ?> query, Map<String, String> keys, boolean fragment) {

		UriComponentsBuilder template = UriComponentsBuilder.newInstance();
		UriComponentsBuilder builder = UriComponentsBuilder.fromUriString(base);
		URI redirectUri;
		try {
			// assume it's encoded to start with (if it came in over the wire)
			redirectUri = builder.build(true).toUri();
		}
		catch (Exception e) {
			// ... but allow client registrations to contain hard-coded non-encoded values
			redirectUri = builder.build().toUri();
			builder = UriComponentsBuilder.fromUri(redirectUri);
		}
		template.scheme(redirectUri.getScheme()).port(redirectUri.getPort()).host(redirectUri.getHost())
				.userInfo(redirectUri.getUserInfo()).path(redirectUri.getPath());

		if (fragment) {
			StringBuilder values = new StringBuilder();
			if (redirectUri.getFragment() != null) {
				String append = redirectUri.getFragment();
				values.append(append);
			}
			for (String key : query.keySet()) {
				if (values.length() > 0) {
					values.append("&");
				}
				String name = key;
				if (keys != null && keys.containsKey(key)) {
					name = keys.get(key);
				}
				values.append(name + "={" + key + "}");
			}
			if (values.length() > 0) {
				template.fragment(values.toString());
			}
			UriComponents encoded = template.build().expand(query).encode();
			builder.fragment(encoded.getFragment());
		}
		else {
			for (String key : query.keySet()) {
				String name = key;
				if (keys != null && keys.containsKey(key)) {
					name = keys.get(key);
				}
				template.queryParam(name, "{" + key + "}");
			}
			template.fragment(redirectUri.getFragment());
			UriComponents encoded = template.build().expand(query).encode();
			builder.query(encoded.getQuery());
		}

		return builder.build().toUriString();

	}
}

```