# CH29 스프링부트 RestAPI 서버 CSR 최종정리

> 최종 프로젝트를 위한 세팅 프로젝트 공유
>
> https://github.com/codingspecialist/Springboot-RestAPI-Security-JWT-End.git

## 1. 소켓통신과 DispatcherServlet을 모르면 아무것도 할 수 없다.

> - 소켓통신
>
> https://github.com/codingspecialist/socket-study.git
>
> - 서블릿
>
> https://github.com/codingspecialist/web1.git
>
> https://github.com/codingspecialist/web2.git
>
> - 모델 1
>
> https://github.com/codingspecialist/web3.git
>
> - FrontController 패턴
>
> https://github.com/codingspecialist/web4.git
>
> https://github.com/codingspecialist/web5.git
>
> - MVC 패턴
>
> https://github.com/codingspecialist/mvcapp

## 2. 디자인패턴 공부 권장

https://www.youtube.com/@user-pw9fm4gc7e

## 3. Controller, Service, Repository의 책임

> 책임에 대해서 정확히 알지 못하면, 디버깅하기 어렵다.

## 4. Solid에 대해서 정리 권장

> SRP
>
> DIP
>
> OCP
>
> 위 3가지만 정리해도 충분하다.

## 5. Lazy 로딩을 사용 권장

> Eager과 Lazy가 섞여 있으면, 혼동스럽다. 모든 것을 Lazy로 적용하고, DTO에서 필요한 것만 지연로딩하자.
> 만약 성능이 잘 안나오는 것 같으면 fetch join을 사용하자.
> 양방향 매핑을 만약에 사용한다면, 내부에 컬렉션에서 N+1 발생하게 되는데, default_batch_fetch_size 사용하자

## 6. OSIV를 비활성화 권장

> OSIV가 활성화 되어있으면 Controller 레이어에서 지연로딩을 하게 되는데, OSIV를 비활성화하고 서비스 레이어에서 DTO로 지연로딩을 완료하는 것을 추천한다.

## 7. 양방향 매핑보다는 쿼리를 여러번 호출하는 것을 권장

> 양방향을 설정하는 것보다, 단방향으로 설정한 뒤, 쿼리를 여러번 호출하는 것이 관리에 용이하다.
>
> 모든 것들은 primitive(원시적인)하게 만들어서, 조합해서 사용하는 것을 추천한다.
>
> 그리고 연관관계 편의 메서드를 만들 필요도 없다.

## 8. default_batch_fetch_size 사용 권장

> 굳이 양방향 매핑을 사용하겠다면 해당 설정을 꼭 해주자.

```yaml
  jpa:
    hibernate:
      ddl-auto: create
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
```

## 9. Join Fetch 사용 권장

> 필요한 데이터는 미리 가져와야 두번 select 하지 않는다.

## 10. 레이어별 테스트 권장

> #### 컨트롤러
>
> - @WebMvcTest
>   - Filter, DS, Controller 까지만 띄움
> - @Sql("classpath:db/teardown.sql") 로 table truncate 활용
> - @Transactional  어노테이션 권장하지 않음
> - @EnableAspectJAutoProxy
> 
> #### 서비스
> 
> - @ExtendWith(MockitoExtension.class)
>   - MockBean 환경만 띄운것
> - @InjectMocks
>   - 진짜 객체 만들어서 MockBean 주입
>   - 객체가 의존하고 있는 것 주입해야함
>     - @Mock
>     - @Spy
> - @Mock
>   - 가짜(껍데기) new
>   - 외부에 @Mock로 존재하는 가짜 객체가 new 되어서, @InjectMocks 객체로 주입된다.
> - @Spy
>   - 진짜 new 
>   - MockBean 에 @Spy로 존재하는 진짜 객체가 new 되어서, @InjectMocks 객체로 주입된다.
> 
> #### 레파지토리
> 
> - @DataJpaTest
>   - Transaction 존재 
>   - @BCrytPasswordEncoder 뜨지않음
>     - 왜냐하면 @DataJpaTest 범위는 Repository만 띄우기 때문에
> - @Import 활용
> - auto_increment 초기화 활용
> - DB 끝에는 em.clear()

