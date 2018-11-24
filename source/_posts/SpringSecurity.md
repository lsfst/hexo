---
title: Springboot 整合Spring Security 登录验证
date: 2018-06-21
tags: [Springboot,Spring Security]
categories: Java
toc: true
---
公司后台管理项目之前采用的是shiro做权限验证，前段时间花了点时间替换成了Spring Security，现在有时间将配置过程整理了一下。


### pom.xml引入依赖

{% codeblock  lang:java  %}
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
	<version>2.0.2.RELEASE</version>
</dependency>
{% endcodeblock %}

### 添加Spring Security配置类

{% codeblock  lang:java  %}
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
 @Override
    public void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers("/css/**", "/js/**","/fonts/**").permitAll()
                .anyRequest().authenticated();
    }
}
{% endcodeblock %}

添加这个类后，基本的验证功能就有了

#### 关于@EnableWebSecurity
@EnableWebSecurity与WebSecurityConfigurerAdapter一起配合即可提供基于web的security，这俩是整个Spring Security配置的基础。


{% codeblock EnableWebSecurity.class  lang:java %}
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({WebSecurityConfiguration.class, ObjectPostProcessorConfiguration.class, SpringWebMvcImportSelector.class})
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
    boolean debug() default false;
}
{% endcodeblock %}

这里需要注意的是@EnableGlobalAuthentication这个注解

{% codeblock EnableGlobalAuthentication.class lang:java %}
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({AuthenticationConfiguration.class})
@Configuration
public @interface EnableGlobalAuthentication {
}
{% endcodeblock %}

在这个注解中，又导入了另外一个配置类AuthenticationConfiguration，AuthenticationConfiguration注册了AuthenticationManagerBuilder，其作用是对用户提交的用户名和密码进行验证，它是Spring Security账户验证的核心

{% codeblock lang:java %}
@Bean
    public AuthenticationManagerBuilder authenticationManagerBuilder(ObjectPostProcessor<Object> objectPostProcessor) {
        return new AuthenticationManagerBuilder(objectPostProcessor);
    }
{% endcodeblock %}

与HttpSecurity类似，AuthenticationManagerBuilder也是SecurityBuilder的一个子类。不同的是，HttpSecurity使用到的SecurityConfigurer基本上最终产生的都是一个过滤器，而AuthenticationManagerBuilder使用到SecurityConfiguer最终产生的都是AuthenticationManager的一个子类实例ProviderManager。ProviderManager类的创建是通过performBuild方法创建的。

{% codeblock lang:java %}
protected ProviderManager performBuild() throws Exception {
   if (!isConfigured()) {
      logger.debug("No authenticationProviders and no parentAuthenticationManager defined. Returning null.");
      return null;
   }
   //创建ProviderManager实例，用于管理验证提供者AuthenticationProvider
   ProviderManager providerManager = new ProviderManager(authenticationProviders,
         parentAuthenticationManager);
   if (eraseCredentials != null) {
      providerManager.setEraseCredentialsAfterAuthentication(eraseCredentials);
   }
   if (eventPublisher != null) {
      providerManager.setAuthenticationEventPublisher(eventPublisher);
   }
   providerManager = postProcess(providerManager);
   return providerManager;
}
{% endcodeblock %}

SpringSecurity的每一种方式都对应一个provider。如果需要联合使用多种验证方式，ProviderManager就可以帮助我们来管理这些provider，例如先用谁验证，后用谁验证，以及是否只要有一个provider验证成功就算用户已经成功验证等。 AuthenticationProvider的创建：

