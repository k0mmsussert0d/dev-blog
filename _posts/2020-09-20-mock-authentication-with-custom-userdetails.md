---
layout: post
title: "Mocking @AuthenticationPrincipal with custom UserDetails object"
date: 2020-09-20 19:00:00
categories:
    - blog
tags:
    - java
    - junit
    - spring
    - mockito
---

Lately I've been working on a project involving Spring Boot and Spring Security. It includes stateless API for authenticating users and restricting access to specific HTTP endpoints. I followed this [Stateless Authentication with Spring Security\[0\]][0] article. Sessions associated with users are stored in the database table, while users authenticate with session ID in `Authorization` request header. Pretty much what JWT does, except for I didn't know about its existene at the moment, lol.

<!--break-->

**NOTE: This article follows my journey of debugging the problem, trying different wrong approaches and eventually finding the solution.
If it's tldr for you, jump straight to [the solution](#final-solution).**

Anyway, I took an advantage of `Authentication` storing principal object and created a class implementing `UserDetails` interface holding my DTO with mapped user entity:

```java
@RequiredArgsConstructor
public class CustomUserDetails implements UserDetails {

    @Getter
    private final UserDTO user;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        RoleDTO role = user.getRole();
        Set<GrantedAuthority> rolesSet = new HashSet<>();
        rolesSet.add(new SimpleGrantedAuthority("ROLE_" + role.getRoleName()));
        return rolesSet;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return user.getIsActive();
    }

    @Override
    public boolean isAccountNonLocked() {
        return user.getIsActive();
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return user.getIsActive();
    }

    @Override
    public boolean isEnabled() {
        return user.getIsActive();
    }
}
```

Essentially, a proxy class implementing `UserDetails` with a `UserDTO` composed in and using its properties for resolving `UserDetails` implemented methods. This approach allowed me to very easily access properties of a user accessing an endpoint. Consider this `GET /user/images/` private endpoint, which is supposed to return authenticated user's uploaded images:

```java
public ResponseEntity<List<ImageDTO>> userImages(@AuthenticationPrincipal CustomUserDetails userDetails) {
       UserDTO user = userDetails.getUser();

       Optional<List<ImageDTO>> images = Optional.ofNullable(imageDTOService.findAllUploadedBy(user));
       if (images.isEmpty()) {
           return ResponseEntity.status(HttpStatus.OK).body(Collections.emptyList());
       }

    return ResponseEntity.status(HttpStatus.OK).body(images.get());
} 
```

To me it looks nice and elegant. This custom `UserDetailsService` returns `CustomUserDetails` based on what persistence layer returns, so the encapsulated DTO object can always be used for interaction with other DTOs or persistence services.

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserDTOService userDTOService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        Optional<UserDTO> user = Optional.ofNullable(userDTOService.findByUsername(username));

        if (user.isEmpty()) {
            throw new UsernameNotFoundException("Username " + username + " not found");
        }

        return new CustomUserDetails(user.get());
    }
}
```

## How do I test this controller?

However, this approach implies some issues when it comes to testing the controller.

While trying to test `UserControllerImpl.userImages()` I realized I need to somehow mock the parameter `@AuthenticationPrincipal CustomUserDetails userDetails`. Not knowing much about it, I did some googling and found out, that [others\[1\]][1] use [@WithMockUser\[2\]][2] annotation in situations like that. However, this turned out unsatisfactionary for my case, since the method I want to test doesn't just restrict an access to some commonly-used resource. If I was, for example, testing the accessibility of resources that should only be used by specified group of users, that would be perfect. Using `@WithMockUser` I can specify roles and/or authorities required to access the resource, then test if access is granted (or properly rejected for mocked users with insufficient authorities). Or even a simpler case, having an endpoint returning user's username. Just a quick `@WithMockUser(username = "foo")` and a job is done!

Meanwhile, my resource uses the principal to pass the encapsulated user DTO to the service. It delegates the job to the persistence layer, which requires DTO object which can be later be mapped to full-fleged entity. Therefore, I needed full-fleged authentication principal mocking.

## Trying to mock the principal by interference in Security Context

What exactly is `@AuthenticationPrincipal`? According to the [Spring Security documentation\[3\]][3], it ties up a method argument with `Authentication.getPrincipal()` return object. `Authentication` object can be easily obtained from `SecurityContextHolder`, so using some help of this [stack overflow answer\[4\]][4] I came up with these few lines of code:

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerImplTest {

    @Test
    public void whenGetUserImages_andUserIsAuthorized_thenReturnListOfImages() throws Exception {
        String username = "username";
        UserDTO userDTO = getSampleUser(username);
        CustomUserDetails userDetails = new CustomUserDetails(userDTO);

        Authentication authentication = Mockito.mock(Authentication.class);
        SecurityContext securityContext = Mockito.mock(SecurityContext.class);
        AuthCookieFilter authCookieFilter = Mockito.mock(AuthCookieFilter.class);


        Mockito.when(securityContext.getAuthentication()).thenReturn(authentication);
        SecurityContextHolder.setContext(securityContext);

        Mockito.when(authentication.getPrincipal()).thenReturn(userDetails);

        Mockito.doNothing().when(authCookieFilter).doFilter(Mockito.any(ServletRequest.class), Mockito.any(ServletResponse.class), Mockito.any(FilterChain.class));

        List<ImageDTO> images = getSampleImages();
        images.forEach(imageDTO -> imageDTO.setUploader(userDTO));

        Mockito.when(imageDTOService.findAllUploadedBy(userDTO)).thenReturn(images);

        mockMvc.perform(get("/user/images/"))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(content().json("expected json response content goes here");
    }

    private UserDTO getSampleUser(String username) {
        return UserDTO.builder()
                .username(username)
                .email(username + "@example.com")
                .password("password")
                .registerTime(LocalDateTime.of(2020, 1, 1, 12, 0, 0, 0))
                .isActive(true)
                .build();
    }

    private List<ImageDTO> getSampleImages() {
        ImageDTO image1 = getSampleImage(1L, "token1");
        ImageDTO image2 = getSampleImage(2L, "token2");
        ImageDTO image3 = getSampleImage(3L, "token3");

        return List.of(image1, image2, image3);
    }

    private ImageDTO getSampleImage(Long id, String token) {
        return ImageDTO.builder()
                .id(id)
                .token(token)
                .isActive(true)
                .isPublic(true)
                .uploadTime(LocalDateTime.of(2020, 1, 1, 12, 0, 0, 0))
                .build();
    }
}
```

