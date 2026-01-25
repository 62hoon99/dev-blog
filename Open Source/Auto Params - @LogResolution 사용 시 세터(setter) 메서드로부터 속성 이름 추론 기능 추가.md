# Auto Params
## Auto Params 오픈 소스에 기여하게 된 계기
[오픈 소스 기여 모임](https://github.com/opensource-contributors-group) 10기에 참가하여 기여할 오픈 소스를 모색하던 중, 이 전에 '한국 스프링 사용자 모임'에서 주관한 [스프링 캠프 2024](https://springcamp.ksug.org/2024/) 에서 Auto Params의 메인테이너이신 이규원님의 세션을 들었었던 기억이 있어서 해당 오픈소스를 알고 있었고, 사용해본 경험은 없었지만 어떤 오픈 소스인지 알고 있었습니다. 그래서 Auto Params의 Github에 들어가 등록되어 있는 이슈 중 해결 가능한 이슈를 선택하여 기여를 진행하게 되었습니다.

## Auto Params 오픈 소스에 대한 간단 설명
AutoParams는 Java 및 Kotlin으로 작성된 테스트 데이터 생성기입니다.

AutoParams는 테스트 메서드 매개변수에 자동으로 생성된 값을 제공하여 이러한 반복적인 작업을 없애주므로, 간결하고 핵심에 집중된 테스트를 작성할 수 있습니다.

시작하는 방법은 다음과 같습니다. `@Test`메서드에 `@AutoParams``@Process` ( `@AutoKotlinParams`Kotlin에서는 `@Process`) 어노테이션을 추가하면 매개변수에 생성된 데이터가 채워집니다.

``` java
public class AutoParamsExampleTest {  
  
    @Test  
    @AutoParams    
    void testMethod(int a, int b) {  
       Calculator sut = new Calculator();  
       int actual = sut.add(a, b);  
       assertEquals(a + b, actual);  
    }
}
```

## @LogResolution 이란?
AutoParams는 테스트 실행 중 객체가 어떻게 해석되는지 이해하는 데 도움이 되는 자세한 로깅 기능을 제공합니다. `@LogResolution`을 테스트 메서드에 추가하면 로깅 기능이 활성화됩니다. 활성화되면 해석 로그가 테스트 실행 중 표준 출력으로 출력되어 각 값이 어떻게 생성되는지 쉽게 추적할 수 있습니다.

예시 코드와 출력 결과는 다음과 같습니다.

**예시 코드**
``` java
public class AutoParamsExampleTest {  
  
    @Test  
    @AutoParams    
    @LogResolution    
    void testMethod(User user) {  
    }  
  
    public record User(UUID id, String email, String username) {  
    }  
}
```

**출력 결과**
```
User user (4ms)
 ├─ UUID id → 85143f69-bb81-4cf2-bedf-3617645bf571 (< 1ms)
 ├─ String email → 4c48a71b-24b2-49f0-8317-f5e31c286e15@test.com (1ms)
 └─ String username → username28fcf5cf-dfea-4a78-9560-58542b1dc35f (< 1ms)
```

# 이슈 선정 및 내용 파악
## 이슈 선정
[해당 이슈](https://github.com/AutoParams/AutoParams/issues/318)는 메인테이너가 제목만 작성한 이슈였습니다. 이슈의 제목은 `@LogResolution 사용 시 세터(setter) 메서드로부터 속성 이름 추론 기능 추가` 이었습니다. 저는 이슈 파악을 위해 Auto Params의 [공식 문서](https://autoparams.github.io/) 를 통해 @LogResolution 이 무슨 어노테이션인지 파악하였습니다. 그리고 