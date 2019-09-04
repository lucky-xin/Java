# SavedRequestAwareAuthenticationSuccessHandler用于恢复[ExceptionTranslationFilter](https://github.com/lucky-xin/Learning/blob/gh-pages/md/SpringSecurity%26OAuth2%E5%AE%89%E5%85%A8%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0-ExceptionTanslationFilter.md)
# 拦截异常AccessDeniedException,AuthenticationException，使用SavedRequest保存的request,认证成功之后从SavedRequest获取出现的异常的uri进行重定向到该uri
```java

package org.springframework.security.web.authentication;

/**
 * An authentication success strategy which can make use of the
 * {@link org.springframework.security.web.savedrequest.DefaultSavedRequest} which may have been stored in the session by the
 * {@link ExceptionTranslationFilter}. When such a request is intercepted and requires
 * authentication, the request data is stored to record the original destination before
 * the authentication process commenced, and to allow the request to be reconstructed when
 * a redirect to the same URL occurs. This class is responsible for performing the
 * redirect to the original URL if appropriate.
 * <p>
 * Following a successful authentication, it decides on the redirect destination, based on
 * the following scenarios:
 * <ul>
 * <li>
 * If the {@code alwaysUseDefaultTargetUrl} property is set to true, the
 * {@code defaultTargetUrl} will be used for the destination. Any
 * {@code DefaultSavedRequest} stored in the session will be removed.</li>
 * <li>
 * If the {@code targetUrlParameter} has been set on the request, the value will be used
 * as the destination. Any {@code DefaultSavedRequest} will again be removed.</li>
 * <li>
 * If a {@link org.springframework.security.web.savedrequest.SavedRequest} is found in the {@code RequestCache} (as set by the
 * {@link ExceptionTranslationFilter} to record the original destination before the
 * authentication process commenced), a redirect will be performed to the Url of that
 * original destination. The {@code SavedRequest} object will remain cached and be picked
 * up when the redirected request is received (See
 * <a href="{@docRoot}/org/springframework/security/web/savedrequest/SavedRequestAwareWrapper.html">SavedRequestAwareWrapper</a>).
 * </li>
 * <li>
 * If no {@link org.springframework.security.web.savedrequest.SavedRequest} is found, it will delegate to the base class.</li>
 * </ul>
 *
 * @author Luke Taylor
 * @since 3.0
 */
public class SavedRequestAwareAuthenticationSuccessHandler extends
		SimpleUrlAuthenticationSuccessHandler {
	protected final Log logger = LogFactory.getLog(this.getClass());

	private RequestCache requestCache = new HttpSessionRequestCache();

	@Override
	public void onAuthenticationSuccess(HttpServletRequest request,
			HttpServletResponse response, Authentication authentication)
			throws ServletException, IOException {
        // 获取缓存的Request 
		SavedRequest savedRequest = requestCache.getRequest(request, response);
        //如果获取不到Request则使用父类onAuthenticationSuccess进行处理
		if (savedRequest == null) {
			super.onAuthenticationSuccess(request, response, authentication);

			return;
		}
        // alwaysUseDefaultTargetUrl默认为false,如果为true,则使用配置targetUrlParameter作为重定向的uri,
        // 并清除缓存的request
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
		String targetUrl = savedRequest.getRedirectUrl();
		logger.debug("Redirecting to DefaultSavedRequest Url: " + targetUrl);
		getRedirectStrategy().sendRedirect(request, response, targetUrl);
	}

	public void setRequestCache(RequestCache requestCache) {
		this.requestCache = requestCache;
	}
}

```