### (1) 컨트롤러

#### core

- test.../core/MyWithMockUser.java

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = MyWithMockUserFactory.class)
public @interface MyWithMockUser {
    long id() default 1L;
    String username() default "cos";
    String role() default "USER";
    String fullName() default "코스";
}
```

- test.../core/MyWithMockUserFactory.java

```java
public class MyWithMockUserFactory implements WithSecurityContextFactory<MyWithMockUser> {
    @Override
    public SecurityContext createSecurityContext(MyWithMockUser mockUser) {
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        User user = User.builder()
                .id(mockUser.id())
                .username(mockUser.username())
                .password("1234")
                .email(mockUser.username()+"@nate.com")
                .fullName(mockUser.fullName())
                .role("USER")
                .status(true)
                .createdAt(LocalDateTime.now())
                .build();
        MyUserDetails myUserDetails = new MyUserDetails(user);
        Authentication auth = new UsernamePasswordAuthenticationToken(myUserDetails, myUserDetails.getPassword(), myUserDetails.getAuthorities());
        context.setAuthentication(auth);
        return context;
    }
}
```

#### 테스트코드

```java
@ActiveProfiles("test")
@EnableAspectJAutoProxy // AOP 활성화
@Import({
        MyValidAdvice.class, // 유효성 검사
        MyLogAdvice.class,
        MySecurityConfig.class,
        MyFilterRegisterConfig.class
}) // Advice 와 Security 설정 가져오기
@WebMvcTest(
        // 원하는 Controller 가져오기, 특정 필터를 제외하기
        controllers = {UserController.class} // Filter, DS, Controller 만 뜬다. 나머지는 뜨지 않는다. 가볍게 테스트하기 위해서
)
public class UserControllerUnitTest extends DummyEntity {
    @Autowired
    private MockMvc mvc;
    @Autowired
    private ObjectMapper om;
    @MockBean // 가짜 서버 -> 껍데기만 만든다
    private UserService userService;

    @Test
    public void join_test() throws Exception {
        // 준비
        UserRequest.JoinInDTO joinInDTO = new UserRequest.JoinInDTO();
        joinInDTO.setUsername("cos");
        joinInDTO.setPassword("1234");
        joinInDTO.setEmail("cos@nate.com");
        joinInDTO.setFullName("코스");
        String requestBody = om.writeValueAsString(joinInDTO);

        // 가정해볼께
        User cos = newMockUser(1L,"cos", "코스"); // 아이디가 필요하기 때문에
        UserResponse.JoinOutDTO joinOutDTO = new UserResponse.JoinOutDTO(cos);

        Mockito.when(userService.회원가입(any())).thenReturn(joinOutDTO);

        // 테스트진행
        ResultActions resultActions = mvc
                .perform(post("/join").content(requestBody).contentType(MediaType.APPLICATION_JSON));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // 검증해볼께
        resultActions.andExpect(jsonPath("$.data.id").value(1L));
        resultActions.andExpect(jsonPath("$.data.username").value("cos"));
        resultActions.andExpect(jsonPath("$.data.fullName").value("코스"));
        resultActions.andExpect(status().isOk());
    }

    @Test
    public void login_test() throws Exception {
        // given
        UserRequest.LoginInDTO loginInDTO = new UserRequest.LoginInDTO();
        loginInDTO.setUsername("cos");
        loginInDTO.setPassword("1234");
        String requestBody = om.writeValueAsString(loginInDTO);

        // stub
        Mockito.when(userService.로그인(any())).thenReturn("Bearer 1234");

        // when
        ResultActions resultActions = mvc
                .perform(post("/login").content(requestBody).contentType(MediaType.APPLICATION_JSON));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        String jwtToken = resultActions.andReturn().getResponse().getHeader(MyJwtProvider.HEADER);
        Assertions.assertThat(jwtToken.startsWith(MyJwtProvider.TOKEN_PREFIX)).isTrue();
        resultActions.andExpect(status().isOk());
    }

