```java
@Configuration
@EnableWebSecurity
@EnableResourceServer
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
public class OAuth2SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Autowired
    private UserDetailsService userDetailsService;
    
    @Bean
    @Override
    @SneakyThrows
    public AuthenticationManager authenticationManagerBean() {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService);
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {

        // a.登录页面权限开放给所有人
        http.formLogin().loginPage("/login").failureUrl("/login?error").permitAll()
                .and()
                .authorizeRequests()
                .antMatchers(
                        "/static/**", "/css/**", "/images/**", "/error/**", "/actuator/**",
                        "/register/**", "/oauth/error/**", "/oauth/confirm_access/**", "/logout/**", "/token/**")
                .permitAll()
                .and().authorizeRequests().anyRequest().authenticated()
                .and()
                .exceptionHandling()
                .authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/login"))
                .and()
                // 添加自定义登录检验提供者
                .authenticationProvider(new LoginAuthenticationProvider(userDetailsService, passwordEncoder(), defaultPassword))
                .csrf().disable();

    }
}
```