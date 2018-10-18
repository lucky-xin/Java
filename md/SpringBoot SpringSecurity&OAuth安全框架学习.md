# 踩过的坑
## ajax发送请求时，登陆成功之后页面并不能转发。理论上应该是访问受限资源登录成功之后跳转至登录前页面。但是ajax并不能控制浏览器跳转
## 登录用户名参数名称必须为username，密码参数名称必须为password,这是在org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
之中指定好了的
## Spring Security框架设计并没有IoC,MVC好,譬如MVC你可以自己指定DispatcherServlet使用HandlerMapping，HandlerAdapter等等等。。。指定优先级，被某一个HandlerMapping拦截
其他HandlerMapping不在执行，在Spring Security之中做不到，虽然必须要执行下一个拦截器，但是可以替换拦截器呀。对，就是不能替换，所有的拦截器都要执行一遍，就算你自定义了拦截器addFilterBefore于UsernamePasswordAuthenticationFilter
下一个就执行UsernamePasswordAuthenticationFilter了！！！

# 源码解析 在org.springframework.security.web.access.intercept.FilterSecurityInterceptor中的doFilter拦截到请求
```java
public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		FilterInvocation fi = new FilterInvocation(request, response, chain);
		invoke(fi);
	}
```

## 进入invoke方法之中
```java
public void invoke(FilterInvocation fi) throws IOException, ServletException {
		if ((fi.getRequest() != null)
				&& (fi.getRequest().getAttribute(FILTER_APPLIED) != null)
				&& observeOncePerRequest) {
			// filter already applied to this request and user wants us to observe
			// once-per-request handling, so don't re-do security checking
			fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
		}
		else {
			// first time this request being called, so perform security checking
			if (fi.getRequest() != null && observeOncePerRequest) {
				fi.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
			}
            // 权限校验入口
			InterceptorStatusToken token = super.beforeInvocation(fi);

			try {
				fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
			}
			finally {
				super.finallyInvocation(token);
			}

			super.afterInvocation(token, null);
		}
	}
```
## 在org.springframework.security.web.access.intercept.FilterSecurityInterceptor
超类org.springframework.security.access.intercept.AbstractSecurityInterceptor之中的beforeInvocation方法根据设置的
SecurityMetadataSource类根据url查询此用户是否有访问该url权限。查询缓存是否有已经登录认证的信息。然后使用设置的AccessDecisionManager比对权限。
```java
protected InterceptorStatusToken beforeInvocation(Object object) {
		Assert.notNull(object, "Object was null");
		final boolean debug = logger.isDebugEnabled();

		if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {
			throw new IllegalArgumentException(
					"Security invocation attempted for object "
							+ object.getClass().getName()
							+ " but AbstractSecurityInterceptor only configured to support secure objects of type: "
							+ getSecureObjectClass());
		}
        //根据url查询权限信息
		Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
				.getAttributes(object);

		if (attributes == null || attributes.isEmpty()) {
			if (rejectPublicInvocations) {
				throw new IllegalArgumentException(
						"Secure object invocation "
								+ object
								+ " was denied as public invocations are not allowed via this interceptor. "
								+ "This indicates a configuration error because the "
								+ "rejectPublicInvocations property is set to 'true'");
			}

			if (debug) {
				logger.debug("Public object - authentication not attempted");
			}

			publishEvent(new PublicInvocationEvent(object));

			return null; // no further work post-invocation
		}

		if (debug) {
			logger.debug("Secure object: " + object + "; Attributes: " + attributes);
		}

		if (SecurityContextHolder.getContext().getAuthentication() == null) {
			credentialsNotFound(messages.getMessage(
					"AbstractSecurityInterceptor.authenticationNotFound",
					"An Authentication object was not found in the SecurityContext"),
					object, attributes);
		}
        // 获取缓存之中的认证权限信息，如何没有就返回anyone权限
		Authentication authenticated = authenticateIfRequired();

		// Attempt authorization
		try {
		    //决定是否有访问权限,如果认证权限失败则抛出AccessDeniedException异常，否则授权访问
			this.accessDecisionManager.decide(authenticated, object, attributes);
		}
		catch (AccessDeniedException accessDeniedException) {
			publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,
					accessDeniedException));

			throw accessDeniedException;
		}

		if (debug) {
			logger.debug("Authorization successful");
		}

		if (publishAuthorizationSuccess) {
			publishEvent(new AuthorizedEvent(object, attributes, authenticated));
		}

		// Attempt to run as a different user
		Authentication runAs = this.runAsManager.buildRunAs(authenticated, object,
				attributes);

		if (runAs == null) {
			if (debug) {
				logger.debug("RunAsManager did not change Authentication object");
			}

			// no further work post-invocation
			return new InterceptorStatusToken(SecurityContextHolder.getContext(), false,
					attributes, object);
		}
		else {
			if (debug) {
				logger.debug("Switching to RunAs Authentication: " + runAs);
			}

			SecurityContext origCtx = SecurityContextHolder.getContext();
			SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
			SecurityContextHolder.getContext().setAuthentication(runAs);
			// need to revert to token.Authenticated post-invocation
			return new InterceptorStatusToken(origCtx, true, attributes, object);
		}
	}
```
## 如果没有权限则抛出异常,抛出的异常会被org.springframework.security.web.access.ExceptionTranslationFilter过滤器拦截并处理
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		try {
			chain.doFilter(request, response);

			logger.debug("Chain processed normally");
		}
		catch (IOException ex) {
			throw ex;
		}
		catch (Exception ex) {
			// Try to extract a SpringSecurityException from the stacktrace
			Throwable[] causeChain = throwableAnalyzer.determineCauseChain(ex);
			RuntimeException ase = (AuthenticationException) throwableAnalyzer
					.getFirstThrowableOfType(AuthenticationException.class, causeChain);

			if (ase == null) {
				ase = (AccessDeniedException) throwableAnalyzer.getFirstThrowableOfType(
						AccessDeniedException.class, causeChain);
			}

			if (ase != null) {
				if (response.isCommitted()) {
					throw new ServletException("Unable to handle the Spring Security Exception because the response is already committed.", ex);
				}
				// 处理权限不足异常
				handleSpringSecurityException(request, response, chain, ase);
			}
			else {
				// Rethrow ServletExceptions and RuntimeExceptions as-is
				if (ex instanceof ServletException) {
					throw (ServletException) ex;
				}
				else if (ex instanceof RuntimeException) {
					throw (RuntimeException) ex;
				}

				// Wrap other Exceptions. This shouldn't actually happen
				// as we've already covered all the possibilities for doFilter
				throw new RuntimeException(ex);
			}
		}
	}
