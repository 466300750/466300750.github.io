---
layout: post
title: spring security 3 -- Authentication Provider
date: 2018-09-14 14:48:31
categories: Spring
share: y
excerpt_separator: <!--more-->
---

<!--more-->

### demo-3

```
@Configuration
@EnableWebSecurity
public class OperationWebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private OperationLoginAuthenticationProvider loginAuthenticationProvider;

    @Autowired
    private OperationJwtAuthenticationProvider jwtAuthenticationProvider;

    @Autowired
    private JwtAuthenticationEntryPoint authenticationEntryPoint;

    @Autowired
    private OperationVerificationCodeService verificationCodeService;


    public void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(loginAuthenticationProvider);
        auth.authenticationProvider(jwtAuthenticationProvider);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf()
            .disable()
            .antMatcher("/operation/**")
            .authorizeRequests()
            .antMatchers(GET, "/operation/soldtos").hasAuthority(CUSTOMER_MANAGEMENT.name())
            .anyRequest()
            .authenticated()
            .and()
            .addFilterBefore(new JwtOperationUserPasswordLoginFilter("/operation/login", this.authenticationManager(), verificationCodeService), UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(new JwtAuthenticationFilter(this.authenticationManager()), UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling()
            .authenticationEntryPoint(authenticationEntryPoint)
            .and()
            .sessionManagement()
            .sessionCreationPolicy(STATELESS);
    }
}
```

```
public class JwtOperationUserPasswordLoginFilter extends AbstractAuthenticationProcessingFilter {
    private ObjectMapper objectMapper = new Dcsp2ObjectMapper();
    private JwtLoginFailureHandler loginFailureHandler = new JwtLoginFailureHandler(objectMapper);
    private OperationVerificationCodeService verificationCodeService;

    public JwtOperationUserPasswordLoginFilter(String processesUrl,
                                               AuthenticationManager authenticationManager,
                                               OperationVerificationCodeService verificationCodeService) {
        super(processesUrl);
        this.verificationCodeService = verificationCodeService;
        setAuthenticationManager(authenticationManager);
        setAuthenticationFailureHandler(new JwtAuthenticationFailureHandler(objectMapper));
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        try {
            return getAuthentication(request);
        } catch (Exception e) {
            throw new AuthenticationServiceException(e.getMessage(), e);
        }
    }

    private Authentication getAuthentication(HttpServletRequest request) {
        if (!POST.name().equals(request.getMethod())) {
            throw new AuthenticationServiceException("只支持POST方法.");
        }

        OperationUsernamePasswordLoginCommand loginCommand;
        try {
            loginCommand = objectMapper.readValue(request.getReader(), OperationUsernamePasswordLoginCommand.class);
        } catch (Exception e) {
            throw new AuthenticationServiceException("无法解析登录请求.");
        }

        String userName = loginCommand.getUsername();
        String password = loginCommand.getPassword();

        if (isEmpty(userName) || isEmpty(password)) {
            throw new AuthenticationServiceException("用户名或密码没有提供.");
        }

        verificationCodeService.verify(loginCommand.getPreCode(), loginCommand.getVerificationCode());

        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(userName, password);

        return this.getAuthenticationManager().authenticate(token);
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
                                            Authentication authResult) throws IOException {
        JwtToken jwtToken = (JwtToken) authResult.getCredentials();
        response.setStatus(OK.value());
        response.setContentType(APPLICATION_JSON_VALUE);
        objectMapper.writeValue(response.getWriter(), jwtToken);
    }


    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                              AuthenticationException e) throws IOException {
        loginFailureHandler.handelFailure(response, e);
    }

}
```

```
public class OperationUsernamePasswordLoginCommand {
    private final String username;

    @Length(max = 50, message = AUTHENTICATION_FAILED_MESSAGE)
    private final String password;

    private final String preCode;

    @Length(max = 10, message = "验证码填写错误")
    private final String verificationCode;

    @JsonCreator
    public OperationUsernamePasswordLoginCommand(@JsonProperty("userName") String username,
                                                 @JsonProperty("password") String password,
                                                 @JsonProperty("preCode") String preCode,
                                                 @JsonProperty("verificationCode") String verificationCode) {
        this.username = username;
        this.password = password;
        this.preCode = preCode;
        this.verificationCode = verificationCode;
    }
}
```

```
@Component
public class OperationLoginAuthenticationProvider implements AuthenticationProvider {
    public static final String AUTHENTICATION_FAILED_MESSAGE = "您输入的账户名或密码错误";

    private PasswordEncoder passwordEncoder;
    private OperationUserRepository userRepository;
    private OperationJwtService jwtService;

    public OperationLoginAuthenticationProvider(PasswordEncoder passwordEncoder, OperationUserRepository userRepository, OperationJwtService jwtService) {
        this.passwordEncoder = passwordEncoder;
        this.userRepository = userRepository;
        this.jwtService = jwtService;
    }

    @Override
    @Transactional
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = (String) authentication.getPrincipal();
        String password = (String) authentication.getCredentials();

        OperationUser user = null;
        try {
            user = userRepository.byEmailOrMobile(username);
        } catch (CommonResourceNotFoundException e) {
            throw new BadCredentialsException(AUTHENTICATION_FAILED_MESSAGE);
        }

        if (passwordNotMatch(password, user.getPassword())) {
            throw new BadCredentialsException(AUTHENTICATION_FAILED_MESSAGE);
        }

        if (isOperationUserDisable(user)) {
            throw new CommonBadRequestException(OPERATION_USER_DISABLE, "该账号已停用，请联系管理员");
        }

        JwtToken jwtToken = jwtService.generateJwtToken(user.toPrincipal());

        return new JwtAuthenticationToken(username, jwtToken, null);
    }

    private boolean isOperationUserDisable(OperationUser user) {
        return user.getState() == OperationUserState.DISABLED;
    }


    private boolean passwordNotMatch(String rawPassword, String encodedPassword) {
        return !passwordEncoder.matches(rawPassword, encodedPassword);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

```
public class JwtAuthenticationToken extends AbstractAuthenticationToken {
    private Object principal;
    private String jwtToken;

