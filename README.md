# 외부설정과 프로필
## 외부 설정 사용 - Environment
스프링에서 제공하는 `Environment`를 통해서 일관된 방식으로 조회할 수 있다.
스프링은 `Environment`는 물론이고 `Environment`를 활용해서 더 편리하게 외부 설정을 읽는 방법들을 제공한다.

스프링이 지원하는 다양한 외부 설정 조회 방법
* `Environment`
* `@Value` - 값 주입
* `@ConfigurationProperties` - 타입 안전한 설정 속성

MyDataSourceEnvConfig
```java
@Slf4j
@Configuration
public class MyDataSourceEnvConfig {
    private final Environment env;

    public MyDataSourceEnvConfig(Environment env) {
        this.env = env;
    }
    
    @Bean
    public MyDataSource myDataSource() {
        String url = env.getProperty("my.datasource.url");
        String username = env.getProperty("my.datasource.username");
        String password = env.getProperty("my.datasource.password");
        int maxConnection = env.getProperty("my.datasource.etc.max-connection", Integer.class);
        Duration timeout = env.getProperty("my.datasource.etc.timeout", Duration.class);
        List<String> options = env.getProperty("my.datasource.etc.options", List.class);
        return new MyDataSource(url, username, password, maxConnection, timeout, options);
    }
}
```
* `Environment`를 사용하면 외부 설정의 종류와 관계없이 코드 안에서 일관성 있게 외부 설정을 조회할 수 있다.
* `Environment.getProperty(key, Type)`를 호출할 때 타입 정보를 주면 해당 타입으로 변환해준다.

ExternalReadApplication
```java
import hello.config.MyDataSourceEnvConfig;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Import;

@Import(MyDataSourceEnvConfig.class)
@SpringBootApplication(scanBasePackages = "hello.datasource")
public class ExternalReadApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExternalReadApplication.class, args);
    }
}
```
* 설정 정보를 빈으로 등록해서 사용하기 위해 `@Import(MyDataSourceEnvConfig.class)`를 추가 했다.

단점
`Environment`를 직접 주입받고, `env.getProperty(key)`를 통해서 값을 꺼내는 과정을 반복 해야 한다는 점이다. 스프링은 `@Value`를 통해서 외부 설정값을 주입 받는 더 편리한 기능을 제공한다.

## 외부설정 사용 - @Value
MyDataSourceValueConfig
```java
import hello.datasource.MyDataSource;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.time.Duration;
import java.util.List;

@Slf4j
@Configuration
public class MyDataSourceValueConfig {
    @Value("${my.datasource.url}")
    private String url;
    @Value("${my.datasource.username}") 
    private String username;
    @Value("${my.datasource.password}")
    private String password;
    @Value("${my.datasource.etc.max-connection}")
    private int maxConnection;
    @Value("${my.datasource.etc.timeout}")
    private Duration timeout;
    @Value("${my.datasource.etc.options}")
    private List<String> options;
    
    @Bean
    public MyDataSource myDataSource1() {
        return new MyDataSource(url, username, password, maxConnection, timeout, options);
    }
    
    @Bean
    public MyDataSource myDataSource2(
        @Value("${my.datasource.url}") String url,
        @Value("${my.datasource.username}") String username,
        @Value("${my.datasource.password}") String password,
        @Value("${my.datasource.etc.max-connection}") int maxConnection,
        @Value("${my.datasource.etc.timeout}") Duration timeout,
        @Value("${my.datasource.etc.options}") List<String> options) {
        return new MyDataSource(url, username, password, maxConnection, timeout, options);
    }
 }
```
* `@Value`에 `${}`를 사용해서 외부 설정의 키 값을 주면 원하는 값을 주입받을 수 있다.
* `@Value`는 필드에 사용할 수 있고, 파라미터에도 사용할 수 있다.

기본값
만약 키를 찾지 못할 경우 코드에서 기본값을 사용하려면 다음과 같이 `:` 뒤에 기본값을 적어주면 된다.
* 예) `@Value("${my.datasource.etc.max-connection:1}")`: `key`가 없는 경우 `1`을 사용한다.

단점
`@Value`를 사용하는 방식도 좋지만, `@Value`로 하나하나 외부 설정 정보의 키 값을 입력받고, 주입 받아와야하는 부분이 번거롭다.
그리고 설정 데이터를 보면 하나하나 분리되어 있는 것이 아니라 정보의 묶음으로 되어있다. 여기서는 `my.datasource` 부분으로 묶여 있다.
이런 부분을 객체로 변환해서 사용할 수 있다면 더 편리하고 더 좋을 것이다.