{% codeblock lang:java %}
    public InMemoryUserDetailsManagerConfigurer<AuthenticationManagerBuilder> inMemoryAuthentication() throws Exception {
        return (InMemoryUserDetailsManagerConfigurer)this.apply(new InMemoryUserDetailsManagerConfigurer());
    }

    public JdbcUserDetailsManagerConfigurer<AuthenticationManagerBuilder> jdbcAuthentication() throws Exception {
        return (JdbcUserDetailsManagerConfigurer)this.apply(new JdbcUserDetailsManagerConfigurer());
    }

    public <T extends UserDetailsService> DaoAuthenticationConfigurer<AuthenticationManagerBuilder, T> userDetailsService(T userDetailsService) throws Exception {
        this.defaultUserDetailsService = userDetailsService;
        return (DaoAuthenticationConfigurer)this.apply(new DaoAuthenticationConfigurer(userDetailsService));
    }

    public LdapAuthenticationProviderConfigurer<AuthenticationManagerBuilder> ldapAuthentication() throws Exception {
        return (LdapAuthenticationProviderConfigurer)this.apply(new LdapAuthenticationProviderConfigurer());
    }
{% endcodeblock %}

用户在登陆时，会被登陆验证拦截器AuthenticationProcessingFilter拦截，调用AuthenticationManager的实现，而AuthenticationManager会调用ProviderManager来获取用户验证信息，如果验证通过后会将用户的权限信息封装成一个User放到spring的全局缓存SecurityContextHolder中，以备后面访问资源时使用。登陆成功访问资源（即授权管理）时，会通过AbstractSecurityInterceptor拦截器拦截，其中会调用FilterInvocationSecurityMetadataSource的方法来获取被拦截url所需的全部权限，然后调用授权管理器AccessDecisionManager，这个授权管理器会通过spring的全局缓存SecurityContextHolder获取用户的权限信息，还会获取被拦截的url和被拦截url所需的全部权限，然后根据所配的策略（有：一票决定，一票否定，少数服从多数等），如果权限足够，则返回，权限不够则报错并调用权限不足页面。
                                                                                                                                                                                                      
#### 开启Spring Security自带注解
Spring Security默认禁用注解，要想开启注解，需要在继承WebSecurityConfigurerAdapter的类上加@EnableGlobalMethodSecurity注解，并在该类中将AuthenticationManager定义为Bean

{% codeblock lang:java %}
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
{% endcodeblock %}

{% codeblock lang:java %}
@Bean
    @Override
    protected AuthenticationManager authenticationManager() throws Exception {
        return super.authenticationManager();
    }
{% endcodeblock %}

JSR-250注解：
@DenyAll 拒绝所有访问
@RolesAllowed({"USER", "ADMIN"}) @PermitAll 允许所有访问

prePostEnabled注解：
@PreAuthorize
@PreAuthorize
@PostAuthorize
@PostAuthorize

securedEnabled注解：
@Secured

### 其他配置
#### 基于UserDetailsService的认证服务
spring security提供了多种认证方式(内存、JDBC、LDAP和自定义UserDetailService验证)，不管是哪一种验证方式，都是通过一个自动注入的AuthenticationManagerBuilder对象来完成的。这个类用于构建AuthenticationManager，其作用是对用户提交的用户名和密码进行验证

##### 定义SecurityUser用户实体类


{% codeblock SecurityUser lang:java %}
public class User implements UserDetails, CredentialsContainer {
    private static final long serialVersionUID = 410L;
    private String password;
    private final String username;
    private final Set<GrantedAuthority> authorities;
    private final boolean accountNonExpired;
    private final boolean accountNonLocked;
    private final boolean credentialsNonExpired;
    private final boolean enabled;
    
    ......
}
{% endcodeblock %}

项目中经常需要获取登录用户的角色信息，因此这里直接将role信息存进SecurityUser类中

{% codeblock SecurityUser lang:java %}
public class SecurityUser extends User {
    private String role;

    public SecurityUser(String username, String password, Collection<? extends GrantedAuthority> authorities) {
        super(username, password, authorities);
    }

    public SecurityUser(String username, String password, boolean enabled, boolean accountNonExpired, boolean credentialsNonExpired, boolean accountNonLocked, Collection<? extends GrantedAuthority> authorities) {
        super(username, password, enabled, accountNonExpired, credentialsNonExpired, accountNonLocked, authorities);
    }

    public SecurityUser(String username, String password, String role, Collection<? extends GrantedAuthority> authorities) {
        super(username, password, authorities);
        this.role = role;
    }

    public String getRole() {
        return this.role;
    }

    public void setRole(String role) {
        this.role = role;
    }

}
{% endcodeblock %}

