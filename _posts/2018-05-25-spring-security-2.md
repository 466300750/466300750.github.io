---
layout: post
title: spring security 2
date: 2018-09-14 14:48:31
categories: spring
share: y
excerpt_separator: <!--more-->
---

<!--more-->

### demo-2

```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    MyUserDetailService myUserDetailService;

    @Autowired
    private JwtAuthenticationProvider jwtAuthenticationProvider;

    @Autowired
    private JwtAuthenticationEntryPoint unauthorizedHandler;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf()
            .disable()
            .authorizeRequests()
            .antMatchers("/login").permitAll()
            .anyRequest()
            .authenticated()
            .and()
            .addFilterBefore(new JwtAuthenticationFilter(this.authenticationManager()), UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling()
            .authenticationEntryPoint(unauthorizedHandler)
            .and()
            .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
            .authenticationProvider(jwtAuthenticationProvider)
            .userDetailsService(myUserDetailService)
            .passwordEncoder(passwordEncoder());
    }

    @Bean(name = BeanIds.AUTHENTICATION_MANAGER)
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```
public class JwtAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    public static final String AUTHORIZATION = "Authorization";
    public static final String TOKEN_HEADER = "Bearer ";


    public JwtAuthenticationFilter(AuthenticationManager authenticationManager) {
        super(new NegatedRequestMatcher(new AntPathRequestMatcher("/login", "POST")));
        setAuthenticationManager(authenticationManager);
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException {
        String tokenString = extractToken(request);
        return getAuthenticationManager().authenticate(new JwtAuthenticationToken(null, tokenString, null));
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
public class JwtAuthenticationProvider implements AuthenticationProvider {

    private JwtTokenService jwtTokenService;

    public JwtAuthenticationProvider(JwtTokenService jwtTokenService) {
        this.jwtTokenService = jwtTokenService;
    }

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        JwtAuthenticationToken authenticationToken = (JwtAuthenticationToken) authentication;
        try {
            return jwtTokenService.from(authenticationToken.getJwtToken());
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

```
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    private static final Logger logger = LoggerFactory.getLogger(JwtAuthenticationEntryPoint.class);
    @Override
    public void commence(HttpServletRequest httpServletRequest,
                         HttpServletResponse httpServletResponse,
                         AuthenticationException e) throws IOException, ServletException {
        logger.error("Responding with unauthorized error. Message - {}", e.getMessage());
        httpServletResponse.sendError(HttpServletResponse.SC_UNAUTHORIZED,
            "Sorry, You're not authorized to access this resource.");
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
@Service
public class MyUserDetailService implements UserDetailsService {

    private UserRepository userRepository;

    public MyUserDetailService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    @Transactional
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByAccount(username);
        if (user == null) {
            throw new UsernameNotFoundException("UsernameNotFoundException");
        } else {
            return new UserPrincipal(user.getId(), user.getAccount(), user.getPassword(), user.getRoles().get(0).getName(), user.getRoles().
                    stream().map((Role role) -> new SimpleGrantedAuthority(role.getName())).collect(Collectors.toList()));
        }
    }
}
```

```
public class UserPrincipal implements UserDetails {

    private long id;
    private String account;
    private String password;
    private String role;
    private Collection<? extends GrantedAuthority> authorities;


    public UserPrincipal(long id, String account, String role) {
        this.id = id;
        this.account = account;
        this.role = role;
    }

    public UserPrincipal(long id, String account, String password, String role, Collection<? extends GrantedAuthority> authorities) {
        this.id = id;
        this.account = account;
        this.password = password;
        this.role = role;
        this.authorities = authorities;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return authorities;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.account;
    }

    public long getId() {
        return id;
    }

    public String getAccount() {
        return account;
    }

    public String getRole() {
        return role;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

```
@RestController
public class LoginController {

    private LoginApplicationService loginApplicationService;

    public LoginController(LoginApplicationService loginApplicationService) {
        this.loginApplicationService = loginApplicationService;
    }

    @PostMapping(value = "/login")
    public String login(@RequestBody UsernamePasswordLoginCommand command) {
        return loginApplicationService.authenticateUser(command);
    }
}
```

```
@Service
public class LoginApplicationService {
    private AuthenticationManager authenticationManager;
    private JwtTokenService jwtTokenService;

    public LoginApplicationService(AuthenticationManager authenticationManager, JwtTokenService jwtTokenService) {
        this.authenticationManager = authenticationManager;
        this.jwtTokenService = jwtTokenService;
    }

    public String authenticateUser(UsernamePasswordLoginCommand command) {
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(command.getUsername(), command.getPassword());
        Authentication authentication = authenticationManager.authenticate(token);
        SecurityContextHolder.getContext().setAuthentication(authentication);
        return jwtTokenService.generateToken((UserPrincipal) authentication.getPrincipal());
    }
}
```