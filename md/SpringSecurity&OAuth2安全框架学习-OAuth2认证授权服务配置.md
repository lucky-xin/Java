```java
@Order(-10)
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
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.withClientDetails(authClientDetailsService);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(redisStore())
                .reuseRefreshTokens(false)
                .accessTokenConverter(oauthTokenEnhancer)
                .userDetailsService(userDetailsService)
                .exceptionTranslator(oauth2ExceptionTranslator())
                .allowedTokenEndpointRequestMethods(HttpMethod.GET, HttpMethod.POST);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer oauthServer) throws Exception {
        //允许表单认证
        oauthServer.allowFormAuthenticationForClients()
                // /oauth/check_token 需要权限认证
                .checkTokenAccess("isAuthenticated()");
    }

    @Bean
    public TokenStore redisStore() {
        RedisTokenStore redisStore = new RedisTokenStore(redisConnectionFactory);
        redisStore.setPrefix(Constants.REDIS_STORE_PREFIX);
        return redisStore;
    }
   
}


```