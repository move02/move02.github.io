---
layout: post
title: Spring Boot Test
categories: [TIL- Spring]
comments: true
---

이번 포스팅에서는 JUnit과 Spring Test 라이브러리를 이용한 Spring 환경에서의 테스트 환경 설정부터 이러한  개발환경에서 좋은 테스트 코드를 작성하기 위한 방법에 대해 알아볼 예정이다.
(이번 포스팅의 예시 코드는 Spring MVC 사용을 기준으로 작성하였다. Spring Webflux 환경에서의 테스트 정리는 공부 더 하고 다음 글에 작성하기로..)

#### Language & Project version
- Java 11
- Spring Boot 2.4.4
- Gradle 6.8.3

## Spring Test
spring-boot-test 라이브러리에서 Test의 설정을 지원하는 annotation에는 다양한 클래스가 존재하는데 기본으로 많이 사용되는 `@SpringBootTest`를 시작으로 몇 가지만 살펴보도록 하겠다.


### 테스트 환경 설정
Spring Boot 프로젝트를 시작하고 gradle dependency에 아래의 구문을 추가한다.
```java
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```
spring-boot-start-test 는 단위 테스트를 위한 JUnit과 Mock 객체 지원을 위한 Mockito, 가독성이 좋은 테스트 표현식을 만들기 위한 Matcher 라이브러리인 Hamcrest 등을 포함하고 있다.
spring-boot-start-test가 포함하는 전체 라이브러리는 아래와 같다. (Spring Boot 버전 2.2.0 이상)
- JUnit 5: The de-facto standard for unit testing Java applications.
- Spring Test & Spring Boot Test: Utilities and integration test support for Spring Boot applications
- AssertJ: A fluent assertion library.
- Hamcrest: A library of matcher objects (also known as constraints or predicates).
- Mockito: A Java mocking framework.
- JSONassert: An assertion library for JSON.
- JsonPath: XPath for JSON.

> 만약 spring-boot-start-test를 쓰지 않고 싶다면, spring-boot-test를 포함해 다른 라이브러리의 버전을 명시하면서 따로 가져오면 된다.

간단하게 테스트 관련 라이브러리들이 잘 동작하는지 확인하는 테스트를 작성해보자.
```java
package com.example.demo;

import com.example.demo.service.BookRepository;
import com.example.demo.service.TestConfiguration;
import org.junit.jupiter.api.Test;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.is;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles
class DemoApplicationTests {

	@Autowired
	@Qualifier("shoesRepository")
	private ShoesRepository shoesRepo;

	@Test
	void getShoesListTest() {
		assertThat(shoesRepo.getShoesList().size(), is(3));
	}
}
```

`@SpringBootTest` 어노테이션을 통해 Spring Boot Test를 라는 사실을 명시하고 테스트에 필요한 의존성을 제공해준다. (JUnit4 을 사용한다면 `@RunWith(SpringRunner.class)` 와 같은 어노테이션을 붙여줘야 하지만 JUnit5 부터는 `@SpringBootTest` 어노테이션에 포함되기 때문에 작성할 필요가 없다.)

`@SpringBootTest` 어노테이션의 classes 속성을 통해 클래스를 선택적으로 로드할 수도 있다. 위에서 작성한 테스트의 경우 `@Configuration` 어노테이션을 통해 `TestConfiguration` 이라는 클래스에서 사용할 빈을 선택적으로 등록했기 때문에 적어주었다. classes 속성에 클래스를 명시하지 않으면 모든 Bean을 가져온다.

`@ActiveProfiles`를 통해 profile 파일에 정의되어 있는 환경 속성들을 가져와서 테스트 어플리케이션을 설정한다. 만약, 실행환경 별로 여러 개의 profile이 정의되어 있다면(ex. profile-stage, profile-local, profile-product 등) `@Activeprofiles("dev")` 와 같이 특정 profile을 지정하는 것이 가능하다.

여기까지가 Spring Boot의 Test 관련 라이브러리들을 잘 가져왔는지에 대한 테스트였다.

### @SpringBootTest
`@SpringBootTest` 어노테이션은 org.springframework.boot.test.context 패키지에 위치하고 있으며 당연하겠지만 Spring Boot 기반의 테스트를 실행하는 테스트 클래스에 지정하는 어노테이션이다.

