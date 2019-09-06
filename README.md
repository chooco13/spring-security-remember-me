# spring-security-remember-me
스프링 시큐리티에서 기본적으로 제공하는 remember me 의 경우 계정당 한 기기에서만 가능하다는 문제점이 있었기에 직접 구현해 나간 과정을 기록

## 참고링크
1. 스프링 시큐리티의 Rememeber me 설명 (구버전 doc이긴 하나 큰 차이 보이지 않음)  
https://docs.spring.io/spring-security/site/docs/3.0.x/reference/remember-me.html

2. Rememeber me 동작원리 설명  
https://isme2n.github.io/devlog/2017/06/13/security-remember-me/


## 기존 스프링 시큐리티에 있는 Remember Me 의 문제점
Token을 저장하기 위한 테이블은 다음과 같은 구조를 갖는다. (참고링크 1 참고)  
```
create table persistent_logins (
    username varchar(64) not null,
    series varchar(64) primary key,
    token varchar(64) not null,
    last_used timestamp not null
)
``` 
여러 기기에서 로그인을 해야 할 때 이슈가 있었다.   
한쪽 에서 Remember me 를 이용하여 로그인 하거나, 다른쪽에서 로그아웃을 할 경우   
인증 TOKEN 데이터가 덮어씌워지거나, 삭제되어서  
기존의 기기는 Remember Me 가 무력화 되었다.  

## 기존 스프링 시큐리티에 있는 Remember Me 의 문제점
Remember me 의 동작원리를 참고하여 직접 구현하여 보았다. (참고링크 2 참고)  

### RememberMe 객체 만들기
먼저 나는 JPA 를 이용하여 구현하므로 RememberMe 객체를 만들어 주었다.  
기본의 구조에서 조금만 확장 하였다. 
JPA repository도 생성해준다.  
```
@Data
@Table(name = "REMEMBER_ME")
@Entity(name = "REMEMBER_ME")
public class RememberMe {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "REMEMBER_ME_ID")
    private int id;

    @Column(name = "USER_ID")
    private String userId;

    @Column(name = "REMEMBER_ME_SERIES", unique = true)
    private String series;

    @Column(name = "REMEMBER_ME_TOKEN")
    private String token;

    @Column(name = "CREATED_AT")
    private Date createdAt = new Date();

    @Column(name = "LAST_USED")
    private Date lastUsed = new Date();

    public RememberMe() {
    }

    public RememberMe(String userId, String series, String token) {
        this.userId = userId;
        this.series = series;
        this.token = token;
    }
}
```

### REMEMBER ME를 위한 유틸 만들기
큰 의미는 없다. 그냥 내 프로젝트의 기본 틀을 유지하기 위해 유틸을 만들었다.
```
public class RememberMeUtil {
    public static final String REMEMEBER_ME_COOKIE_NAME = "REMEMBER_ME";

    public static String getSeries() {
        return CMSUtil.randomString();
    }

    public static String getToken() {
        return CMSUtil.randomString();
    }

    public static String get(HttpServletRequest request) {
        return CookieUtil.getValue(request, REMEMEBER_ME_COOKIE_NAME);
    }
}
```
```
public class CookieUtil {
    public static void create(HttpServletResponse response, String name, String value, int maxAge) {
        Cookie cookie = new Cookie(name, value);
        if (System.getProperty("dev") == null) {
            cookie.setSecure(true);
            cookie.setDomain("yourdomain.com");
        } else {
            cookie.setDomain("127.0.0.1");
        }
        cookie.setHttpOnly(true);
        cookie.setMaxAge(maxAge);
        cookie.setPath("/");

        response.addCookie(cookie);
    }

    public static void clear(HttpServletResponse httpServletResponse, String name) {
        Cookie cookie = new Cookie(name, null);
        cookie.setPath("/");
        cookie.setHttpOnly(true);
        cookie.setMaxAge(0);
        httpServletResponse.addCookie(cookie);
    }

    public static String getValue(HttpServletRequest httpServletRequest, String name) {
        Cookie cookie = WebUtils.getCookie(httpServletRequest, name);
        return cookie != null ? cookie.getValue() : null;
    }
}
```

