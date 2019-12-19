```text
ResourceOwnerPasswordTokenGranter(grant_type=password)为账号密码登录模式。
请求示例：
1）.http://127.0.0.1:1000/oauth/token?username=用户名&password=密码=
&grant_type=password&scope=read&client_id=aaa&client_secret=bbb

2).http://127.0.0.1:1000/oauth/token?username=用户名&password=密码=
   &grant_type=password&scope=read
   请求头 Authorization = Basic YWFhOmJiYg==
    Basic 空格 base64(aaa:bbb) -> Basic YWFhOmJiYg==
```
先执行超类[AbstractTokenGranter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-AbstractTokenGranter.md)
的grant方法,ResourceOwnerPasswordTokenGranter重写了getOAuth2Authentication方法，检验用户信息
源码如下
```java
    //类org.springframework.security.oauth2.provider.token.AbstractTokenGranter
    @Override
	public OAuth2AccessToken grant(String grantType, TokenRequest tokenRequest) {

		if (!this.grantType.equals(grantType)) {
			return null;
		}
		// 获取客户端信息进行校验
		String clientId = tokenRequest.getClientId();
		ClientDetails client = clientDetailsService.loadClientByClientId(clientId);
		validateGrantType(grantType, client);

		if (logger.isDebugEnabled()) {
			logger.debug("Getting access token for: " + clientId);
		}
        // 创建token
		return getAccessToken(client, tokenRequest);

	}

	protected OAuth2AccessToken getAccessToken(ClientDetails client, TokenRequest tokenRequest) {
		return tokenServices.createAccessToken(getOAuth2Authentication(client, tokenRequest));
	}
```
```java
package org.springframework.security.oauth2.provider.password;
/**
 * @author Dave Syer
 * 
 */
public class ResourceOwnerPasswordTokenGranter extends AbstractTokenGranter {

	private static final String GRANT_TYPE = "password";

	private final AuthenticationManager authenticationManager;

	public ResourceOwnerPasswordTokenGranter(AuthenticationManager authenticationManager,
			AuthorizationServerTokenServices tokenServices, ClientDetailsService clientDetailsService, OAuth2RequestFactory requestFactory) {
		this(authenticationManager, tokenServices, clientDetailsService, requestFactory, GRANT_TYPE);
	}

	protected ResourceOwnerPasswordTokenGranter(AuthenticationManager authenticationManager, AuthorizationServerTokenServices tokenServices,
			ClientDetailsService clientDetailsService, OAuth2RequestFactory requestFactory, String grantType) {
		super(tokenServices, clientDetailsService, requestFactory, grantType);
		this.authenticationManager = authenticationManager;
	}

	@Override
	protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {

		Map<String, String> parameters = new LinkedHashMap<String, String>(tokenRequest.getRequestParameters());
		String username = parameters.get("username");
		String password = parameters.get("password");
		// Protect from downstream leaks of password
		parameters.remove("password");
        //获取用户名及密码创建UsernamePasswordAuthenticationToken委派到AuthenticationManager进行校验
		Authentication userAuth = new UsernamePasswordAuthenticationToken(username, password);
		((AbstractAuthenticationToken) userAuth).setDetails(parameters);
		try {
            // authenticationManager为
            // org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter.AuthenticationManagerDelegator
            // 委派到org.springframework.security.authentication.ProviderManager
            // 最后委派到org.springframework.security.authentication.dao.DaoAuthenticationProvider
			userAuth = authenticationManager.authenticate(userAuth);
		}
		catch (AccountStatusException ase) {
			//covers expired, locked, disabled cases (mentioned in section 5.2, draft 31)
			throw new InvalidGrantException(ase.getMessage());
		}
		catch (BadCredentialsException e) {
			// If the username/password are wrong the spec says we should send 400/invalid grant
			throw new InvalidGrantException(e.getMessage());
		}
		if (userAuth == null || !userAuth.isAuthenticated()) {
			throw new InvalidGrantException("Could not authenticate user: " + username);
		}
		
		OAuth2Request storedOAuth2Request = getRequestFactory().createOAuth2Request(client, tokenRequest);		
		return new OAuth2Authentication(storedOAuth2Request, userAuth);
	}
}
```
# org.springframework.security.authentication.dao.DaoAuthenticationProvider源码
```java
package org.springframework.security.authentication.dao;

public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
	// ~ Static fields/initializers
	// =====================================================================================

	/**
	 * The plaintext password used to perform
	 * PasswordEncoder#matches(CharSequence, String)}  on when the user is
	 * not found to avoid SEC-2056.
	 */
	private static final String USER_NOT_FOUND_PASSWORD = "userNotFoundPassword";

	// ~ Instance fields
	// ================================================================================================

	private PasswordEncoder passwordEncoder;

	/**
	 * The password used to perform
	 * {@link PasswordEncoder#matches(CharSequence, String)} on when the user is
	 * not found to avoid SEC-2056. This is necessary, because some
	 * {@link PasswordEncoder} implementations will short circuit if the password is not
	 * in a valid format.
	 */
	private volatile String userNotFoundEncodedPassword;

	private UserDetailsService userDetailsService;

	private UserDetailsPasswordService userDetailsPasswordService;

	public DaoAuthenticationProvider() {
		setPasswordEncoder(PasswordEncoderFactories.createDelegatingPasswordEncoder());
	}

	// ~ Methods
	// ========================================================================================================

	@SuppressWarnings("deprecation")
	protected void additionalAuthenticationChecks(UserDetails userDetails,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
		if (authentication.getCredentials() == null) {
			logger.debug("Authentication failed: no credentials provided");

			throw new BadCredentialsException(messages.getMessage(
					"AbstractUserDetailsAuthenticationProvider.badCredentials",
					"Bad credentials"));
		}

		String presentedPassword = authentication.getCredentials().toString();

		if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			logger.debug("Authentication failed: password does not match stored value");

			throw new BadCredentialsException(messages.getMessage(
					"AbstractUserDetailsAuthenticationProvider.badCredentials",
					"Bad credentials"));
		}
	}

	protected void doAfterPropertiesSet() throws Exception {
		Assert.notNull(this.userDetailsService, "A UserDetailsService must be set");
	}

	protected final UserDetails retrieveUser(String username,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
		prepareTimingAttackProtection();
		try {
            // 根据org.springframework.security.core.userdetails.UserDetailsService接口实现获取UserDetails
			UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
			if (loadedUser == null) {
				throw new InternalAuthenticationServiceException(
						"UserDetailsService returned null, which is an interface contract violation");
			}
			return loadedUser;
		}
		catch (UsernameNotFoundException ex) {
			mitigateAgainstTimingAttack(authentication);
			throw ex;
		}
		catch (InternalAuthenticationServiceException ex) {
			throw ex;
		}
		catch (Exception ex) {
			throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
		}
	}

	@Override
	protected Authentication createSuccessAuthentication(Object principal,
			Authentication authentication, UserDetails user) {
		boolean upgradeEncoding = this.userDetailsPasswordService != null
				&& this.passwordEncoder.upgradeEncoding(user.getPassword());
		if (upgradeEncoding) {
			String presentedPassword = authentication.getCredentials().toString();
			String newPassword = this.passwordEncoder.encode(presentedPassword);
			user = this.userDetailsPasswordService.updatePassword(user, newPassword);
		}
		return super.createSuccessAuthentication(principal, authentication, user);
	}
}

```
```text
调用 org.springframework.security.authentication.dao.DaoAuthenticationProvider父类的authenticate方法来进行验证，
根据输入的username来获取UserDetails,获取到UserDetails之后，校验账号没有异常以及密码没有异常之后创建Authentication，并存入上下文
```
```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
				() -> messages.getMessage(
						"AbstractUserDetailsAuthenticationProvider.onlySupports",
						"Only UsernamePasswordAuthenticationToken is supported"));

		// Determine username
		String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
				: authentication.getName();

		boolean cacheWasUsed = true;
        // 查询缓存获取UserDetails默认为UserCache userCache = new NullUserCache();
		UserDetails user = this.userCache.getUserFromCache(username);

		if (user == null) {
			cacheWasUsed = false;

			try {
                // 抽象方法具体实现在子类DaoAuthenticationProvider中
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication);
			}
			catch (UsernameNotFoundException notFound) {
				logger.debug("User '" + username + "' not found");

				if (hideUserNotFoundExceptions) {
					throw new BadCredentialsException(messages.getMessage(
							"AbstractUserDetailsAuthenticationProvider.badCredentials",
							"Bad credentials"));
				}
				else {
					throw notFound;
				}
			}

			Assert.notNull(user,
					"retrieveUser returned null - a violation of the interface contract");
		}
		try {
            // 前置校验。根据UserDetails信息，校验账号是否可用（被锁定的账号不可用），是否已经过期
			preAuthenticationChecks.check(user);
            //校验账号密码是否正确
			additionalAuthenticationChecks(user,
					(UsernamePasswordAuthenticationToken) authentication);
		}
		catch (AuthenticationException exception) {
			if (cacheWasUsed) {
				// There was a problem, so try again after checking
				// we're using latest data (i.e. not from the cache)
				cacheWasUsed = false;
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication);
				preAuthenticationChecks.check(user);
				additionalAuthenticationChecks(user,
						(UsernamePasswordAuthenticationToken) authentication);
			}
			else {
				throw exception;
			}
		}
        // 后置校验。根据UserDetails信息，校验账号是否可用（被锁定的账号不可用），是否已经过期
		postAuthenticationChecks.check(user);

		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}

		Object principalToReturn = user;

		if (forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}
        //账号密码正确，并且账号没有异常则创建Authentication，实际对象为UsernamePasswordAuthenticationToken
		return createSuccessAuthentication(principalToReturn, authentication, user);
	}
	
	protected Authentication createSuccessAuthentication(Object principal,
			Authentication authentication, UserDetails user) {
		// Ensure we return the original credentials the user supplied,
		// so subsequent attempts are successful even with encoded passwords.
		// Also ensure we return the original getDetails(), so that future
		// authentication events after cache expiry contain the details
		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
				principal, authentication.getCredentials(),
				authoritiesMapper.mapAuthorities(user.getAuthorities()));
		result.setDetails(authentication.getDetails());

		return result;
	}
```
```text
创建Authentication之后回到ResourceOwnerPasswordTokenGranter的getOAuth2Authentication方法，创建OAuth2Authentication并返回
（return new OAuth2Authentication(storedOAuth2Request, userAuth)）,然后回到超类之中如下，使用tokenServices创建OAuth2AccessToken，
最后回到ResourceOwnerPasswordTokenGranter的postAccessToken方法
创建ResponseEntity并相应请求。
tokenServices
```
为[org.springframework.security.oauth2.provider.token.DefaultTokenServices](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-DefaultTokenServices.md)
```java
public abstract class AbstractTokenGranter implements TokenGranter {

	protected OAuth2AccessToken getAccessToken(ClientDetails client, TokenRequest tokenRequest) {
		return tokenServices.createAccessToken(getOAuth2Authentication(client, tokenRequest));
	}
}
```