##### 实现UserDetailsService账号验证

{% codeblock SecurityUser lang:java %}
@Service
public class SecurityUserService implements UserDetailsService {

    private static Logger logger= LoggerFactory.getLogger( SecurityUserService.class );

    @Autowired
    EouCfrmUserService userService;
    @Autowired
    EouCfrmAuthorizationsService authorizationService;
    @Autowired
    EouAgentsService agentsService;

    @Override
    public SecurityUser loadUserByUsername( String username )  {

        TbCfrmUser user=userService.selectByPrimaryKey( username );

        if(user==null){
            logger.info( "用户名不存在" );
            throw new BadCredentialsException("用户名不存在");
        }
        List<GrantedAuthority > grantedAuthorities  = new ArrayList<GrantedAuthority>();

        List<FrameAuthorization > authorizations = authorizationService.selectAuthorizationByRole( user.getIdxRoleId_tbRole() );

        GrantedAuthority grantedAuthority = new SimpleGrantedAuthority("ROLE_"+user.getIdxRoleId_tbRole());
        //1：此处将权限信息添加到 GrantedAuthority 对象中，在后面进行权限验证时会使用GrantedAuthority 对象。
        grantedAuthorities.add(grantedAuthority);
        for(FrameAuthorization authorization:authorizations){
            //默认情况下，GrantedAuthority对象存储的是用户role信息，默认前缀ROLE_，可以按照实际情况自由配置
            //此处将权限信息添加到 GrantedAuthority 对象中，在后面进行权限验证时会使用GrantedAuthority 对象。
                    grantedAuthority = new SimpleGrantedAuthority(authorization.getUrl());
                    grantedAuthorities.add(grantedAuthority);
        }

        String role=user.getIdxRoleId_tbRole();
        if(role.equals( "admin" )){

        }else if(role.equals( "agent" )){
            try {
                TbAgent agent=agentsService.selectAgentByAgentName( username );
                idxOwnerId=agent.getIdxAgentId()+".";
            }catch ( Exception e){
                e.printStackTrace();
                throw new BadCredentialsException("代理账号异常");
            }

        }else {
            throw new BadCredentialsException("该账号暂无权限");
        }
        return  new SecurityUser(user.getKeyUserId(), user.getPassword(),user.getIdxRoleId_tbRole(), grantedAuthorities);
    }
}
{% endcodeblock %}

