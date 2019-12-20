```java
@Configuration
@AllArgsConstructor
@EnableAuthorizationServer
public class AuthorizationServerConfiguration extends AuthorizationServerConfigurerAdapter {

    private AuthenticationManager authenticationManager;

    private RedisConnectionFactory redisConnectionFactory;

    private OAuth2AccessTokenConverter oauthTokenEnhancer;

    private UserDetailsService userDetailsService;

    private JdbcClientDetailsService authClientDetailsService;

    @Override
    @SneakyThrows
    public void configure(ClientDetailsServiceConfigurer clients) {
        clients.withClientDetails(clientDetailsService);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer oauthServer) {
        oauthServer
                .allowFormAuthenticationForClients()
                // /oauth/check_token 需验证才能访问
                .checkTokenAccess("isAuthenticated()");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints
                .allowedTokenEndpointRequestMethods(HttpMethod.GET, HttpMethod.POST)
                .tokenStore(tokenStore())
                .authorizationCodeServices(authorizationCodeServices)
                .tokenEnhancer(tokenEnhancer())
                .userDetailsService(authUserDetailsService)
                .authenticationManager(authenticationManagerBean)
                .reuseRefreshTokens(false)
                .pathMapping("/oauth/confirm_access", "/token/confirm_access")
                .exceptionTranslator(new AuthWebResponseExceptionTranslator());

    }

    public AuthorizationServerTokenServices authorizationServerTokenServices() {
        // 配置TokenServices参数
        DefaultTokenServices tokenServices = new DefaultTokenServices();
        tokenServices.setTokenStore(tokenStore());
        tokenServices.setSupportRefreshToken(false);
        tokenServices.setClientDetailsService(clientDetailsService);
        tokenServices.setTokenEnhancer(tokenEnhancer());
        return tokenServices;
    }

    @Bean
    public TokenStore tokenStore() {
        RedisTokenStore tokenStore = new RedisTokenStore(redisConnectionFactory);
        tokenStore.setPrefix(SecurityConstants.DATA_INSIGHTS_PREFIX + SecurityConstants.OAUTH_PREFIX);
        tokenStore.setAuthenticationKeyGenerator(new DefaultAuthenticationKeyGenerator() {
            @Override
            public String extractKey(OAuth2Authentication authentication) {
                return super.extractKey(authentication) + ":" + TenantContextHolder.getTenantId();
            }
        });
        return tokenStore;
    }

    /**
     * token增强，客户端模式不增强。
     *
     * @return TokenEnhancer
     */
    @Bean
    public TokenEnhancer tokenEnhancer() {
        return (accessToken, authentication) -> {

            if (SecurityConstants.CLIENT_CREDENTIALS.equals(authentication.getOAuth2Request().getGrantType())
                    || !(authentication.getUserAuthentication().getPrincipal() instanceof AuthUser)) {
                return accessToken;
            }
            AuthUser authUser = (AuthUser) authentication.getUserAuthentication().getPrincipal();
            final Map<String, Object> additionalInfo = AuthUtils.getAdditionalInfo(authUser);
            if (Objects.nonNull(additionalInfo)) {
                ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(additionalInfo);
            }
            return accessToken;
        };
    }
}

```

```java
/**
 * @author luchaoxin
 * @date 2018/6/22
 * 认证相关配置
 */
@Primary
@Order(90)
@Configuration
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
	@Autowired
	private ObjectMapper objectMapper;
	@Autowired
	private ClientDetailsService clientDetailsService;
	@Autowired
	private AuthUserDetailsService userDetailsService;
	@Autowired
	private RemoteUserService remoteUserService;
	@Autowired
	private RemoteEmailService remoteEmailService;
	@Autowired
	private RemoteSmsService remoteSmsService;
	@Autowired
	private ResourceAuthExceptionEntryPoint resourceAuthExceptionEntryPoint;
	@Lazy
	@Autowired
	private AuthorizationServerTokenServices defaultAuthorizationServerTokenServices;

	@Override
	@SneakyThrows
	protected void configure(HttpSecurity http) {
		http
				.formLogin()
				.loginPage("/token/login")
				.loginProcessingUrl("/token/form")
				.and()
				.authorizeRequests()
				.antMatchers(
						"/token/**",
						"/error/**",
						"/actuator/**",
						"/auth/dynamic-code/**",
						"/mobile/**").permitAll()
				.and().exceptionHandling()

				.and().authorizeRequests().anyRequest().authenticated()

				.and().csrf().disable()
				.apply(socialContactSecurityConfigurer());
		http.authenticationProvider(authenticationProvider());
		http.addFilterBefore(credentialPerRequestFilter(), UsernamePasswordAuthenticationFilter.class);
	}

	/**
	 * 不拦截静态资源
	 *
	 * @param web
	 */
	@Override
	public void configure(WebSecurity web) {
		web.ignoring().antMatchers("/css/**");
	}

	@Bean
	@Override
	@SneakyThrows
	public AuthenticationManager authenticationManagerBean() {
		return super.authenticationManagerBean();
	}

	public ClientCredentialPerRequestFilter credentialPerRequestFilter() {
		return new ClientCredentialPerRequestFilter(resourceAuthExceptionEntryPoint,
				clientDetailsService,
				passwordEncoder());
	}

	@Bean
	public AuthenticationSuccessHandler authenticationSuccessHandler() {
		return SocialLoginSuccessHandler.builder()
				.objectMapper(objectMapper)
				.clientDetailsService(clientDetailsService)
				.passwordEncoder(passwordEncoder())
				.defaultAuthorizationServerTokenServices(defaultAuthorizationServerTokenServices).build();
	}

	@Bean
	public SocialContactSecurityConfigurer socialContactSecurityConfigurer() {
		SocialContactSecurityConfigurer socialContactSecurityConfigurer = new SocialContactSecurityConfigurer();
		socialContactSecurityConfigurer.setAuthenticationSuccessHandler(authenticationSuccessHandler());
		socialContactSecurityConfigurer.setObjectMapper(objectMapper);
		socialContactSecurityConfigurer.setUserDetailsService(userDetailsService);
		socialContactSecurityConfigurer.setRemoteEmailService(remoteEmailService);
		socialContactSecurityConfigurer.setRemoteSmsService(remoteSmsService);
		socialContactSecurityConfigurer.setRemoteUserService(remoteUserService);
		socialContactSecurityConfigurer.setPasswordEncoder(passwordEncoder());
		return socialContactSecurityConfigurer;
	}

	/**
	 * https://spring.io/blog/2017/11/01/spring-security-5-0-0-rc1-released#password-storage-updated
	 * Encoded password does not look like BCrypt
	 *
	 * @return PasswordEncoder
	 */
	@Bean
	public PasswordEncoder passwordEncoder() {
		return PasswordEncoderFactories.createDelegatingPasswordEncoder();
	}

	@Bean
	public DaoAuthenticationProvider authenticationProvider() {
		DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
		provider.setHideUserNotFoundExceptions(false);
		provider.setUserDetailsService(userDetailsService);
		provider.setPasswordEncoder(passwordEncoder());
		return provider;
	}
}
```