`AuthCookieFilter` is my request filter (`GenericFilterBean`), scanning requests for `Authorization` token, looking their content up in database and setting `Authentication` in `SecurityContextHolder` if it matches any. I had to silence it, since I'm setting up the Security Context up manually.

Unfortunately, it didn't work. For some reason, I couldn't get pass authorization, getting 401 responses all the time. After some time debugging, I found that one of the first culprits of failing the authentication is `ProviderManager.authenticate()` throwing a [ProviderNotFoundException\[5\]][5]. So, there's no `AuthenticationProvider` capable of handling my `Authentication`, huh? What's the `Authentication` now, anyway? According to the debugger: `Authentication$MockitoMock`. Honestly, I don't really know much about how mocks work internally. How are public properties and methods made visible, how references to them are being passed and most importantly, how is such an object perceived by other components of the application? My guess was that `ProviderManager` can't match any provider with my not-so-ordinary object.

And so, I decided to abadon this idea and keep looking further.

## Injecting custom UserDetailsService

Then I stumbled upon this [stack overflow answer\[6\]][6], that made me aware of [`@WithUserDetails`\[7\]][7] annotation. Similarly to `@WithMockUser` it allows to inject a mock user to the request, but delegating the job of creating `UserDetails` object to the developer. `@WithMockUser` is higher level functionality, creating a simple `UserDetails` based on input parameters. With `@WithUserDetails` one needs to provide own `UserDetailsService`.

Perfect! I created additional configuration class for such a custom service:
```java
@TestConfiguration
public class SpringSecurityForUserControllerImplTestConfig {

    @Bean
    @Primary
    public UserDetailsService userDetailsService() {

        UserDTO userDTO = UserDTO.builder()
                .username("username")
                .email("username@example.com")
                .password("password")
                .registerTime(LocalDateTime.of(2020, 1, 1, 12, 0, 0, 0))
                .isActive(true)
                .role(RoleDTO.builder().roleName("USER").build())
                .build();
        CustomUserDetails userDetails = new CustomUserDetails(userDTO);

        return new InMemoryUserDetailsManager(List.of(userDetails));
    }
}
```

added `@SpringBootTest(classes = SpringSecurityForUserControllerImplTestConfig.class)` above tests class and `@WithUserDetails("username")` above the test. To retrieve `UserDTO` object inside test method I used the following line:
```java
UserDTO userDTO = ((CustomUserDetails) SecurityContextHolder.getContext().getAuthentication().getPrincipal()).getUser();
```

which unfortunately resulted in ClassCastException between `org.springframework.security.core.userdetails.User` and my `CustomUserDetails`. It turns out, that `InMemoryUserDetailsManager` method has this method:
```java
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserDetails user = (UserDetails)this.users.get(username.toLowerCase());
        if (user == null) {
            throw new UsernameNotFoundException(username);
        } else {
            return new User(user.getUsername(), user.getPassword(), user.isEnabled(), user.isAccountNonExpired(), user.isCredentialsNonExpired(), user.isAccountNonLocked(), user.getAuthorities());
        }
    }
```

explictly casting found principals to Spring generic User class, despite my custom implementaion of UserDetails put into it. That's sad.