这里有一个坑，用户在登录失败时，需要根据错误信息提示用户是账号密码错误。但在Spring Security中，默认情况下不管你是用户名不存在，密码错误，还是其他错误，都会转换成Bad credentials异常信息，而不是具体的错误。原因在于DaoAuthenticationProvider的父类AbstractUserDetailsAuthenticationProvider的authenticate方法中进行了处理：
{% codeblock lang:java %} 
try {  
    user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);  
} catch (UsernameNotFoundException notFound) {  
    logger.debug("User '" + username + "' not found");  
  
    if (hideUserNotFoundExceptions) {  
        throw new BadCredentialsException(messages.getMessage(  
                "AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));  
    } else {  
        throw notFound;  
    }  
}   
{% endcodeblock %}

所以如果这里需要做自定义过滤验证，可以直接抛出BadCredentialsException。
如果前端是JSP，可以通过 ${sessionScope.SPRING_SECURITY_LAST_EXCEPTION.message} 显示错误信息

#### md5加密
{% codeblock lang:java %}
public class MyMessageDigestPasswordEncoder extends MessageDigestPasswordEncoder {

    private static Logger logger= LoggerFactory.getLogger( MyMessageDigestPasswordEncoder.class );

    public MyMessageDigestPasswordEncoder( String algorithm ) {
        super( algorithm );
    }

    public MyMessageDigestPasswordEncoder( String algorithm, boolean encodeHashAsBase64 ) throws IllegalArgumentException {
        super( algorithm, encodeHashAsBase64 );
    }
    //即使项目重启，页面刷新第一时间会在isPasswordValid方法中验证账号信息，验证成功后前往loginsuccesshandler
    public boolean isPasswordValid( String encPass, String rawPass, Object salt ) {
        String pass1 = "" + encPass;
        String pass2 = MD5Utils.md5( rawPass );
        //官方文档是通过encodePassword加盐加密，这个可以根据实际需求随意修改
        logger.info( "encPass:" + encPass + ";rawPass:" + rawPass + "pass1:" + pass1 + ";pass2:" + pass2 );
        boolean bool = false;
        if ( pass1.equals( pass2 ) ) {
            bool = true;
        }
        return bool;
    }
    
        //自定义md5加密
    public static String md5(String strObj) {
        String resultString = null;

        try {
            new String(strObj);
            MessageDigest ex = MessageDigest.getInstance("MD5");
            resultString = byteToString(ex.digest(strObj.getBytes()));
        } catch (NoSuchAlgorithmException var3) {
            var3.printStackTrace();
        }

        return resultString;
    }
}
 {% endcodeblock %}

#### 静态资源控制
HttpSecurity的ignoring()与WebSecurity的permitAll()都可以控制静态资源。 WebSecurityConfigurerAdapter提供了三个configure方法，分别提供对AuthenticationManagerBuilder，WebSecurity与HttpSecurity的配置，其中AuthenticationManagerBuilder进行账号认证配置。
WebSecurity主要是跟web资源相关的配置，HttpSecurity则是对所有http请求进行管理，两者可以控制静态资源的权限，但本质的区别在于：
{% blockquote %}
HttpSecurity的ignoring()完全绕过了spring security的所有filter，相当于不走验证，比较适合配置前端相关的静态资源；
HttpSecurity的permitAll()则没有绕过spring security，其中包含了登录的以及匿名的，会给没有登录的用户适配一个AnonymousAuthenticationToken，设置到SecurityContextHolder，方便后面的filter可以统一处理authentication。
{% endblockquote %}

#### CSRF 
{% codeblock lang:java %}
public class CsrfSecurityRequestMatcher implements RequestMatcher {

    private static Logger logger = LoggerFactory.getLogger( CsrfSecurityRequestMatcher.class );
    private Pattern allowedMethods = Pattern.compile( "^(GET|HEAD|TRACE|OPTIONS)$" );
    private RegexRequestMatcher unprotectedMatcher = new RegexRequestMatcher( "^/api/.*", null );

    @Override
    public boolean matches( HttpServletRequest httpServletRequest ) {
        String uri=httpServletRequest.getRequestURI();
        if ( allowedMethods.matcher( httpServletRequest.getMethod() ).matches()) {
            return false;
        }

        return !unprotectedMatcher.matches( httpServletRequest );
    }
 }
 {% endcodeblock %}
 
 开启CSRF后，页面所有表单提交都需要带上token，如果页面是jsp/thymeleaf模板可以将token存进通用的页面中
 {% codeblock lang:js %}
 <meta name="_csrf" content="${_csrf.token}"/>
 <meta name="_csrf_header" content="${_csrf.headerName}"/>
  {% endcodeblock %}
  
  如果想偷懒的话，也可以直接在CsrfSecurityRequestMatcher中对相应接口放行，我就是这么做的(逃
  
  注意：开启csrf后注销需使用post表单提交logout
  
  #### 自定义LoginSuccessHandler
  {% codeblock lang:java %}
  public class LoginSuccessHandler extends SavedRequestAwareAuthenticationSuccessHandler {
      private static Logger logger = LoggerFactory.getLogger( LoginSuccessHandler.class );
      private static String defaultTargetUrl = "/index";
  
      @Override
      public void onAuthenticationSuccess( HttpServletRequest request,
                                           HttpServletResponse response, Authentication authentication ) throws IOException,
              ServletException {
          //获得授权后可得到用户信息
          SecurityUser userDetails = ( SecurityUser ) authentication.getPrincipal();
          logger.info( "[" + userDetails.getRole() + "]|" + userDetails.getUsername() + "|[" + IPUtils.getIpAddress( request ) + "] login success" );
          request.getRequestDispatcher( defaultTargetUrl ).forward( request, response );
      }
  }
   {% endcodeblock %}
   
   #### 自定义安全过滤器
   相对于Springboot大部分的傻瓜式配置来说，这一步算是稍微复杂点的。
   AbstractSecurityInterceptor是认证和授权的集成 ,没有继承和实现任何和过滤器相关的类，具体和过滤器有关的部分由其子类所实现。每一种受保护对象都拥有继承自AbstrachSecurityInterceptor的拦截器类。spring security 提供了两个具体实现类，MethodSecurityInterceptor 用于受保护的方法，FilterSecurityInterceptor 用于受保护的web 请求，spring security 默认的过滤器是FilterSecurityInterceptor。两者具体工作流程为：
    {% blockquote %}
    查找当前请求里分配的配置属性。
    把安全对象，当前的Authentication和配置属性,提交给AccessDecisionManager来进行以此认证决定。
    有可能在调用的过程中,对Authentication进行修改。
    允许安全对象进行处理（假设访问被允许了）。
    在调用返回的时候执行配置的AfterInvocationManager。如果调用引发异常,AfterInvocationManager将不会被调用。
      {% endblockquote %}
    权限鉴定是由AccessDecisionManager 接口中的decide()方法负责的。decide() 方法需要接收一个受保护对象对应的configAttribute集合的。一个configAttribute可能只是一个简单的角色名称，具体将视AccessDecisionManager的实现者而定。由于我们需要自定义过滤器，所以需要重写AbstrachSecurityInterceptor的实现。
    
   {% codeblock lang:java %}
   
  @Service 
  public class MyFilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {

      @Autowired
      public MyInvocationSecurityMetadataSourceService myInvocationSecurityMetadataSourceService;
  
      @Autowired
      public void setMyAccessDecisionManager( MyAccessDecisionManager myAccessDecisionManager ) {
          super.setAccessDecisionManager( myAccessDecisionManager );
      }
  
      @Override
      public void init( FilterConfig filterConfig ) throws ServletException {}
  
      @Override
      public void doFilter( ServletRequest request, ServletResponse response, FilterChain chain ) throws IOException, ServletException {
          FilterInvocation fi = new FilterInvocation(request, response, chain);
          invoke(fi);
      }
  
      public void invoke( FilterInvocation fi ) throws IOException, ServletException {
          //fi里面有一个被拦截的url
          //里面调用MyInvocationSecurityMetadataSource的getAttributes(Object object)这个方法获取fi对应的所有权限
          //再调用MyAccessDecisionManager的decide方法来校验用户的权限是否足够
          InterceptorStatusToken token = super.beforeInvocation( fi );
          try {
              //执行下一个拦截器
              fi.getChain().doFilter( fi.getRequest(), fi.getResponse() );
          } finally {
              super.afterInvocation( token, null );
          }
      }
  
      @Override
      public void destroy() {}
  
      @Override
      public Class< ? > getSecureObjectClass() {
          return FilterInvocation.class;
      }
      @Override
      public SecurityMetadataSource obtainSecurityMetadataSource() {
          return this.myInvocationSecurityMetadataSourceService;
      }
} 
  
  @Service
  public class MyInvocationSecurityMetadataSourceService implements FilterInvocationSecurityMetadataSource {
      @Autowired
      public TbCfrmUserMapper cfrmUserMapper;
  
      private static Logger logger= LoggerFactory.getLogger( MyInvocationSecurityMetadataSourceService.class );
      private static HashMap<String, Collection<ConfigAttribute>> map =null;
  
      /**
       * 加载权限表中所有权限
       */
      public void loadResourceDefine(){
      
          logger.info( "=================loadResourceDefine=================" );
          map = new HashMap();
          Collection<ConfigAttribute> array;
          ConfigAttribute cfg;
          String role= SecurityUtils.getRole();
          List<FrameAuthorization > authorizations = cfrmUserMapper.selectAuthorizationByRole(role);
          
          for(FrameAuthorization authorization : authorizations) {
             cfg = new SecurityConfig(authorization.getUrl());
              //此处添加的信息将会作为MyAccessDecisionManager类的decide的第三个参数。
              array.add(cfg);
              //用权限的getUrl() 作为map的key，用ConfigAttribute的集合作为 value，
              map.put(authorization.getUrl(), array);
          }
      }
  
      //判定用户请求的url 是否在权限表中，如果在权限表中，则返回给 decide 方法，用来判定用户是否有此权限。如果不在权限表中则放行。
      @Override
      public Collection< ConfigAttribute > getAttributes( Object object ) throws IllegalArgumentException {
      
          if(map ==null) loadResourceDefine();
          //object 中包含用户请求的request 信息
          HttpServletRequest request = ((FilterInvocation ) object).getHttpRequest();
          AntPathRequestMatcher matcher;
          String resUrl;
          
          for( Iterator<String> iter = map.keySet().iterator(); iter.hasNext(); ) {
              resUrl = iter.next();
              matcher = new AntPathRequestMatcher(resUrl);
              if(matcher.matches(request)) {
                  return map.get(resUrl);
              }
          }
          return null;
      }
  
      @Override
      public Collection< ConfigAttribute > getAllConfigAttributes() {
          return null;
      }
  
      @Override
      public boolean supports( Class< ? > aClass ) {
          return true;
      }
  }
  
  //权限验证的核心管理器，重写decide方法可按照实际需求进行验证(比如实现投票器，多对一，一对多，一对一等)。
  @Service
  public class MyAccessDecisionManager implements AccessDecisionManager {
      Logger logger= LoggerFactory.getLogger( MyAccessDecisionManager.class );
      
      //authentication 是SecurityUserService中循环添加到 GrantedAuthority 对象中的权限信息集合.
      //object 包含客户端发起的请求的requset信息，可转换为 HttpServletRequest request = ((FilterInvocation) object).getHttpRequest();
      //configAttributes 为MyInvocationSecurityMetadataSource的getAttributes(Object object)这个方法返回的结果，此方法是为了判定用户请求的url 是否在权限表中，如果在权限表中，则返回给 decide 方法，用来判定用户是否有此权限。如果不在权限表中则放行。
      //也即在loadResourceDefine必须对所有需要配置权限的url进行注册，未注册的url不会去检测权限
      //在Authentication中则注册角色权限
      @Override
      public void decide( Authentication authentication, Object o, Collection< ConfigAttribute > configAttributes ) throws AccessDeniedException, InsufficientAuthenticationException {
  
          if(null== configAttributes || configAttributes.size() <=0) {
              return;
          }
          ConfigAttribute c;
          String needRole;
          
          for( Iterator<ConfigAttribute> iter = configAttributes.iterator(); iter.hasNext(); ) {
              c = iter.next();
              needRole = c.getAttribute();
              for(GrantedAuthority ga : authentication.getAuthorities()) {//authentication 为在注释1 中循环添加到 GrantedAuthority 对象中的权限信息集合
                  if(needRole.trim().equals(ga.getAuthority())) {
                      return;
                  }
              }
          }
  
          logger.info( "AccessDecisionManager decide:no right" );
          throw new AccessDeniedException("no right");
      }
  
      //全部改为true
      @Override
      public boolean supports( ConfigAttribute configAttribute ) {
          return true;
      }
  
      @Override
      public boolean supports( Class< ? > aClass ) {
          return true;
      }
  }
  
{% endcodeblock %} 
 注意：这里有个坑，由于项目采用Mybatis作为持久化框架,FilterInvocationSecurityMetadataSource中采用@Autowired注入mapper时可能会报错，建议在xml里面配置强制注入mapper。

#### Basic认证 
如果项目提供的是restful服务，需要对请求进行basic认证，也非常简单

{% codeblock lang:java %}
@Override
    protected void configure( HttpSecurity http ) throws Exception {
        http.authorizeRequests().
                requestMatchers( CorsUtils::isPreFlightRequest).permitAll().    //跨域设置：预请求放行
                anyRequest().authenticated()
                .and()
                .csrf().disable()    //禁用csrf
                .httpBasic();
    }
{% endcodeblock %}
     
#### principal获取用户信息
{% codeblock lang:java %}
将获取登录用户信息的方法进行整理，便于之后使用
public class SecurityUtils {
    public static SecurityUser getSecurityUser() {
        Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        return principal != null && !principal.equals("anonymousUser")?(SecurityUser)principal:null;
    }

    public static String getUserName() {
        SecurityUser user = getSecurityUser();
        return user == null?"":user.getUsername();
    }

    public static String getRole() {
        SecurityUser user = getSecurityUser();
        return user == null?"":user.getRole();
    }
}
{% endcodeblock %}

### 收工
最后完成的WebSecurityConfig配置类就是这样子了：
     {% codeblock lang:java %}
     @Configuration
     @EnableWebSecurity
     @EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)
     public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
     
         @Bean
         public MyFilterSecurityInterceptor myFilterSecurityInterceptor(){
             return new MyFilterSecurityInterceptor();
         }
     
         @Bean
         UserDetailsService SecurityUserService() { //注册UserDetailsService 的bean
             return new SecurityUserService();
         }
     
         @Bean
         CsrfSecurityRequestMatcher CsrfSecurityRequestMatcher() { //注册UserDetailsService 的bean
             return new CsrfSecurityRequestMatcher();
         }
     
         @Bean
         MyMessageDigestPasswordEncoder MyMessageDigestPasswordEncoder() { //注册md5加密
             return new MyMessageDigestPasswordEncoder( "md5" );
         }
     
         @Bean
         LoginSuccessHandler LoginSuccessHandler() { //登录成功后可使用loginSuccessHandler()处理逻辑。
             return new LoginSuccessHandler();
         }
     
         @Bean
         MyLoginUrlAuthenticationEntryPoint MyLoginUrlAuthenticationEntryPoint() {
             return new MyLoginUrlAuthenticationEntryPoint("/login");
         }
     
         @Bean
         @Override
         protected AuthenticationManager authenticationManager() throws Exception {
             return super.authenticationManager();
         }
         
         @Override
             protected void configure( AuthenticationManagerBuilder auth ) throws Exception {
                 auth.userDetailsService( SecurityUserService() ).passwordEncoder( MyMessageDigestPasswordEncoder() );
             }
         
             @Override
             public void configure(WebSecurity web) throws Exception {
                 //HttpSecurity优先级高于WebSecurity
                 web.ignoring().antMatchers( "/assets/**" );
             }
         
             @Override
             protected void configure( HttpSecurity http ) throws Exception {
         
                 http.authorizeRequests()
                         .antMatchers("/login" , "/state/**","/api/**").permitAll()
                         .antMatchers( "/frame/**" ).hasRole( "admin" )       //hasRole/hasAuthority本质上一样
                         .anyRequest().authenticated()
                         .and()
                         .formLogin()
                         .loginPage( "/login" )
                         .loginProcessingUrl( "/login" )
         //                .successForwardUrl( "/index" )  //登录是post请求，成功后跳转至index页面，需要index也支持post
                         .successHandler( LoginSuccessHandler() )  //配置successHandler后,successForwardUrl会失效，默认重定向路径"/"
                         .failureUrl( "/login" )
                         .and()
                         .csrf()
         //                .disable()
                         .requireCsrfProtectionMatcher( CsrfSecurityRequestMatcher() )  //配置csrf
                         .and()
                         .rememberMe()
                         .tokenRepository( new InMemoryTokenRepositoryImpl() )
                         .tokenValiditySeconds( 1209600 )/*记住登录状态2周*/
                         .and()
                         .logout()
                         .logoutRequestMatcher( new AntPathRequestMatcher( "/logout", "POST" ) )   //开启csrf后注销需使用post表单提交
         //                .deleteCookies( "remember-me" )
         //                .deleteCookies( "JSESSIONID","remember-me" )
                         .clearAuthentication( true )
                         .invalidateHttpSession( true )
                         .logoutSuccessUrl( "/login" )
                         .and()
                         .sessionManagement()
                         .invalidSessionUrl("/login")
                         .maximumSessions( 1 )   //设置最大登录数1，后面登录会踢掉当前用户
                         .expiredUrl( "/login" );
         
                         http.addFilterBefore( myFilterSecurityInterceptor(), FilterSecurityInterceptor.class );
             }
}
     {% endcodeblock %}