    public JwtAuthenticationToken(Object principal, String jwtToken, Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        this.jwtToken = jwtToken;
    }

    @Override
    public Object getCredentials() {
        return jwtToken;
    }

    @Override
    public Object getPrincipal() {
        return principal;
    }

    public String getJwtToken() {
        return jwtToken;
    }

}
```

```
@Component
public class JwtTokenService implements Serializable {
    private final String secret = "12345678";

    public String generateToken(UserPrincipal userPrincipal) {

        byte[] encodedKey = Base64.decodeBase64(secret);
        Date expirationDate = new Date(System.currentTimeMillis() + 5000 * 60 * 60);
        Claims claims = Jwts.claims().setSubject(String.valueOf(userPrincipal.getId()));
        claims.put("NAME", userPrincipal.getUsername());
        claims.put("SCOPES", userPrincipal.getRole());

        SecretKey key = new SecretKeySpec(encodedKey, 0, encodedKey.length, "AES");
        String tokenString = Jwts
            .builder()
            .setClaims(claims)
            .setIssuedAt(new Date())
            .setExpiration(expirationDate)
            .signWith(SignatureAlgorithm.HS512, key)
            .compact();
        updateResponseHeader(tokenString);
        return tokenString;
    }

    private void updateResponseHeader(String tokenString) {
        Optional.ofNullable((ServletRequestAttributes) getRequestAttributes())
            .map(ServletRequestAttributes::getResponse)
            .ifPresent(resp -> resp.setHeader(AUTHORIZATION, JwtAuthenticationFilter.TOKEN_HEADER + tokenString));
    }

    public JwtAuthenticationToken from(String jwtToken) {
        Claims claims = getClaimsFromToken(jwtToken);
        String role = (String) claims.get("SCOPES");
        List<GrantedAuthority> grantedAuthorities = new ArrayList<>();
        grantedAuthorities.add(new SimpleGrantedAuthority(role));
        UserPrincipal user = new UserPrincipal(Long.valueOf(claims.getSubject()), String.valueOf(claims.get("NAME")), role);
        JwtAuthenticationToken token = new JwtAuthenticationToken(user, null, grantedAuthorities);
        token.setAuthenticated(true);
        return token;
    }

    private Claims getClaimsFromToken(String token) {
        byte[] encodedKey = Base64.decodeBase64(secret);
        SecretKey key = new SecretKeySpec(encodedKey, 0, encodedKey.length, "AES");
        return Jwts
            .parser()
            .setSigningKey(key)
            .parseClaimsJws(token)
            .getBody();
    }
}
```

```
public class JwtAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    public static final String AUTHORIZATION = "Authorization";
    public static final String TOKEN_HEADER = "Bearer ";


    public JwtAuthenticationFilter(AuthenticationManager authenticationManager) {
        super("/**");
        setAuthenticationManager(authenticationManager);
        setAuthenticationFailureHandler(new JwtAuthenticationFailureHandler(new Dcsp2ObjectMapper()));
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException {
        String tokenString = extractToken(request);

        return getAuthenticationManager().authenticate(new JwtAuthenticationToken(null, JwtToken.of(tokenString), null));
    }

    private String extractToken(HttpServletRequest request) {
        String authorizationString = request.getHeader(AUTHORIZATION);

        if (authorizationString == null || !authorizationString.startsWith(TOKEN_HEADER)) {
            throw new BadCredentialsException("Authorization header missing or with invalid format.");
        }

        return authorizationString.substring(TOKEN_HEADER.length(), authorizationString.length());
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
                                            Authentication authentication) throws IOException, ServletException {
        SecurityContext context = createEmptyContext();
        context.setAuthentication(authentication);
        SecurityContextHolder.setContext(context);
        chain.doFilter(request, response);
    }

    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                              AuthenticationException exception) throws IOException, ServletException {
        getFailureHandler().onAuthenticationFailure(request, response, exception);
    }

}
```

```
@Component
public class OperationJwtAuthenticationProvider implements AuthenticationProvider {

    private OperationJwtService operationJwtService;

    public OperationJwtAuthenticationProvider(OperationJwtService operationJwtService) {
        this.operationJwtService = operationJwtService;
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        JwtAuthenticationToken authenticationToken = (JwtAuthenticationToken) authentication;
        try {
            return operationJwtService.from(authenticationToken.getJwtToken());
        } catch (Exception e) {
            throw new BadCredentialsException("登录信息失效.", e);
        }
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return JwtAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```