    // 중요!!
    @MyWithMockUser(id = 1L, username = "cos", role = "USER", fullName = "코스") // 내부에 getUser 들고 있다
    //@WithMockUser(value = "ssar", password = "1234", roles = "USER") user객체가 없다 (id 가 없음)
    @Test
    public void detail_test() throws Exception {
        // given
        Long id = 1L;

        // stub
        User cos = newMockUser(1L,"cos", "코스");
        UserResponse.DetailOutDTO detailOutDTO = new UserResponse.DetailOutDTO(cos);
        Mockito.when(userService.회원상세보기(any())).thenReturn(detailOutDTO);

        // when
        ResultActions resultActions = mvc
                .perform(get("/s/user/"+id));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.data.id").value(1L));
        resultActions.andExpect(jsonPath("$.data.username").value("cos"));
        resultActions.andExpect(jsonPath("$.data.email").value("cos@nate.com"));
        resultActions.andExpect(jsonPath("$.data.fullName").value("코스"));
        resultActions.andExpect(jsonPath("$.data.role").value("USER"));
        resultActions.andExpect(status().isOk());
    }
}
```

### (2) 서비스

```java
@ActiveProfiles("test")
@ExtendWith(MockitoExtension.class) // 이것만 띄우기
public class UserServiceTest extends DummyEntity {

    // 진짜 userService 객체를 만들고 Mock로 Load된 모든 객체를 userService에 주입
    @InjectMocks
    private UserService userService;

    // 진짜 객체를 만들어서 Mockito 환경에 Load(userService에 주입)
    @Mock
    private UserRepository userRepository;

    // 가짜 객체를 만들어서 Mockito 환경에 Load(userService에 주입)
    @Mock
    private AuthenticationManager authenticationManager;

    // 진짜 객체를 만들어서 Mockito 환경에 Load
    @Spy
    private BCryptPasswordEncoder bCryptPasswordEncoder;

    @Test
    public void hello_test(){
        String pw = "1234";
        String enc = bCryptPasswordEncoder.encode(pw);
        System.out.println(enc);
    }

    @Test
    public void 회원가입_test() throws Exception{

        // given
        UserRequest.JoinInDTO joinInDTO = new UserRequest.JoinInDTO();
        joinInDTO.setUsername("cos");
        joinInDTO.setPassword("1234");
        joinInDTO.setEmail("cos@nate.com");
        joinInDTO.setFullName("코스");

        // stub 1
        Mockito.when(userRepository.findByUsername(any())).thenReturn(Optional.empty());

        // stub 2
        User cos = newMockUser(1L, "cos", "코스");
        Mockito.when(userRepository.save(any())).thenReturn(cos);

        // when
        UserResponse.JoinOutDTO joinOutDTO = userService.회원가입(joinInDTO);

        // then
        Assertions.assertThat(joinOutDTO.getId()).isEqualTo(1L);
        Assertions.assertThat(joinOutDTO.getUsername()).isEqualTo("cos");
        Assertions.assertThat(joinOutDTO.getFullName()).isEqualTo("코스");
    }

    @Test
    public void 로그인_test() throws Exception{
        // given
        UserRequest.LoginInDTO loginInDTO = new UserRequest.LoginInDTO();
        loginInDTO.setUsername("cos");
        loginInDTO.setPassword("1234");

        // stub
        User cos = newMockUser(1L, "cos", "코스");
        MyUserDetails myUserDetails = new MyUserDetails(cos);
        Authentication authentication = new UsernamePasswordAuthenticationToken(
                myUserDetails, myUserDetails.getPassword(), myUserDetails.getAuthorities()
        );
        Mockito.when(authenticationManager.authenticate(any())).thenReturn(authentication);

        // when
        String jwt = userService.로그인(loginInDTO);
        System.out.println("디버그 : "+jwt);

        // then
        Assertions.assertThat(jwt.startsWith("Bearer ")).isTrue();
    }

