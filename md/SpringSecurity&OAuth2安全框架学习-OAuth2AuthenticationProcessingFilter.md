```text
OAuth2AuthenticationProcessingFilter 用于拦截访问SpringBoot & SpringCloud 的请求，使用TokenExtractor接口的实现类默认为[BearerTokenExtractor]()解析以Bearer开头（忽略大小写）的token
解析请求头Authorization获取认证的token值
```

```java
package org.springframework.security.oauth2.provider.authentication;

/**
 * A pre-authentication filter for OAuth2 protected resources. Extracts an OAuth2 token from the incoming request and
 * uses it to populate the Spring Security context with an {@link OAuth2Authentication} (if used in conjunction with an
 * {@link OAuth2AuthenticationManager}).
 * 
 * @author Dave Syer
 * 
 */
public class OAuth2AuthenticationProcessingFilter implements Filter, InitializingBean {

	private final static Log logger = LogFactory.getLog(OAuth2AuthenticationProcessingFilter.class);

	private AuthenticationEntryPoint authenticationEntryPoint = new OAuth2AuthenticationEntryPoint();

	private AuthenticationManager authenticationManager;

	private AuthenticationDetailsSource<HttpServletRequest, ?> authenticationDetailsSource = new OAuth2AuthenticationDetailsSource();

	private TokenExtractor tokenExtractor = new BearerTokenExtractor();

	private AuthenticationEventPublisher eventPublisher = new NullEventPublisher();

	private boolean stateless = true;
	
    //...

	public void afterPropertiesSet() {
		Assert.state(authenticationManager != null, "AuthenticationManager is required");
	}

	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException,
			ServletException {

		final boolean debug = logger.isDebugEnabled();
		final HttpServletRequest request = (HttpServletRequest) req;
		final HttpServletResponse response = (HttpServletResponse) res;

		try {

			Authentication authentication = tokenExtractor.extract(request);
			
			if (authentication == null) {
				if (stateless && isAuthenticated()) {
					if (debug) {
						logger.debug("Clearing security context.");
					}
					SecurityContextHolder.clearContext();
				}
				if (debug) {
					logger.debug("No token in request, will continue chain.");
				}
			}
			else {
				request.setAttribute(OAuth2AuthenticationDetails.ACCESS_TOKEN_VALUE, authentication.getPrincipal());
				if (authentication instanceof AbstractAuthenticationToken) {
					AbstractAuthenticationToken needsDetails = (AbstractAuthenticationToken) authentication;
					needsDetails.setDetails(authenticationDetailsSource.buildDetails(request));
				}
				Authentication authResult = authenticationManager.authenticate(authentication);

				if (debug) {
					logger.debug("Authentication success: " + authResult);
				}

				eventPublisher.publishAuthenticationSuccess(authResult);
				SecurityContextHolder.getContext().setAuthentication(authResult);

			}
		}
		catch (OAuth2Exception failed) {
			SecurityContextHolder.clearContext();

			if (debug) {
				logger.debug("Authentication request failed: " + failed);
			}
			eventPublisher.publishAuthenticationFailure(new BadCredentialsException(failed.getMessage(), failed),
					new PreAuthenticatedAuthenticationToken("access-token", "N/A"));

			authenticationEntryPoint.commence(request, response,
					new InsufficientAuthenticationException(failed.getMessage(), failed));

			return;
		}

		chain.doFilter(request, response);
	}

	private boolean isAuthenticated() {
		Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
		if (authentication == null || authentication instanceof AnonymousAuthenticationToken) {
			return false;
		}
		return true;
	}

	public void init(FilterConfig filterConfig) throws ServletException {
	}

	public void destroy() {
	}

	private static final class NullEventPublisher implements AuthenticationEventPublisher {
		public void publishAuthenticationFailure(AuthenticationException exception, Authentication authentication) {
		}

		public void publishAuthenticationSuccess(Authentication authentication) {
		}
	}

}

```