```
## 进入handleSpringSecurityException方法保存本次请求信息，使用配置的AuthenticationEntryPoint来处理异常请求，
比如LoginUrlAuthenticationEntryPoint 可以转发到登录页面。
```java
private void handleSpringSecurityException(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, RuntimeException exception)
			throws IOException, ServletException {
		if (exception instanceof AuthenticationException) {
			logger.debug(
					"Authentication exception occurred; redirecting to authentication entry point",
					exception);

			sendStartAuthentication(request, response, chain,
					(AuthenticationException) exception);
		}
		else if (exception instanceof AccessDeniedException) {
			Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
			if (authenticationTrustResolver.isAnonymous(authentication) || authenticationTrustResolver.isRememberMe(authentication)) {
				logger.debug(
						"Access is denied (user is " + (authenticationTrustResolver.isAnonymous(authentication) ? "anonymous" : "not fully authenticated") + "); redirecting to authentication entry point",
						exception);

				sendStartAuthentication(
						request,
						response,
						chain,
						new InsufficientAuthenticationException(
							messages.getMessage(
								"ExceptionTranslationFilter.insufficientAuthentication",
								"Full authentication is required to access this resource")));
			}
			else {
				logger.debug(
						"Access is denied (user is not anonymous); delegating to AccessDeniedHandler",
						exception);

				accessDeniedHandler.handle(request, response,(AccessDeniedException) exception);
			}
		}
	}
	// sendStartAuthentication进入sendStartAuthentication方法使用配置的
	// AuthenticationEntryPoint来处理异常。如使用LoginUrlAuthenticationEntryPoint来跳转到登录页面
	protected void sendStartAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain,
			AuthenticationException reason) throws ServletException, IOException {
		// SEC-112: Clear the SecurityContextHolder's Authentication, as the
		// existing Authentication is no longer considered valid
		SecurityContextHolder.getContext().setAuthentication(null);
		
		requestCache.saveRequest(request, response);
		logger.debug("Calling Authentication entry point.");
		authenticationEntryPoint.commence(request, response, reason);
	}