다음과 같은 기능을 제공한다. (from API Doc)
1. 별도의 `@ContextConfiguration(loader=...)` 어노테이션이 지정되어있지 않으면 `SpringBootContextLoader`를 default `ContextLoader`로 사용한다.
2. 내부에 `@Configuration`이나 `classes` 속성을 따로 정의하지 않았다면 `@SpringBootConfiguration`을 자동적으로 검색한다.
3. `properties` 속성을 통해 커스텀 환경속성을 정의할 수 있다.
4. `args` 속성을 통해 어플리케이션의 argument를 정의할 수 있다.
5. 특정 포트나 임의의 포트를 이용하여 실제 웹서버를 시작할 수 있는 기능을 포함한 몇 가지 `webEnvironment` 모드를 제공한다.
6. 실제 웹 서버를 실행시키는 웹 테스트를 위한 TestRestTemplate 과 WebTestClient bean을 등록한다.

1, 2번 특징을 통해 **별도의 설정을 해주지 않으면** 모든 bean을 로드하여 테스트를 할 수 있게 해준다. 반대로 말하면, 별도의 설정을 하지 않으면 테스트가 의도한바 보다 너무 무거워진다는 뜻이다.

5번의 `webEnvironment` 에서 제공하는 몇 가지 모드의 옵션은 다음과 같다.
- MOCK(default) : web ApplicationContext 를 로드하고 mock 웹 환경을 제공한다.
- RANDOM_PORT : WebServerApplicationContext(실제 서블릿 환경)를 로드하고 실제 웹 환경을 제공한다. 내장된 서버가 시작되고 랜덤한 포트를 listen한다. 통합테스트을 할 때만 이 옵션을 지정하는 것이 좋다.
- DEFINED_PORT : WebServerApplicationContext를 로드하고 실제 휍 환경을 제공한다.
- NONE : SpringApplication을 이용해 ApplicationContext를 로드하지만, 웹 환경은 제공하지 않는다.

TestConfiguration을 통해 기존의 Configuration을 커스터마이징이 가능하다. TestConfiguration은 ComponentScan 과정에서 생성되며, 테스트가 실행될 떄 bean을 생성하여 등록한다.

내부의 ComponentScan을 통해 감지되기 때문에 `classes` 속성을 통해 특정 클래스만 지정하면 TestConfiguration은 감지되지 않는다. 그런 경우에 `classes` 속성 값에 TestConfiguration 클래스를 추가하거나 `@Import` 어노테이션을 통해 가져올 수 있다.

```java
package com.example.demo;

import com.example.demo.service.RecommendationService;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;

import java.util.Random;

@TestConfiguration
public class DemoTestConfiguration {
    @Bean
    public RecommendationService restTemplate() {
        return new RecommendationService(){
            @Override
            public String recommendShoes() {
                return repository.getShoesList().get(new Random().nextInt(3));
            }
        };
    }
}
```
```java
package com.example.demo;

import com.example.demo.service.RecommendationService;
import com.example.demo.service.ShoesRepository;
import com.example.demo.service.DemoConfiguration;
import org.junit.jupiter.api.Test;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@SpringBootTest
@Import(DemoTestConfiguration.class)
class DemoApplicationTests {

	@Autowired
	@Qualifier("shoesRepository")
	private ShoesRepository shoesRepo;

	@Autowired
	@Qualifier("recommendationService")
	private RecommendationService service;

	@Test
	void getShoesListTest() {
		assertThat(shoesRepo.getShoesList().size(), is(3));
	}

	@Test
	void recommendTest() {
		assertThat(service.recommendShoes(), notNullValue());
	}
}
```

> **Note.** 
> `@Import` 어노테이션에 대해.. `@ComponentScan` 을 통해 ApplicationContext에 bean을 등록할 수 있는데 `@Import`라는 어노테이션이 따로 있는 이유는 무엇일까? Spring은 '설정보다는 관습' 이라는 철학을 기억해야한다. (baeldung 가이드에서는 `@ComponentScan`은 관습에 가깝지만 `@Import`는 설정에 가까워 보인다는 의견을 내놓았다.) `@Import` 는 모든(또는 다수의) Configuration이나 Component를 등록하고싶지 않거나, @ComponentScan으로 닿지 않는 영역의 클래스를 가져올 때 유용하다. 
> 가벼운 테스트 환경을 유지하기 위해 따로 테스트마다 필요한 설정만 모아두고 사용할 때 좋아보인다.