    @Test
    public void 유저상세보기_test() throws Exception{
        // given
        Long id = 1L;

        // stub
        User cos = newMockUser(1L, "cos", "코스");
        Mockito.when(userRepository.findById(any())).thenReturn(Optional.of(cos));

        // when
        UserResponse.DetailOutDTO detailOutDTO  = userService.회원상세보기(id);

        // then
        Assertions.assertThat(detailOutDTO.getId()).isEqualTo(1L);
        Assertions.assertThat(detailOutDTO.getUsername()).isEqualTo("cos");
        Assertions.assertThat(detailOutDTO.getEmail()).isEqualTo("cos@nate.com");
        Assertions.assertThat(detailOutDTO.getFullName()).isEqualTo(LocalDateTime.now());
        Assertions.assertThat(detailOutDTO.getRole()).isEqualTo("USER");
    }
}
```

### (3) 레포지토리

```java
@Import(BCryptPasswordEncoder.class) // BCryptPasswordEncoder 안뜨기 때문에
@ActiveProfiles("test")
@DataJpaTest // 내부에 트랜젝션이 존재 -> 롤백 -> auto_increment는 안돈다 -> setUp
public class UserRepositoryTest extends DummyEntity {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private EntityManager em;
    @Autowired
    private BCryptPasswordEncoder passwordEncoder;

    @BeforeEach
    public void setUp() {
        em.createNativeQuery("ALTER TABLE user_tb ALTER COLUMN `id` RESTART WITH 1").executeUpdate();
        userRepository.save(newUser("ssar", "쌀"));
        userRepository.save(newUser("cos", "코스"));
        em.clear(); // 무조건
    }

    @Test
    public void findById() {
        // given
        Long id = 1L;

        // when
        Optional<User> userOP = userRepository.findById(id);
        if (userOP.isEmpty()) {
            throw new Exception400("id", "아이디를 찾을 수 없습니다");
        }
        User userPS = userOP.get();

        // then
        Assertions.assertThat(userPS.getId()).isEqualTo(1L);
        Assertions.assertThat(userPS.getUsername()).isEqualTo("ssar");
        Assertions.assertThat(
                passwordEncoder.matches("1234", userPS.getPassword())
        ).isEqualTo(true);
        Assertions.assertThat(userPS.getEmail()).isEqualTo("ssar@nate.com");
        Assertions.assertThat(userPS.getFullName()).isEqualTo("쌀");
        Assertions.assertThat(userPS.getRole()).isEqualTo("USER");
        Assertions.assertThat(userPS.getStatus()).isEqualTo(true);
        Assertions.assertThat(userPS.getCreatedAt().toLocalDate()).isEqualTo(LocalDate.now());
        Assertions.assertThat(userPS.getUpdatedAt()).isNull();
    }

    @Test
    public void findByUsername() {
        // given
        String username = "ssar";

        // when
        Optional<User> userOP = userRepository.findByUsername(username);
        if (userOP.isEmpty()) {
            throw new Exception400("username", "유저네임을 찾을 수 없습니다");
        }
        User userPS = userOP.get();

        // then
        Assertions.assertThat(userPS.getId()).isEqualTo(1L);
        Assertions.assertThat(userPS.getUsername()).isEqualTo("ssar");
        Assertions.assertThat(
                passwordEncoder.matches("1234", userPS.getPassword())
        ).isEqualTo(true);
        Assertions.assertThat(userPS.getEmail()).isEqualTo("ssar@nate.com");
        Assertions.assertThat(userPS.getFullName()).isEqualTo("쌀");
        Assertions.assertThat(userPS.getRole()).isEqualTo("USER");
        Assertions.assertThat(userPS.getStatus()).isEqualTo(true);
        Assertions.assertThat(userPS.getCreatedAt().toLocalDate()).isEqualTo(LocalDate.now());
        Assertions.assertThat(userPS.getUpdatedAt()).isNull();
    }