## Final solution
Here's test-scope class with custom `UserDetailsManager` implementation altering `InMemoryUserDetailsManager.loadUserByUsername` unwanted behavior. It could've been done better, this implementation assumes there is only one user object stored inside by the Manager. `UserDTO` was turned into a Bean, to allow easy access to it via autowiring from the test class.

```java
@TestConfiguration
public class SpringSecurityForUserControllerImplTestConfig {

    @Bean
    public UserDTO testUser() {
        return UserDTO.builder()
                .username("username")
                .email("username@example.com")
                .password("password")
                .registerTime(LocalDateTime.of(2020, 1, 1, 12, 0, 0, 0))
                .isActive(true)
                .role(RoleDTO.builder().roleName("USER").build())
                .build();
    }

    @Bean
    @Primary
    public UserDetailsService userDetailsService() {

        CustomUserDetails userDetails = new CustomUserDetails(testUser());

        return new UserDetailsManager() {

            private final InMemoryUserDetailsManager inMemoryUserDetailsManager = new InMemoryUserDetailsManager(List.of(userDetails));


            @Override
            public void createUser(UserDetails userDetails) {
                this.inMemoryUserDetailsManager.createUser(userDetails);
            }

            @Override
            public void updateUser(UserDetails userDetails) {
                this.inMemoryUserDetailsManager.updateUser(userDetails);
            }

            @Override
            public void deleteUser(String s) {
                this.inMemoryUserDetailsManager.deleteUser(s);
            }

            @Override
            public void changePassword(String s, String s1) {
                this.inMemoryUserDetailsManager.changePassword(s, s1);
            }

            @Override
            public boolean userExists(String s) {
                return this.inMemoryUserDetailsManager.userExists(s);
            }

            @Override
            public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
                return new CustomUserDetails(testUser());
            }
        };
    }
}
```

Test class now looks as following:
```java
@SpringBootTest(
        classes = SpringSecurityForUserControllerImplTestConfig.class
)
@AutoConfigureMockMvc
class UserControllerImplTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ImageDTOService imageDTOService;

    @Autowired
    UserDTO testUser;

    @Test
    @WithUserDetails("username")
    public void whenGetUserImages_andUserIsAuthorized_thenReturnListOfImages() throws Exception {

        List<ImageDTO> images = getSampleImages();
        images.forEach(imageDTO -> imageDTO.setUploader(testUser));

        Mockito.when(imageDTOService.findAllUploadedBy(testUser)).thenReturn(images);

        mockMvc.perform(get("/user/images/"))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(content().json(
                        "[\n" +
                                "  {\n" +
                                "    \"id\": 1,\n" +
                                "    \"isActive\": true,\n" +
                                "    \"isPublic\": true,\n" +
                                "    \"title\": null,\n" +
                                "    \"token\": \"token1\",\n" +
                                "    \"uploadTime\": \"2020-01-01T12:00:00\",\n" +
                                "    \"uploader\": \"username\"\n" +
                                "  },\n" +
                                "  {\n" +
                                "    \"id\": 2,\n" +
                                "    \"isActive\": true,\n" +
                                "    \"isPublic\": true,\n" +
                                "    \"title\": null,\n" +
                                "    \"token\": \"token2\",\n" +
                                "    \"uploadTime\": \"2020-01-01T12:00:00\",\n" +
                                "    \"uploader\": \"username\"\n" +
                                "  },\n" +
                                "  {\n" +
                                "    \"id\": 3," +
                                "    \n" +
                                "    \"isActive\":true,\n" +
                                "    \"isPublic\": true,\n" +
                                "    \"title\": null,\n" +
                                "    \"token\": \"token3\",\n" +
                                "    \"uploadTime\": \"2020-01-01T12:00:00\",\n" +
                                "    \"uploader\": \"username\"" +
                                "  }\n" +
                                "]"
                ));
    }
```

And it passes the test at last! 🎉🥰


## Links
~~~
[0]: https://golb.hplar.ch/2019/05/stateless.html
[1]: https://stackoverflow.com/questions/46615504/springboottest-mock-authentication-principal-with-a-custom-user-does-not-work
[2]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/test/context/support/WithMockUser.html
[3]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/core/annotation/AuthenticationPrincipal.html
[4]: https://stackoverflow.com/a/46631015/6158307
[5]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/authentication/ProviderNotFoundException.html
[6]: https://stackoverflow.com/a/43920932/6158307
[7]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/test/context/support/WithUserDetails.html
~~~

[0]: https://golb.hplar.ch/2019/05/stateless.html
[1]: https://stackoverflow.com/questions/46615504/springboottest-mock-authentication-principal-with-a-custom-user-does-not-work
[2]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/test/context/support/WithMockUser.html
[3]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/core/annotation/AuthenticationPrincipal.html
[4]: https://stackoverflow.com/a/46631015/6158307
[5]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/authentication/ProviderNotFoundException.html
[6]: https://stackoverflow.com/a/43920932/6158307
[7]: https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/test/context/support/WithUserDetails.html