```
# 跳转到登录页面进行登录认证流程。登录认证默认使用org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
配置了url匹配规则，如构造方法
```java
    // 设置匹配规则为/login uri，匹配请求方法为post请求
	public UsernamePasswordAuthenticationFilter() {
		super(new AntPathRequestMatcher("/login", "POST"));
	}
```
## 在UsernamePasswordAuthenticationFilter的超类org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter
会根据匹配规则来判断是否执行拦截方法
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {

		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
        // 判断是否匹配此过滤器，如果不匹配调用过滤器链下一个过滤器
		if (!requiresAuthentication(request, response)) {
			chain.doFilter(request, response);

			return;
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Request is to process authentication");
		}

		Authentication authResult;

		try {
		    // 尝试认证,为抽象方法，具体实现在子类UsernamePasswordAuthenticationFilter之中
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				// return immediately as subclass has indicated that it hasn't completed
				// authentication
				return;
			}
			sessionStrategy.onAuthentication(authResult, request, response);
		}
		catch (InternalAuthenticationServiceException failed) {
			logger.error(
					"An internal error occurred while trying to authenticate the user.",
					failed);
			unsuccessfulAuthentication(request, response, failed);

			return;
		}
		catch (AuthenticationException failed) {
			// Authentication failed
			unsuccessfulAuthentication(request, response, failed);

			return;
		}

		// Authentication success
		if (continueChainBeforeSuccessfulAuthentication) {
			chain.doFilter(request, response);
		}
        // 认证成功，调用认证成功处理器进行后续操作
		successfulAuthentication(request, response, chain, authResult);
	}
```
## 调用attemptAuthentication进行认证,在UsernamePasswordAuthenticationFilter类之中
```java
public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}
        // 获取请求之中的用户名密码
		String username = obtainUsername(request);
		String password = obtainPassword(request);

		if (username == null) {
			username = "";
		}

		if (password == null) {
			password = "";
		}

		username = username.trim();
		// 封装用户名，密码信息
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);

		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);
		// 调用配置的认证管理器来进行认证
		return this.getAuthenticationManager().authenticate(authRequest);
	}
```
## 获取用户名密码这里有坑用户名参数名称必须为username,密码参数必须为password
```java
    public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
	public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";

	private String usernameParameter = SPRING_SECURITY_FORM_USERNAME_KEY;
	private String passwordParameter = SPRING_SECURITY_FORM_PASSWORD_KEY;
	    // 获取密码
	    protected String obtainPassword(HttpServletRequest request) {
    		return request.getParameter(passwordParameter);
    	}
    
    	// 获取用户名称
    	protected String obtainUsername(HttpServletRequest request) {
    		return request.getParameter(usernameParameter);
    	}
	
```
## 调用AuthenticationManager来进行认证。可以自己配置.默认使用org.springframework.security.authentication.ProviderManager来进行认证。
```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		Authentication result = null;
		boolean debug = logger.isDebugEnabled();
        // 获取到所有的认证提供者来进行认证，如果有其中一个认证通过则表示本次认证已经成功，跳出for循环
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}

			if (debug) {
				logger.debug("Authentication attempt using "
						+ provider.getClass().getName());
			}

			try {
				result = provider.authenticate(authentication);

				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException e) {
				prepareException(e, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw e;
			}
			catch (InternalAuthenticationServiceException e) {
				prepareException(e, authentication);
				throw e;
			}
			catch (AuthenticationException e) {
				lastException = e;
			}
		}

		if (result == null && parent != null) {
			// Allow the parent to try.
			try {
				result = parent.authenticate(authentication);
			}
			catch (ProviderNotFoundException e) {
				// ignore as we will throw below if no other exception occurred prior to
				// calling parent and the parent
				// may throw ProviderNotFound even though a provider in the child already
				// handled the request
			}
			catch (AuthenticationException e) {
				lastException = e;
			}
		}

		if (result != null) {
			if (eraseCredentialsAfterAuthentication
					&& (result instanceof CredentialsContainer)) {
				// Authentication is complete. Remove credentials and other secret data
				// from authentication
				((CredentialsContainer) result).eraseCredentials();
			}

			eventPublisher.publishAuthenticationSuccess(result);
			return result;
		}

		// Parent was null, or didn't authenticate (or throw an exception).

		if (lastException == null) {
			lastException = new ProviderNotFoundException(messages.getMessage(
					"ProviderManager.providerNotFound",
					new Object[] { toTest.getName() },
					"No AuthenticationProvider found for {0}"));
		}

		prepareException(lastException, authentication);

		throw lastException;
	}
```
## 认证成功之后进入返回认证信息Authentication，回到AbstractAuthenticationProcessingFilter之中调用successfulAuthentication方法完成后续操作
```java
protected void successfulAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, Authentication authResult)
			throws IOException, ServletException {

		if (logger.isDebugEnabled()) {
			logger.debug("Authentication success. Updating SecurityContextHolder to contain: "
					+ authResult);
		}
        //把认证信息存入缓存之中，本次请求结束之后会清除
		SecurityContextHolder.getContext().setAuthentication(authResult);

		rememberMeServices.loginSuccess(request, response, authResult);

		// Fire event
		if (this.eventPublisher != null) {
			eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
					authResult, this.getClass()));
		}
        //调用认证成功处理器来处理认证成功操作，successHandler可以自己配置
		successHandler.onAuthenticationSuccess(request, response, authResult);
	}
```
## 默认认证成功处理为。跳转到被拦截之前的页面
```java
@Override
	public void onAuthenticationSuccess(HttpServletRequest request,
			HttpServletResponse response, Authentication authentication)
			throws ServletException, IOException {
        //获取到被拦截url信息。用户访问受限资源而且没有登录会抛出异常，保存现场。认证成功之后会获取现场信息
		SavedRequest savedRequest = requestCache.getRequest(request, response);

		if (savedRequest == null) {
			super.onAuthenticationSuccess(request, response, authentication);

			return;
		}
		String targetUrlParameter = getTargetUrlParameter();
		if (isAlwaysUseDefaultTargetUrl()
				|| (targetUrlParameter != null && StringUtils.hasText(request
						.getParameter(targetUrlParameter)))) {
			requestCache.removeRequest(request, response);
			super.onAuthenticationSuccess(request, response, authentication);

			return;
		}

		clearAuthenticationAttributes(request);

		// Use the DefaultSavedRequest URL
		// 获取被拦截的url,进行页面跳转
		String targetUrl = savedRequest.getRedirectUrl();
		logger.debug("Redirecting to DefaultSavedRequest Url: " + targetUrl);
		getRedirectStrategy().sendRedirect(request, response, targetUrl);
	}
```