    @Test
    public void save() {
        // given
        User love = newUser("love", "러브");

        // when
        User userPS = userRepository.save(love);

        // then (beforeEach에서 2건이 insert 되어 있음)
        Assertions.assertThat(userPS.getId()).isEqualTo(3L);
        Assertions.assertThat(userPS.getUsername()).isEqualTo("love");
        Assertions.assertThat(
                passwordEncoder.matches("1234", userPS.getPassword())
        ).isEqualTo(true);
        Assertions.assertThat(userPS.getEmail()).isEqualTo("love@nate.com");
        Assertions.assertThat(userPS.getFullName()).isEqualTo("러브");
        Assertions.assertThat(userPS.getRole()).isEqualTo("USER");
        Assertions.assertThat(userPS.getStatus()).isEqualTo(true);
        Assertions.assertThat(userPS.getCreatedAt().toLocalDate()).isEqualTo(LocalDate.now());
        Assertions.assertThat(userPS.getUpdatedAt()).isNull();
    }
}
```

## 11. 단위 테스트가 끝나면 통합 테스트 권장

> @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
>
> - @AutoConfigureMockMvc
> - @Sql("classpath:db/teardown.sql") 로 table truncate 활용
>   - SpringBootTest에는 Transaction 존재 X
> - @Transactional  어노테이션 권장하지 않음

- resources/db/teardown.sql

```sql
-- 모든 제약 조건 비활성화
SET REFERENTIAL_INTEGRITY FALSE;
truncate table user_tb; -- 내용만 날림
SET REFERENTIAL_INTEGRITY TRUE;
-- 모든 제약 조건 활성화
```

- User 통합테스트

```java
// 통합 테스트는 해야한다! -> 모든게 다 뜬다 SpringBootTest -> 메소드 실행할 떄마다 초기화가 안된다
@DisplayName("회원 API")
@AutoConfigureRestDocs(uriScheme = "http", uriHost = "localhost", uriPort = 8080)
@ActiveProfiles("test")
@Sql("classpath:db/teardown.sql")
@AutoConfigureMockMvc
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
public class UserControllerTest extends MyRestDoc {

    private DummyEntity dummy = new DummyEntity();

    @Autowired
    private MockMvc mvc;
    @Autowired
    private ObjectMapper om;
    @Autowired
    private UserRepository userRepository;
    @Autowired
    private EntityManager em;

    @BeforeEach
    public void setUp() {
        userRepository.save(dummy.newUser("ssar", "쌀"));
        userRepository.save(dummy.newUser("cos", "코스"));
        em.clear();
    }

    @DisplayName("회원가입 성공")
    @Test
    public void join_test() throws Exception {
        // given
        UserRequest.JoinInDTO joinInDTO = new UserRequest.JoinInDTO();
        joinInDTO.setUsername("love");
        joinInDTO.setPassword("1234");
        joinInDTO.setEmail("love@nate.com");
        joinInDTO.setFullName("러브");
        String requestBody = om.writeValueAsString(joinInDTO);

        // when
        ResultActions resultActions = mvc
                .perform(post("/join").content(requestBody).contentType(MediaType.APPLICATION_JSON));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.data.id").value(3L));
        resultActions.andExpect(jsonPath("$.data.username").value("love"));
        resultActions.andExpect(jsonPath("$.data.fullName").value("러브"));
        resultActions.andExpect(status().isOk());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }

    @DisplayName("회원가입 실패")
    @Test
    public void join_fail_bad_request_test() throws Exception {
        // given
        UserRequest.JoinInDTO joinInDTO = new UserRequest.JoinInDTO();
        joinInDTO.setUsername("ssar");
        joinInDTO.setPassword("1234");
        joinInDTO.setEmail("ssar@nate.com");
        joinInDTO.setFullName("쌀");
        String requestBody = om.writeValueAsString(joinInDTO);

        // when
        ResultActions resultActions = mvc
                .perform(post("/join").content(requestBody).contentType(MediaType.APPLICATION_JSON));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.status").value(400));
        resultActions.andExpect(jsonPath("$.msg").value("badRequest"));
        resultActions.andExpect(jsonPath("$.data.key").value("username"));
        resultActions.andExpect(jsonPath("$.data.value").value("유저네임이 존재합니다"));
        resultActions.andExpect(status().isBadRequest());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }

    @DisplayName("회원가입 유효성 검사 실패")
    @Test
    public void join_fail_valid_test() throws Exception {
        // given
        UserRequest.JoinInDTO joinInDTO = new UserRequest.JoinInDTO();
        joinInDTO.setUsername("s");
        joinInDTO.setPassword("1234");
        joinInDTO.setEmail("ssar@nate.com");
        joinInDTO.setFullName("쌀");
        String requestBody = om.writeValueAsString(joinInDTO);

        // when
        ResultActions resultActions = mvc
                .perform(post("/join").content(requestBody).contentType(MediaType.APPLICATION_JSON));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.status").value(400));
        resultActions.andExpect(jsonPath("$.msg").value("badRequest"));
        resultActions.andExpect(jsonPath("$.data.key").value("username"));
        resultActions.andExpect(jsonPath("$.data.value").value("영문/숫자 2~20자 이내로 작성해주세요"));
        resultActions.andExpect(status().isBadRequest());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }

    @DisplayName("로그인 성공")
    @Test
    public void login_test() throws Exception {
        // given
        UserRequest.LoginInDTO loginInDTO = new UserRequest.LoginInDTO();
        loginInDTO.setUsername("ssar");
        loginInDTO.setPassword("1234");
        String requestBody = om.writeValueAsString(loginInDTO);

        // when
        ResultActions resultActions = mvc
                .perform(post("/login").content(requestBody).contentType(MediaType.APPLICATION_JSON));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        String jwtToken = resultActions.andReturn().getResponse().getHeader(MyJwtProvider.HEADER);
        Assertions.assertThat(jwtToken.startsWith(MyJwtProvider.TOKEN_PREFIX)).isTrue();
        resultActions.andExpect(status().isOk());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }

    @DisplayName("로그인 인증 실패")
    @Test
    public void login_fail_un_authorized_test() throws Exception {
        // given
        UserRequest.LoginInDTO loginInDTO = new UserRequest.LoginInDTO();
        loginInDTO.setUsername("ssar");
        loginInDTO.setPassword("12345");
        String requestBody = om.writeValueAsString(loginInDTO);

        // when
        ResultActions resultActions = mvc
                .perform(post("/login").content(requestBody).contentType(MediaType.APPLICATION_JSON));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.status").value(401));
        resultActions.andExpect(jsonPath("$.msg").value("unAuthorized"));
        resultActions.andExpect(jsonPath("$.data").value("인증되지 않았습니다"));
        resultActions.andExpect(status().isUnauthorized());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }

    // jwt token -> 인증필터 -> 시큐리티 세션생성
    // setupBefore=TEST_METHOD (setUp 메서드 실행전에 수행)
    // setupBefore=TEST_EXECUTION (saveAccount_test 메서드 실행전에 수행)
    // @WithUserDetails(value = "ssar", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    // authenticationManager.authenticate() 실행해서 MyUserDetailsService를 호출하고
    // usrename=ssar을 찾아서 세션에 담아주는 어노테이션
    @DisplayName("회원상세보기 성공")
    @WithUserDetails(value = "ssar", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @Test
    public void detail_test() throws Exception {
        // given
        Long id = 1L;

        // when
        ResultActions resultActions = mvc
                .perform(get("/s/user/"+id));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.data.id").value(1L));
        resultActions.andExpect(jsonPath("$.data.username").value("ssar"));
        resultActions.andExpect(jsonPath("$.data.email").value("ssar@nate.com"));
        resultActions.andExpect(jsonPath("$.data.fullName").value("쌀"));
        resultActions.andExpect(jsonPath("$.data.role").value("USER"));
        resultActions.andExpect(status().isOk());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }

    @DisplayName("회원상세보기 인증 실패")
    @Test
    public void detail_fail_un_authorized__test() throws Exception {
        // given
        Long id = 1L;

        // when
        ResultActions resultActions = mvc
                .perform(get("/s/user/"+id));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.status").value(401));
        resultActions.andExpect(jsonPath("$.msg").value("unAuthorized"));
        resultActions.andExpect(jsonPath("$.data").value("인증되지 않았습니다"));
        resultActions.andExpect(status().isUnauthorized());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }

    @DisplayName("회원상세보기 권한 실패")
    @WithUserDetails(value = "cos", setupBefore = TestExecutionEvent.TEST_EXECUTION)
    @Test
    public void detail_fail_forbidden_test() throws Exception {
        // given
        Long id = 1L;

        // when
        ResultActions resultActions = mvc
                .perform(get("/s/user/"+id));
        String responseBody = resultActions.andReturn().getResponse().getContentAsString();
        System.out.println("테스트 : " + responseBody);

        // then
        resultActions.andExpect(jsonPath("$.status").value(403));
        resultActions.andExpect(jsonPath("$.msg").value("forbidden"));
        resultActions.andExpect(jsonPath("$.data").value("권한이 없습니다"));
        resultActions.andExpect(status().isForbidden());
        resultActions.andDo(MockMvcResultHandlers.print()).andDo(document); // RestDoc을 위한 추가
    }
}
```

## 12. 커스텀 어노테이션을 만들수 없다면 리플렉션 공부 권장

https://github.com/codingspecialist/java-reflection

## 13. AOP 사용 권장

https://github.com/codingspecialist/Springboot-AOP

## 14. 자바스크립트 공부 권장

> 강의가 너무 많아서 충분히 찾아서 공부할 수 있음.

## 15. AWS 클라우드 CI/CD 배포 공부 권장

http://www.yes24.com/Product/Goods/117628175

> 이것이 기본이 되어야 Docker, Kubernetes 를 공부할 수 있다.

## 16. WebFlux 공부 권장

https://www.youtube.com/watch?v=Oo_eHCr_HsQ&list=PL93mKxaRDidHRYNYYFr1x3mLKIx1wFeFc&index=1

## 17. JWT의 Refresh Token 구현 공부 권장

> - 중복 로그인을 허용하지 않을 때 프로세스는 어떻게 될까? (보안 고려해서)
> - 중복 로그인을 허용할 때 프로세스는 어떻게 될까? (보안 고려해서)
>   (아이패드, 컴퓨터, 노트북, 회사컴퓨터, 휴대폰)
> - 서로 다른 디바이스에서 중복 로그인이 있을 때마다 유저에게 알림을 줄 수 있다. 유저는 해당 알림에 대해서 수긍할 수 있지만, 수긍하지 않을 수 있다. 수긍하지 않을 때, 블랙리스트는 어떻게 관리해야할까? 그리고 앞으로 블랙리스트는 스프링에서 어느 레이어에서 막아야 할까?
> - CSRF 공부하기
> - Redis 공부하기
> - Https 서버 만들어보기

## 18. 시큐리티 세션 주입해서 테스트 하는 법

https://github.com/codingspecialist/junit-bank-class

```java
@BeforeEach
public void setUp() {
    User ssar = userRepository.save(newUser("ssar", "쌀"));
}

```

```java
// jwt token -> 인증필터 -> 시큐리티 세션생성
// setupBefore=TEST_METHOD (setUp 메서드 실행전에 수행)
// setupBefore=TEST_EXECUTION (saveAccount_test 메서드 실행전에 수행)
// 디비에서 username=ssar 조회를 해서 세션에 담아주는 어노테이션!!
@WithUserDetails(value = "ssar", setupBefore = TestExecutionEvent.TEST_EXECUTION) 
@Test
public void saveAccount_test() throws Exception {
	// given


   // when


   // then
   resultActions.andExpect(status().isCreated());
}
```