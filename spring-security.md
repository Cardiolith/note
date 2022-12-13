### Spring Security

#### 1. Top Class

> Authentication	用于存储用户的认证信息

```java
public interface Authentication extends Principal, Serializable {
       Collection<? extends GrantedAuthority> getAuthorities();
       Object getCredentials();
       Object getDetails();
       Object getPrincipal();
       boolean isAuthenticated();
       void setAuthenticated(boolean isAuthenticated);
}
```



> AuthenticationManager	负责认证工作

```java
public interface AuthenticationManager {
       Authentication authenticate(Authentication authentication)
           throws AuthenticationException;
}
```



> AuthenticationProvider

```java
public interface AuthenticationProvider {
       Authentication authenticate(Authentication authentication)
            throws AuthenticationException;
       boolean supports(Class<?> authentication);
}
```



##### 关于登录数据保存

如果不使用Spring Security这一类的安全管理框架，大部分的开发者可能会将登录用户数据保存在Session中，事实上，Spring Security也是这么做的。但是，为了使用方便，Spring Security在此基础上还做了一些改进，其中最主要的一个变化就是线程绑定。
当用户登录成功后，Spring Security会将登录成功的用户信息保存到SecurityContextHolder中。SecurityContextHolder中的数据保存默认是通过**ThreadLocal**来实现的，使用ThreadLocal创建的变量只能被当前线程访问，不能被其他线程访问和修改，也就是用户数据和请求线程绑定在一起。当登录请求处理完毕后，Spring Security会将SecurityContextHolder中的数据拿出来保存到**Session**中，同时将SecurityContextHolder中的数据清空。以后每当有请求到来时，Spring Security就会先从Session中取出用户登录数据，保存到SecurityContextHolder中，方便在该请求的后续处理过程中使用，同时在请求结束时将SecurityContextHolder中的数据拿出来保存到Session中，然后将SecurityContextHolder中的数据清空。
这一策略非常方便用户在Controller或者Service层获取当前登录用户数据，但是带来的另外一个问题就是，在子线程中想要获取用户登录数据就比较麻烦。Spring Security对此也提供了相应的解决方案，如果开发者使用@Async注解来开启异步任务的话，那么只需要添加如下配置，使用Spring Security提供的异步任务代理，就可以在异步任务中从Security ContextHolder里边获取当前登录用户的信息：



#### 2. 认证

![image-20211105213512229](C:\Users\tangc\AppData\Roaming\Typora\typora-user-images\image-20211105213512229.png)

> UserDetails	定义用户对象

```java
public interface UserDetails extends Serializable {
       Collection<? extends GrantedAuthority> getAuthorities();
       String getPassword();
       String getUsername();
       boolean isAccountNonExpired();
       boolean isAccountNonLocked();
       boolean isCredentialsNonExpired();
       boolean isEnabled();
}
```



> UserDetailsServiceAutoConfiguration 自动化配置类-默认生成InMemoryUserDetailsManager

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({AuthenticationManager.class})
@ConditionalOnBean({ObjectPostProcessor.class})
@ConditionalOnMissingBean(
    value = {AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class},
    type = {"org.springframework.security.oauth2.jwt.JwtDecoder", "org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector"}
)
public class UserDetailsServiceAutoConfiguration {
    private static final String NOOP_PASSWORD_PREFIX = "{noop}";
    private static final Pattern PASSWORD_ALGORITHM_PATTERN = Pattern.compile("^\\{.+}.*$");
    private static final Log logger = LogFactory.getLog(UserDetailsServiceAutoConfiguration.class);

    public UserDetailsServiceAutoConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean(
        type = {"org.springframework.security.oauth2.client.registration.ClientRegistrationRepository"}
    )
    @Lazy
    public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties, ObjectProvider<PasswordEncoder> passwordEncoder) {
        User user = properties.getUser();
        List<String> roles = user.getRoles();
        return new InMemoryUserDetailsManager(new UserDetails[]{org.springframework.security.core.userdetails.User.withUsername(user.getName()).password(this.getOrDeducePassword(user, (PasswordEncoder)passwordEncoder.getIfAvailable())).roles(StringUtils.toStringArray(roles)).build()});
    }

    private String getOrDeducePassword(User user, PasswordEncoder encoder) {
        String password = user.getPassword();
        if (user.isPasswordGenerated()) {
            logger.info(String.format("%n%nUsing generated security password: %s%n", user.getPassword()));
        }

        return encoder == null && !PASSWORD_ALGORITHM_PATTERN.matcher(password).matches() ? "{noop}" + password : password;
    }
}
```



> DefaultLoginPageGeneratingFilter	处理登录出错/注销/登录请求中任何一个拦截

> DefaultLogoutPageGeneratingFilter	拦截登出请求

> AuthenticationSuccessHandler	

```java
public interface AuthenticationSuccessHandler {
    default void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authentication) throws IOException, ServletException {
        this.onAuthenticationSuccess(request, response, authentication);
        chain.doFilter(request, response);
    }

    void onAuthenticationSuccess(HttpServletRequest var1, HttpServletResponse var2, Authentication var3) throws IOException, ServletException;
}
```

 