CMSUtil의 randomString 메소드는 다음과 같다. 별거 없다. 꼭 이게 아니라도 된다.
```
public static String randomString() {
    return UUID.randomUUID().toString().replace("-", "");
}
```

### 로그인에 REMEMBER ME 적용하기
로그인 폼에 다음과 같이 name 이 "rememeber-me" 인 checkbox를 두었다. 
```
<input type="checkbox" id="remember" name="remember-me" class="d-none">     
```

로그인 성공시 LoginSuccessHandler 의 onAuthenticationSuccess 에서 다음과 같이 처리를 해주었다.  
textEncryptor 는 spring에서 기본적으로 제공해주는 유틸이다.  
```
Map<String, String[]> parameters = request.getParameterMap();
if (parameters.get("remember-me") != null) {
    String series = RememberMeUtil.getSeries();
    while (rememberMeRepository.countBySeries(series) > 0) {
        series = RememberMeUtil.getSeries();
    }

    String rememberMeToken = RememberMeUtil.getToken();

    String key = textEncryptor.encrypt(String.format("[\"%s\",\"%s\"]", series, rememberMeToken));
    logger.debug("[RememberMe - key]::{}", key);
    CookieUtil.create(response, RememberMeUtil.REMEMEBER_ME_COOKIE_NAME, key, 365 * 24 * 60 * 60);

    rememberMeRepository.save(new RememberMe(userDetail.getUsername(), series, rememberMeToken));
}
```

로그인 페이지가 Mapping 되어있는 곳에 다음과 같이 추가하여 주었다.  
Rememeber me 쿠키가 있을 경우 valid한지 확인한 후
series는 유지하고 token은 갱신한 뒤 다시 암호화하여 쿠키에 저장한다.
쿠키가 invalid 할경우 쿠키를 클리어 해준다. 추후에 사용자한테 경고 메시지를 띄워주는 방식으로 해도 될 것 같다.  
```
String key = RememberMeUtil.get(request);
if (StringUtils.isNotEmpty(key)) {
    try {
        key = textEncryptor.decrypt(key);
        logger.debug("[REMEMBER_ME] key::{}", key);
        String[] seriesAndToken = new Gson().fromJson(key, String[].class);
        RememberMe rememberMe = rememberMeRepository.findBySeries(seriesAndToken[0]);
        if (rememberMe.getToken().equals(seriesAndToken[1])) {
            User user = userRepository.findById(rememberMe.getUserId().toLowerCase().replaceAll("\\s+", "")).orElse(null);
            if (user != null) {
                CMSUserDetail cmsUserDetail = new CMSUserDetail(user);
                Authentication auth = new UsernamePasswordAuthenticationToken(cmsUserDetail, null, cmsUserDetail.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(auth);

                String token = RememberMeUtil.getToken();
                key = textEncryptor.encrypt(String.format("[%s,%s]", seriesAndToken[0], token));
                logger.debug("[RememberMe - new key]::{}", key);
                CookieUtil.create(response, RememberMeUtil.REMEMEBER_ME_COOKIE_NAME, key, 365 * 24 * 60 * 60);

                rememberMe.setToken(token);
                rememberMe.setLastUsed(new Date());
                rememberMeRepository.save(rememberMe);
            } else {
                throw new Exception();
            }
        } else {
            throw new Exception();
        }
    } catch (Exception e) {
        CookieUtil.clear(response, RememberMeUtil.REMEMEBER_ME_COOKIE_NAME);
        logger.debug("this is not valid user!!");
    }

    return "redirect:/";
}
```


## 기타
잘못된 정보나 더 좋은 개선방법이 있다면 issue에 남겨주세요   
개발자에게 큰 도움이 됩니다 :>