#### Mocking
spring-boot-starter-test 라이브러리에 포함된 Mockito를 이용하여 mock 객체를 만들고 이를 이용해 테스트를 해볼 수 있다. 이렇게하면 스프링 빈 주입에 의존하지 않아도 되고, 구체적인 동작이 구현되지 않은 컴포넌트의 테스트가 가능하다.
```java
package com.example.demo;

import com.example.demo.service.RecommendationService;
import com.example.demo.service.Shoes;
import com.example.demo.service.ShoesRepository;
import com.example.demo.service.DemoConfiguration;
import org.junit.jupiter.api.Test;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@SpringBootTest(classes = ShoesRepository.class)
class DemoApplicationTests {

	@Autowired
	@Qualifier("shoesRepository")
	private ShoesRepository shoesRepo;

	@Test
	void getShoesListTest() {
		assertThat(shoesRepo.getShoesList().size(), is(3));
	}

	@Test
	void recommendTest() {
		RecommendationService service = Mockito.mock(RecommendationService.class);
		Mockito.when(service.recommendShoes()).thenReturn(new Shoes("tennis"));

		assertThat(service.recommendShoes().getName(), is("tennis"));
	}
}
```

직접 Mockito를 이용하는 방법도 있지만, `@MockBean` 어노테이션을 사용해 Mock 객체를 빈으로 등록할 수 있다.
`@MockBean`으로 선언된 빈을 주입받으면 해당 빈에 의존성이 있는 다른 빈에는 Mock 객체를 주입한다.
```java
package com.example.demo;

import com.example.demo.service.RecommendationService;
import com.example.demo.service.Shoes;
import com.example.demo.service.ShoesRepository;
import com.example.demo.service.DemoConfiguration;
import org.junit.jupiter.api.Test;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.context.annotation.Import;

@SpringBootTest(classes = RecommendationService.class)
class DemoApplicationTests {

	@MockBean
	private ShoesRepository shoesRepo;

	@Autowired
	@Qualifier("recommendationService")
	private RecommendationService service;

	@Test
	void getShoesListTest() {
		assertThat(shoesRepo.getShoesList().size(), is(3));
	}

	@Test
	void recommendTest() {
		assertThat(service.recommendShoes().getName(), notNullValue());
	}
}
```
Controller 등을 테스트 할 때에는 MockMVC 객체를 이용해 서블릿을 모킹하여 테스트할 수 있다.
```java
package com.example.demo;

import com.example.demo.service.*;
import org.junit.jupiter.api.Test;

import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

@SpringBootTest(classes = DemoConfiguration.class)
@AutoConfigureMockMvc
class DemoApplicationTests {

	@Autowired
	MockMvc mockMvc;

	@Test
	public void controllerTest() throws Exception {
		mockMvc.perform(
				MockMvcRequestBuilders.get("/shoes/recommend")
						.contentType(MediaType.APPLICATION_JSON))
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andExpect(MockMvcResultMatchers.jsonPath("name").isNotEmpty());
	}
}
```

> Note.
> DemoConfiguration.class 를 통해 설정을 해주었는데 이때, Configuration 클래스에 `@EnableWebMvc` 어노테이션을 붙어야, Spring MVC 설정과 관련된 지원을 받아 정상적으로 Controller를 테스트할 수 있다.

### Test Slices
test-starter 패키지에 들어있는 spring-boot-test-autoconfigure 모듈은 전체가 스프링 어플리케이션이 아니라 원하는 부분만 조각으로 떼어내서 테스트 할 수 있는 설정을 자동으로 할 수 있도록 도와준다. `@...Test` 접미사가 붙으며 공통적으로 ApplicationContext를 로드하고 해당 테스트 슬라이스와 관련된 설정을 자동으로 해주는 `@AutoConfigure...` 어노테이션을 포함한다.