## 외부설정 사용 - @ConfigurationProperties
Type-safe Configuration Properties
스프링은 외부 설정의 묶음 정보를 객체로 변환하는 기능을 제공한다. 이것들을 타입 안전한 설정 속성이라 한다.
객체를 사용하면 타입을 사용할 수 있다. 따라서 실수로 잘못된 타입이 들어오는 문제도 방지할 수 있고, 객체를 통해서
활용할 수 있는 부분이 많아진다. 쉽게 이야기해서 외부 설정을 자바 코드로 관리할 수 있는 것이다. 그리고 설정 정보 그 자체도 타입을 가지게 된다.

MyDataSourcePropertiesV1
```java
@Data
@ConfigurationProperties("my.datasource")
public class MyDataSourcePropertiesV1 {
    private String url;
    private String username;
    private String password;
    private Etc etc = new Etc();
    
    @Data
    public static class Etc {
        private int maxConnection;
        private Duration timeout;
    }
    private List<String> options = new ArrayList<>();
}
```
* 외부설정을 주입 받을 객체를 생성한다. 그리고 각 필드를 외부 설정의 키 값에 맞추어 준비한다.
* `@ConfigurationProperties` 애노테이션이 있으면 외부설정을 주입 받는 객체라는 뜻이다. 여기에 외부 설정 KEY의 묶음 시점인 `my.datasource`를 적어준다.
* 기본 주입 방식은 자바빈 프로퍼티 방식이다. `Getter`, `Setter`가 필요하다.

MyDataSourceConfigV1
```java
@Slf4j
@EnableConfigurationProperties(MyDataSourcePropertiesV1.class)
public class MyDataSourceConfigV1 {
    private final MyDataSourcePropertiesV1 properties;
    public MyDataSourceConfigV1(MyDataSourcePropertiesV1 properties) {
        this.properties = properties;
    }
    
    @Bean
    public MyDataSource dataSource() {
        return new MyDataSource(
                properties.getUrl(),
                properties.getUsername(),
                properties.getPassword(),
                properties.getEtc().getMaxConnection(),
                properties.getEtc().getTimeout(),
                properties.getEtc().getOptions());
    }
 }
```
* `@EnableConfigurationProperties(MyDataSourcePropertiesV1.class)`
  * 스프링에게 사용할 `@ConfigurationProperties`를 지정해주어야 한다. 이렇게 하면 해당 클래스는 스프링 빈으로 등록되고, 필요한 곳에서 주입받아서 사용할 수 있다.
* `private final MyDataSourcePropertiesV1 properties` 설정 속성을 생성자를 통해 주입받아서 사용한다.

문제
`MyDataSourcePropertiesV1`은 스프링 빈으로 등록된다. 그런데 `Setter`를 가지고 있기 때문에 누군가 실수로 값을 변경하는 문제가 발생할 수 있다.
여기에 있는 값들은 외부 설정값을 사용해서 초기에만 설정이 되고, 이후에는 변경하면 안된다.
이럴때 `Setter`를 제거하고 대신에 생성자를 사용하면 중간에 데이터를 변경하는 실수를 근본적으로 방지할 수 있다.

## 외부설정 사용 - @ConfigurationProperties
`@ConfigurationProperties`는 `Getter`, `Setter`를 사용하는 자바빈 프로퍼티 방식이 아니라 생성자를 통해서 객체를 만드는 기능도 지원한다.

MyDataSourcePropertiesV2
```java
@Getter
@ConfigurationProperties("my.datasource")
public class MyDataSourcePropertiesV2 {
    private String url;
    private String username;
    private String password;
    private Etc etc;
    
    public MyDataSourcePropertiesV2(String url, String username, String password, @DefaultValue Etc etc) {
        this.url = url;
        this.username = username;
        this.password = password;
        this.etc = etc;
    }
    
    @Getter
    public static class Etc {
        private int maxConnection;
        private Duration timeout;
        private List<String> options;
        
        public Etc(int maxConnection, Duration timeout, @DefaultValue("DEFAULT") List<String> options) {
            this.maxConnection = maxConnection;
            this.timeout = timeout;
            this.options = options;
        }
    }
 }
```
* 생성자를 만들어 두면 생성자를 통해서 설정 정보를 주입한다.

> 참고 `@ConstructorBinding`

스프링 부트 3.0 이전에는 생성자 바인딩시에 `@ConstructorBinding` 애노테이션을 필수로 사용해야 했다.
스프링 부트 3.0부터는 생성자가 하나일 때는 생략할 수 있다. 생성자가 둘 이상인 경우에는 사용할 생성자에 `@ConstructorBinding` 애노테이션을 적용하면 된다.

## 외부설정 사용 - @ConfigurationProperties 검증
`@ConfigurationProperties`를 통해서 숫자가 들어가야 하는 부분에 문자가 입력되는 문제와 같은 타입이 맞지 않는 데이터를 입력하는 문제는 예방할 수 있다.
그런데 문제는 숫자의 범위라던가, 문자의 길이 같은 부분은 검증이 어렵다.
예를 들어 최대 커넥션 숫자는 최소 `1`, 최대 `999`라는 범위를 가져야 한다면 어떻게 검증할 수 있을까?
개발자가 직접 하나하나 검증 코드를 작성해도 되지만, 자바에는 자바 빈 검증기(java bean validation)이라는 훌륭한 표준 검증기가 제공된다.
`@ConfigurationProperties`는 자바 객체이기 때문에 스프링이 자바 빈 검증기를 사용할 수 있도록 지원한다.

```java
@Getter
@ConfigurationProperties("my.datasource")
@Validated
public class MyDataSourcePropertiesV3 {
    @NotEmpty
    private String url;
    @NotEmpty
    private String username;
    @NotEmpty
    private String password;
    private Etc etc;
    
    public MyDataSourcePropertiesV3(String url, String username, String password, Etc etc) {
        this.url = url;
        this.username = username;
        this.password = password;
        this.etc = etc;
    }
    
    @Getter
    public static class Etc {
        @Min(1)
        @Max(999)
        private int maxConnection;
        @DurationMin(seconds = 1)
        @DurationMax(seconds = 60)
        private Duration timeout;
        private List<String> options;
 
        public Etc(int maxConnection, Duration timeout, List<String> options) {
            this.maxConnection = maxConnection;
            this.timeout = timeout;
            this.options = options;
        }
    }
 }
```

ConfigurationProperties 장점
* 외부설정을 객체로 편리하게 변환해서 사용할 수 있다.
* 외부설정의 계층을 객체로 편리하게 표현할 수 있다.
* 외부설정을 타입 안전하게 사용할 수 있다.
* 검증기를 적용할 수 있다.

## YAML
스프링 설정 데이터를 사용할때 `application.properties`뿐만 아니라 `application.yml`이라는 형식도 지원한다.
YAML(YAML Ain't Markup Language)은 사람이 읽기 좋은 데이터 구조를 목표로 한다. 확장자는 `yaml`, `yml`이다. 주로 `yml`을 사용한다.

## @Profile
프로필과 외부설정을 사용해서 각 환경마다 설정값을 다르게 적용하는 것은 이해했다.
그런데 설정값이 다른 정도가 아니라 각 환경마다 서로 다른 빈을 등록해야 한다면 어떻게 해야할까?

PayClient
```java
public interface PayClient {
 void pay(int money);
}
```
LocalPayClient
```java
@Slf4j
public class LocalPayClient implements PayClient {
    @Override
    public void pay(int money) {
        log.info("로컬 결제 money={}", money);
    }
 }
```
ProdPayClient
```java
@Slf4j
public class ProdPayClient implements PayClient {
    @Override
    public void pay(int money) {
        log.info("운영 결제 money={}", money);
    }
 }
```
OrderService
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PayClient payClient;
    public void order(int money) {
        payClient.pay(money);
    }
 }
```
* `PayClient`를 사용하는 부분이다. 상황에 따라 `LocalPayClient` 또는 `ProdPayClient`를 주입받는다.

PayConfig
```java
@Slf4j
@Configuration
public class PayConfig {
    @Bean
    @Profile("default")
    public LocalPayClient localPayClient() {
        log.info("LocalPayClient 빈 등록");
        return new LocalPayClient();
    }
    
    @Bean
    @Profile("prod")
    public ProdPayClient prodPayClient() {
        log.info("ProdPayClient 빈 등록");
        return new ProdPayClient();
    }
 }
```
* `@Profile` 애노테이션을 사용하면 해당 프로필이 활성화 된 경우에만 빈을 등록한다.