각각의 슬라이스는 적절한 컴포넌트만 찾도록 ComponentScan을 제한하며 매우 제한된 자동설정 클래스 집합을 로드한다. `@...Test` 어노테이션의 `excludeAutoConfiguration` 이라는 속성을 이용하거나 `@ImportAutoConfiguration#exclude` 어노테이션을 통해 특정 클래스를 제외시키면서 설정을 가져올 수 있다.

하나의 테스트에서 여러 슬라이스를 사용하는 것은 지원하지 않는다. 이를 원할경우, `@...Test` 어노테이션 중 하나를 선택하고, `@AutoConfigure...`를 통해 원하는 슬라이스의 설정들을 (+ 필요한 빈) 직접 로드해야 한다. 

#### 목록
다 사용해보면서 포스팅을 작성할 수 없기 때문에 `@...Test` 슬라이스들의 종류만 나열하고 넘어가겠다.
각 슬라이스에 포함된 auto-configuration 어노테이션들은 [이 페이지](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-test-auto-configuration.html)에서 확인할 수 있다.
각 슬라이스에 대한 자세한 설명은 [공식문서 - Auto-configured Tests](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests) 에서 확인할 수 있다.
- `@DataCassandraTest`
- `@DataJdbcTest`
- `@DataJpaTest`
- `@DataLdapTest`
- `@DataMongoTest`
- `@DataNeo4jTest`
- `@DataR2dbcTest`
- `@DataRedisTest`
- `@JdbcTest`
- `@JooqTest`
- `@JsonTest`
- `@RestClientTest`
- `@WebFluxTest`
- `@WebMvcTest`
- `@WebServiceClientTest`

### Spring Unit Test?

앞선 TDD 정리 포스팅에서 단위테스트의 Best practic를 보면 **단위 테스트는 고립되어야 한다.** 라고 되어있다.
그러나 Spring Framework는 객체 생성과 주입에 깊게 관여하기 때문에 `@SpringBootTest` 와 같은 어노테이션을 사용하는 테스트 Unit Test라고 말하기 어렵다고 생각한다. (Test Slice들도 마찬가지) 

아래 예시 코드와 같이 Spring에서 bean으로 등록해둔 클래스도 단위 테스트가 가능하다.

```java
// ShoesController class
@Controller
public class ShoesController {

    @GetMapping("/shoes")
    public ModelAndView getShoes() {
        return new ModelAndView("shoes", "shoe_type", "boots");
    }
}

// ShoesHandlerTest class

import com.sample.move02.handler.ShoesController;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.hamcrest.MatcherAssert.*;
import static org.hamcrest.Matchers.*;

public class ShoesControllerTest {
    private ShoesController controller;

    @Before
    public void setUp() {
        controller = new ShoesController();
    }

    @Test
    public void getShoesShouldUseShoesView() {
       assertThat(controller.getShoes().getViewName()).isEqualTo("shoes");
    }

    @Test
    public void getShoesShouldAddAShoeTypeModel() {
        assertThat(controller.getShoes().getModel()).containsEntry("shoe_type", "boots");
    }
}
```

위의 예제처럼 Test 자동화를 위한 JUnit 프레임워크 외에 다른 의존성이 없어야만 "단위 테스트" 라고 부를만 하다. 이렇기 때문에 Spring 프레임워크 위에서 모든 클래스에 대해 완벽한 단위테스트를 실행하기는 어렵고 번거롭다. 

Spring Test에서 제공하는 Slice나 클래스의 부분적인 주입을 적절히 이용하여 Spring 컨테이너 위에서도 가능한 가볍게 테스트를 작성하는 습관을 들이는 것이 좋아보인다.


## 참고
- [SpringBoot 공식문서](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-testing)
- [SpringBoot API 문서](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/test/context/SpringBootTest.html)
- [NHN Meetup](https://meetup.toast.com/posts/124)
- [에디의 기술블로그](https://brunch.co.kr/@springboot/207)
- [Baeldung - Spring @Import Annotation](https://www.baeldung.com/spring-import-annotation)
- [Stackoverflow - Is Spring Framework used in a unit test?](https://stackoverflow.com/questions/50346837/is-spring-framework-used-in-a-unit-test)