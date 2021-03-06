# my-junit

안녕하세요? 이번 시간엔 JUnit을 직접 만들어보는 시간을 가지려고 합니다.  
모든 코드는 [Github](https://github.com/jojoldu/blog-code/tree/master/my-junit)에 있기 때문에 함께 보시면 더 이해하기 쉬우실 것 같습니다.  
(공부한 내용을 정리하는 [Github](https://github.com/jojoldu/blog-code)와 세미나+책 후기를 정리하는 [Github](https://github.com/jojoldu/review), 이 모든 내용을 담고 있는 [블로그](http://jojoldu.tistory.com/)가 있습니다. )<br/>
 

## 계기

긴 추석연휴 기간동안 미뤄둔 포스팅 예정 글들을 정리했습니다.  
3개를 연달아 처리하고 뭐가 더 남았나 에버노트를 보다가 아주 예전에 메모해놓은 일감이 있었습니다.  
바로 **나만의 XUnit 만들기**입니다.  
  
토비님께서 올리신 글을 보고 일감 등록을 했었던 기억이 떠올랐습니다.  
![토비님글](./images/토비님글.png)

(원분 : [페이스북링크](https://www.facebook.com/tobyilee/posts/10208774948625630))  
  
일단 회사에서 사용하는 기술들을 익히기에 급급해 계속 미루다가 이제야 다시 봤습니다.  
장기간 휴식이 또 언제 생길지 모르니 지금 당장 시작하기로 마음먹고 진행하게 되었습니다.  
(토비님의 블로그 글을 꼭 읽어보시면 큰 도움이 되실것 같습니다.)  
  
마침 [[번역]JUnit A Cook’s Tour](https://bluepoet.me/2016/12/03/%EB%B2%88%EC%97%ADjunit-a-cooks-tour/)도 있어 참고하며 진행할 예정입니다.

## 본문

빌드툴은 Gradle을 사용할 예정입니다.  
IDE는 IntelliJ 를 사용하는데, 스프링과 같은 웹 개발이 필요한 부분이 아니기 때문에 꼭 유료버전인 Ultimate이 아니더라도 무료 버전인 Community 버전을 사용하셔도 됩니다.  
(이클립스 쓰셔도 됩니다)  
  
build.gradle은 로그 관리를 위해 2개의 의존성을 먼저 추가하고 시작하겠습니다.  

build.gradle

```gradle
dependencies {
    compile group: 'org.slf4j', name: 'slf4j-api', version: '1.7.25' // log를 위해 slf4j 추가
    compile group: 'ch.qos.logback', name: 'logback-classic', version: '1.2.3' // log를 위해 logback 추가
}
```

혹시나 logback을 처음 접하신다면 [소내기님의 logback 튜토리얼](https://sonegy.wordpress.com/category/logback/)을 참고하시면 좋습니다.  
  
기본적인 디렉토리 구조는 다음과 같습니다.

![디렉토리구조](./images/디렉토리구조.png)

src/main/java,resources & src/test/java,resources이며 기본 패키지는 myjunit입니다.  
  
자 그럼 첫번째 테스트 코드를 작성해보겠습니다.

### 1. 첫 테스트 코드

간단하게 테스트 코드를 작성하겠습니다.  
src/test/java/myjunit에 ```TestCaseTest.java```를 생성하고 아래의 코드를 추가하겠습니다.

```java
public class TestCaseTest {

    public static void main(String[] args) {
        new TestCaseTest().runTest();
    }

    public void runTest () {
        long sum = 10+10;
        Assert.assertTrue(sum == 20);
    }
}

```

현재 JUnit이 의존성으로 등록되어있지 않기 때문에 Assert 클래스가 존재하지 않습니다.  
이 테스트 코드를 수행하기 위해 ```Assert``` 클래스와 static Method인 ```assertTrue```를 생성하겠습니다.  

```java
public class Assert {
    private static final Logger logger = LoggerFactory.getLogger(Assert.class);

    private Assert() {} // 인스턴스 생성을 막기 위해 기본생성자 private 선언

    public static void assertTrue(boolean condition) {
        if(!condition){
            throw new AssertionFailedError();
        }

        logger.info("Test Passed");
    }
}
```

slf4j logger가 의존성으로 등록되어있기 때문에 logger는 sfl4j logger를 사용합니다.  
 ```assertTrue```는 단순합니다.  
파라미터로 넘겨진 boolean 값이 ```true```이면 Test는 성공이며(```logger.info("Test Passed")```), ```false```하면 ```throw new AssertionFailedError()```를 발생하여 테스트 실패를 알립니다.  
 ```AssertionFailedError```도 존재하지 않는 클래스이니 추가 생성합니다.

```java
public class AssertionFailedError extends Error {
    public AssertionFailedError() {}
}
```

자 이렇게 구성하고 ```TestCaseTest```의 main 메소드를 다시 실행해보겠습니다.

![테스트성공1](./images/테스트성공1.png)

이렇게 첫번째 테스트가 성공되었음을 확인할 수 있습니다.

### 2. TestCase

현재 구조는 **테스트 프레임워크**라고 할수는 없습니다.  
결국 각각의 테스트 케이스 단위로 요청을 나눌수 있는 구조가 되어야 합니다.  
이런 의도에 가장 어울리는 것이 [커맨드패턴](http://javacan.tistory.com/entry/6)입니다.  
  
각각의 테스트 케이스를 Command로 보고, 이를 실행하는 것은 ```run```(보통은 ```execute```를 사용) 메소드가 담당하도록 합니다.  
  
이렇게 해서 만든 ```TestCase```의 코드는 아래와 같습니다.

```java
public abstract class TestCase {

    protected String testCaseName;

    public TestCase(String testCaseName) {
        this.testCaseName = testCaseName;
    }

    public abstract void run();
}
```

각각의 TestCase는 이름을 가져야 식별가능하기 때문에 생성자에서 이를 받도록 하였습니다.  
 ```TestCase```는 그 자체로 사용하기 보다는 이를 상속한 실제 테스트 케이스 클래스들을 사용할것이기 때문에 추상 클래스(```abstract class```)로 선언하였습니다.  
  
위 ```TestCase```클래스를 사용하여 ```TestCaseTest```코드를 수정하겠습니다.  

```java
public class TestCaseTest extends TestCase {

    public TestCaseTest(String testCaseName) {
        super(testCaseName);
    }

    @Override
    public void run() {
        try {
            Method method = this.getClass().getMethod(super.testCaseName, null);
            method.invoke(this, null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void runTest() {
        long sum = 10+10;
        Assert.assertTrue(sum == 20);
    }

    public static void main(String[] args) {
        new TestCaseTest("runTest").run();
    }
}
```

> 왜 TestCase의 구현체인 TestCaseTest를 new로 인스턴스 생성해서 써야하는지 의문이실수 있습니다.  
**각각의 테스트가 독립적으로 실행**하기 위해 모든 테스트 케이스는 새로운 인스턴스에서 수행되도록 하였습니다.  
  
그리고 변경한 테스트 코드도 잘 성공되는지 테스트 해보겠습니다.  

![테스트성공2](./images/테스트성공2.png)

이제는 테스트 케이스 메소드들을 하나만 생성하는 것이 아니라, 여러개를 생성하고 실제 main 메소드에선 해당 메소드들의 이름만 추가하면 테스트를 실행할수 있게 되었습니다.  

실제로 ```TestCaseTest```에 테스트 메소드를 하나더 추가해보겠습니다.

```java
    ... 
    private static final Logger logger = LoggerFactory.getLogger(TestCaseTest.class);

    ... 

    @Override
    public void run() {
        try {
            logger.info("{} execute ", testCaseName); // 테스트 케이스들 구별을 위해 name 출력 코드
            Method method = this.getClass().getMethod(super.testCaseName, null);
            method.invoke(this, null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    ...
    
    public void runTestMinus() {
        long minus = 100-10;
        Assert.assertTrue(minus == 90);
    }

    public static void main(String[] args) {
        new TestCaseTest("runTest").run();
        new TestCaseTest("runTestMinus").run();
    }
```

 ```run``` 메소드에 실행한 테스트 메소드를 확인하기 위해 메소드명을 출력하도록 ```logger```를 추가하였습니다.  
 ```runTestMinus```를 추가하여 main 메소드에서 2개의 테스트 케이스를 실행하는데 파라미터만 변경하면 실행하는것을 확인할 수 있습니다.  

이 코드를 실제로 수행해보시면!

![테스트성공3](./images/테스트성공3.png)

테스트가 잘 성공됨을 알 수 있습니다.  
이 코드를 보시면서 뭔가 리팩토링 할만한게 보이시지 않나요?  
  
오버라이딩 하고 있는 ```run``` 메소드는 ```TestCase``` 클래스를 상속하는 모든 하위 클래스들이 다시 구현해야하는데, 실제로 하는 일은 공통으로 뽑아도 무방한 일입니다.  
이런 코드는 부모인 ```TestCase```가 가지는게 좀 더 좋아보입니다.  
그래서 ```run``` 메소드를 ```TestCase```로 이동시키겠습니다.

```java
public abstract class TestCase {
    
    private static final Logger logger = LoggerFactory.getLogger(TestCase.class);

    protected String testCaseName;

    public TestCase(String testCaseName) {
        this.testCaseName = testCaseName;
    }

    public void run() {
        try {
            logger.info("{} execute ", testCaseName); // 테스트 케이스들 구별을 위해 name 출력 코드
            Method method = this.getClass().getMethod(testCaseName, null);
            method.invoke(this, null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

그리고 코드를 리팩토링했으니 다시한번 기존 테스트를 실행해보겠습니다.

![테스트성공4](./images/테스트성공4.png)

 ```TestCaseTest``` **코드가 대폭 다이어트 되면서도 테스트 코드는 여전히 잘 수행**되었습니다.  
리팩토링이 잘된거라 볼수있겠죠?  
  
현재까지 상황을 다이어그램으로 표기하면 아래와 같습니다.

![스냅샷1](./images/스냅샷1.png)

### 3. Fixture 메소드

현재까지 제작한 ```TestCase``` 클래스와 기존에 사용하던 JUnit 기능를 비교해서 빠진 기능이 뭐가 있을까요?  
가장 먼저 떠오르는 기능은 ```Fixture``` 메소드인것 같습니다.  
용어가 조금 어려운데 저희가 흔히 사용하는 ```@Before```, ```@After```와 같이 **각각의 테스트 케이스들에게 공통적으로 수행되는 메소드**를 얘기합니다.  
  
자 예를 들어 아래와 같이 ```TestCaseTest```를 수정해보겠습니다.

![fixture문제](./images/fixture문제.png)

JUnit이였다면 이렇게 테스트 코드를 작성하지 않겠죠?  
각각의 테스트 케이스들 앞/뒤로 혹은 **특별한 시점에 공통적으로 코드를 수행**하고 싶다면 어떻게 해야할까요?  
  
[템플릿메소드 패턴](http://jdm.kr/blog/116)은 현재 상황에 적용할 수 있는 아주 적절한 디자인패턴입니다.  
  
템플릿 메소드 패턴을 적용하여 ```TestCase```를 수정해보겠습니다.

```java
public abstract class TestCase {

    private static final Logger logger = LoggerFactory.getLogger(TestCase.class);

    protected String testCaseName;

    public TestCase(String testCaseName) {
        this.testCaseName = testCaseName;
    }

    public void run(){
        before();
        runTestCase();
        after();
    }

    protected void before() {}

    private void runTestCase() {
        try {
            logger.info("{} execute ", testCaseName); // 테스트 케이스들 구별을 위해 name 출력 코드
            Method method = this.getClass().getMethod(testCaseName, null);
            method.invoke(this, null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    protected void after() {}
}
```

> JUnit A Cook’s Tour에서는 ```setup```, ```tearDown```이라고 표현하지만 여기선 좀더 친숙하게 before, after로 표현하겠습니다.

기존의 ```run``` 메소드를 수정하여 ```before()``` -> ```runTestCase()``` -> ```after()``` 순으로 실행되도록 구조화시켰습니다.  
  
여기서 ```before()```와 ```after()```는 추상메소드가 아닙니다.  
**일반메소드이지만 구현부분이 없이** 생성하였습니다.  
추상메소드로 구현할 경우 상속받는 클래스들에선 무조건 오버라이딩 해야하는데, 이들은 **강제로 구현해야할 대상은 아니고 선택대상**이기 때문입니다.  
  
이제 이에 맞춰 테스트 클래스인 ```TestCaseTest``` 코드를 개선하고 다시 테스트를 실행해보겠습니다.

![테스트성공5](./images/테스트성공5.png)

 ```before``` 메소드가 각각의 테스트 케이스마다 수행되어 ```base``` 변수에 10이 할당되어 테스트가 성공됨을 확인할 수 있습니다.  
  
자 여기까지 상황을 다이어그램으로 표현하면 아래와 같습니다.

![스냅샷2](./images/스냅샷2.png)  

### 4. TestResult

자 이제 이번에는 테스트 결과와 관련된 기능을 추가하겠습니다.  
테스트 코드를 수행하면 전체 테스트 케이스 몇개가 수행되었으며, 이들중 몇개가 테스트가 실패하고 몇개가 성공했는지 등등의 기능이 제공됩니다.  
  
현재는 이런 테스트 결과에 대한 레포팅을 할 수 없기 때문에 이 기능을 추가해보겠습니다.  
  
여러 테스트 메소드들, Fixture 메소드등으로 결과와 관련된 데이터들을 다 수집하기에 적절한 패턴은 [Collecting Parameter 패턴](http://www.javajigi.net/display/SWD/Move+Accumulation+to+Collecting+Parameter) 입니다.  
**메서드 파라미터에 결과를 수집할 객체를 전달**하는 방식으로 구현할 예정입니다.  
수집할 테스트 결과 객체의 클래스명은 ```TestResult```로 하겠습니다.  
현재 기능은 간단하게 수행된 테스트 개수만 담도록 하겠습니다.

```java
public class TestResult {
    private static final Logger logger = LoggerFactory.getLogger(TestResult.class);
    private int runTestCount;

    public TestResult() {
        this.runTestCount = 0;
    }

    /**
     synchronized: 하나의 TestResult 인스턴스를 여러 테스트 케이스에서 사용하게 될 경우
     쓰레드 동기화 문제가 발생하므로 여기선 synchronized로 간단하게 해결합니다.
     (테스트 케이스에서만 사용하므로 실시간 성능 이슈를 고려하지 않아도 되기 때문입니다)
     */
    public synchronized void startTest() {
        this.runTestCount++;
    }

    public void printCount(){
        logger.info("Total Test Count: {}", runTestCount);
    }
}
```

 ```startTest``` 메소드엔 ```synchronized```를 사용하였습니다.  
하나의 ```TestResult``` 인스턴스에 테스트 전체 결과에 대한 정보를 담으려면 여러 테스트 케이스가 사용하게되는데, 이때
쓰레드 동기화 문제가 발생하므로 여기선 synchronized로 간단하게 해결하였습니다.
(테스트 케이스에서만 사용하므로 실시간 성능 이슈를 고려하지 않아도 되기 때문입니다)  
  
자 그러면 이제 각 테스트 케이스마다 이 TestResult를 사용하도록 ```TestCase``` 클래스를 수정해보겠습니다.

```java
public abstract class TestCase {

    ...

    public TestResult run(){
        TestResult testResult = createTestResult();
        run(testResult);

        return testResult;
    }

    public void run(TestResult testResult){
        testResult.startTest();
        before();
        runTestCase();
        after();
    }

    private TestResult createTestResult() {
        return new TestResult();
    }
    ...
}

```

기존에 있던 ```run()``` 메소드는 ```TestResult```를 파라미터로 받아, ```startTest()```를 실행하는 메소드로 수정하였습니다.  
그리고 파라미터 없는 ```run()```를 다시 생성하여 ```TestResult```를 자체적으로 인스턴스 생성해서 파라미터 **있는** ```run``` 메소드를 호출하도록 하였습니다.  
이렇게 되면 어느 테이스 케이스든 ```run```를 호출할 경우 테스트 count가 1씩 증가하는 것을 체크할 수 있게 됩니다.  
  
그럼 ```TestCaseTest``` 코드를 다시 수정해서 기능을 확인해보겠습니다!

```java
public class TestCaseTest extends TestCase {
    ...
    public static void main(String[] args) {
        TestResult testResult = new TestResult();
        new TestCaseTest("runTest").run(testResult);
        new TestCaseTest("runTestMinus").run(testResult);
        testResult.printCount();
    }
}
```

main 메소드에서 ```TestResult```를 생성해서 각각의 테스트케이스에 파라미터로 전달합니다.  
이렇게 되면 테스트 케이스가 실행될때마다 카운트가 체크되니 전체 카운트를 확인할수 있습니다.  
마지막 ```printCount```를 통해 미리 선언된 방식으로 레포팅합니다.

![테스트성공6](./images/테스트성공6.png)

> printCount를 assertTrue로 검증하지 않는 이유는, 이부분은 검증의 영역이 아닌 레포팅 영역이기 때문입니다.  
차후에 본인이 원하는 방식으로 레포팅 형태롤 변경하면 됩니다.  
(ex: HTML, JSON 등등)  
이 부분도 나는 테스트 코드로 작성하고 싶다! 하시는 분들은 getCount와 같은 메소드를 생성하여 검증코드를 추가하셔도 무방합니다.   
지금은 간단하게 console로 표현하였습니다.

테스트 결과를 담을 객체도 완성하였습니다!  
여기까지를 다이어그램으로 표현하면 아래와 같습니다.

![스냅샷3](./images/스냅샷3.png)


[[ad]]

### 5. Fail 처리

현재 TestResult에는 큰 단점이 하나 있습니다.  

![틀린케이스](./images/틀린케이스.png)

위와 같이 일부러 첫번째 테스트 케이스를 실패시켰습니다.  
당연하게도 첫번째 테스트 케이스가 실패해서 ```AssertionFailedError```이 발생하고, **다른 모든 코드가 모두 작동이 중단**되었습니다.  
  
앞의 테스트가 실패하는것과 관계없이 다른 테스트케이스들은 모두 실행되어야 합니다.  
추가로, **테스트가 실패한 것인지/오류가 발생한것인지도 구분**할수 있어야 합니다.  
즉, ```AssertionFailedError```와 다른 Exception들 (ex: ```ArrayIndexOutOfBoundsException```)은 구분되어야 한다는 것입니다.
  
이 문제를 해결하기 위해 ```TestCase```에 ```try~catch```를 추가하겠습니다.

```java
public abstract class TestCase {
    ...

    public void run(TestResult testResult){
        testResult.startTest();
        before();
        try{
            runTestCase();
        } catch (InvocationTargetException ite) {
            if(isAssertionFailed(ite)){
                testResult.addFailure(this);
            } else {
                testResult.addError(this, ite);
            }
        } catch (Exception e) {
            testResult.addError(this, e);
        } finally {
            after();
        }
    }

    private boolean isAssertionFailed(InvocationTargetException ite) {
        return ite.getTargetException() instanceof AssertionFailedError;
    }

    ...

    private void runTestCase() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        logger.info("{} execute ", testCaseName); // 테스트 케이스들 구별을 위해 name 출력 코드
        Method method = this.getClass().getMethod(testCaseName, null);
        method.invoke(this, null);
    }

    ...
}
```

먼저 ```run``` 메소드에서 ```runTestCase()``` 실행부분을 ```try```~```catch```로 감싸고, ```catch```에서 ```InvocationTargetException```과 ```Exception```을 분리해서 제어합니다.  
  
여기서 ```InvocationTargetException```은 테스트가 실패했을때를 나타내는데, ```AssertionFailedError```로 바로 잡지 않은 이유는 ```method.invoke``` 때문입니다.  
  
저희가 만든 프레임워크에서 각각의 테스트 메소드 실행은 ```method.invoke```로 실행되는데 이것때문에 Exception이 발생할 경우 ```InvocationTargetException```로 랩핑되어 나가기 때문에 ```run()``` 메소드에선 진짜 ```InvocationTargetException``` 발생한것인지, ```AssertionFailedError```가 발생했는데 ```InvocationTargetException```로 랩핑된것인지 알수가 없습니다.  
그래서 ```InvocationTargetException```이 발생했을 경우, 실제 그 안에 있는 Exception이 ```AssertionFailedError```인지 확인하는 메소드 (```isAssertionFailed```)를 추가한것입니다.  
  
그외 다른 Exception에선 모두 Error로 간주하고 처리합니다.  
  
 ```TestCase``` 코드는 여기까지이며, 이제 ```TestResult``` 코드를 수정하겠습니다.  

```java
public class TestResult {

    ...

    private List<TestFailure> failures;
    private List<TestError> errors;

    public TestResult() {
        this.runTestCount = 0;
        this.failures = new ArrayList<>();
        this.errors = new ArrayList<>();
    }
    
    ...

    public synchronized void addFailure(TestCase testCase) {
        this.failures.add(new TestFailure(testCase));
    }

    public synchronized void addError(TestCase testCase, Exception e){
        this.errors.add(new TestError(testCase, e));
    }

    ...

    public void printCount(){
        logger.info("Total Test Count: {}", runTestCount);
        logger.info("Total Test Success Count: {}", runTestCount - failures.size() - errors.size());
        logger.info("Total Test Failure Count: {}", failures.size());
        logger.info("Total Test Error Count: {}", errors.size());
    }
}
```

 ```TestResult```코드의 수정은 간단합니다.  
테스트 실패와 오류발생에 대한 처리 메소드들만 추가된 것입니다.  
마지막 레포팅 메소드인 ```printCount```에선 성공한 테스트 수, 실패한 테스트수, 오류가발생한 테스트수를 차례로 Console에 출력시켜줍니다.  
  
마지막으로 테스트 실패를 나타내는 ```TestFailure``` 클래스와 테스트 오류를 나타내는 ```TestError``` 클래스를 생성하겠습니다. 

```java
public class TestFailure {
    private TestCase testCase;

    public TestFailure(TestCase testCase) {
        this.testCase = testCase;
    }

    public String getTestCaseName() {
        return testCase.getTestCaseName();
    }
}

public class TestError {
    private TestCase testCase;
    private Exception exception;

    public TestError(TestCase testCase, Exception exception) {
        this.testCase = testCase;
        this.exception = exception;
    }

    public String getTestCaseName() {
        return testCase.getTestCaseName();
    }

    public Exception getException() {
        return exception;
    }
}
```

 ```TestFailure```는 실패한 테스트케이스만 저장하지만, ```TestError```는 어떤 Exception이 발생했는지도 알 필요가 있다고 생각되어 필드에 추가하였습니다.  
  
자 그럼 이제 실제로 테스트가 잘 수행되는지 확인하겠습니다.  

![테스트성공7](./images/테스트성공7.png)

실패한 테스트인 ```runTest```는 ```Test Passed``` 메세지가 노출안되고, 최종 결과물에서 성공한 테스트 수와 실패한 테스트 수를 확인할 수 있습니다.  
  
 ```TestResult``` 는 차후에 필요에 의하면 ```HtmlTestResult```, ```JsonTestResult``` 등으로 확장할 수도 있으며, 출력시킬 데이터 형태도 단순 count 외에도 여러 데이터를 출력시킬 수 있습니다.  

### 6. TestSuite

지금까지 저희는 개별 테스트만 수행했습니다.  
실제 Main 메소드에서는 각각의 ```TestCase```가 한묶음의 테스트인지, 개별 테스트인지 전혀 구분할 수 없습니다.  
그냥 개별적으로 실행된것일 뿐입니다.  
  
하지만 좀 더 견고한 어플리케이션을 만들려면 테스트들을 그룹화 하고 그룹단위로 실행할 수 있어야만 합니다.  
그래서 ```TestCase```를 그룹화하여 관리할 수 있도록 기능을 개선해볼 예정입니다.  
  
여기서 중요한 점은, **개별 테스트케이스 단위로도 테스트는 수행할 수 있어야하며 그룹 단위로도 수행될수 있어야 한다**는 점입니다.  
그룹과 개별은 단위가 다른데 어떻게 하면 같은 인터페이스로 관리할 수 있을까요?  
  
바로 [Composite 패턴](https://ko.wikipedia.org/wiki/%EC%BB%B4%ED%8F%AC%EC%A7%80%ED%8A%B8_%ED%8C%A8%ED%84%B4)입니다.  
  
Composite 패턴의 구성요소는 아래와 같습니다.  

* Component: 테스트와 상호작용하는데 사용할 인터페이스
* Composite: Component 인터페이스를 구현하고 Leaf의 컬렉션을 관리
* Leaf: Component 인터페이스를 따르는 자식 (여기선 ```TestCase```)

자바에서 Composite을 적용할 때는 **추상 클래스가 아닌 인터페이스를 정의하는 것을 더 선호**합니다.  
인터페이스를 사용하면 Composite와 Leaf는 특정 클래스를 따르지 않아도 되고 그저 인터페이스를 따르기만 하면 됩니다.  
  
자 그럼 Composite 패턴의 Component(인터페이스) 역할을 할 ```Test``` 인터페이스를 생성하겠습니다.  

```java
public interface Test {
    void run(TestResult result);
}
```

제일 처음 도입한 패턴이 기억나시나요?  
Command 패턴을 도입하면서 이 프로젝트의 command는 ```run()```라고 얘기했습니다.  
마찬가지로 Composite 패턴의 Component에서 가질 메소드도 오직 ```run()```만 있으면 됩니다.  
이외 다른 메소드들은 인터페이스를 구분 지을 단위가 될만큼 중요한 요소가 아닙니다.  
  
자 그럼 이제 Composite 패턴의 Composite역할이자, 각각의 ```TestCase```를 관리할 클래스를 생성하겠습니다.  
해당 클래스의 이름은 ```TestSuite```라 하겠습니다.

```java
public class TestSuite implements Test {
    private List<TestCase> testCases = new ArrayList<>();

    @Override
    public void run(TestResult result) {
        for (TestCase testCase : this.testCases) {
            testCase.run(result);
        }
    }

    public void addTestCase(TestCase testCase) {
        this.testCases.add(testCase);
    }
}
```

간단한 코드이지만, ```Test``` 인터페이스를 구현하고, ```run(TestResult result)``` 메소드에선 Leaf 역할은 ```TestCase```에게 위임한 것입니다.  
이렇게 하면 외부의 클라이언트는 ```Test``` 인터페이스 스펙만으로 ```TestCase```, ```TestSuite```를 모두 다룰수 있게 됩니다.  
추가로 ```TestSuite```에서 ```TestCase```를 추가할 수 있도록 ```addTestCase```를 ```public```으로 생성하였습니다.  
  
동일한 인터페이스 스펙을 갖추기 위해 ```TestCase```도 ```Test``` 인터페이스를 구현(```implements```) 하도록 수정하겠습니다.

```java
public abstract class TestCase implements Test {
    ....
}
```

그럼 이제 테스트 코드를 ```TestSuite```단위로 변경해보겠습니다.  

```java
public class TestCaseTest extends TestCase {
    
    ...

    public static void main(String[] args) {
        TestSuite testSuite = new TestSuite();
        testSuite.addTestCase(new TestCaseTest("runTest"));
        testSuite.addTestCase(new TestCaseTest("runTestMinus"));

        TestResult testResult = new TestResult();
        testSuite.run(testResult);

        testResult.printCount();
    }
}
```

기존에 따로놀던 ```TestCase``` 코드들이 ```TestSuite``` 안에 포함되어 관리되고 실행되는 것을 확인할 수 있습니다.  

> 위 코드에선 동일한 테스트 클래스인 ```TestCaseTest```가 2번 호출되고 있습니다.  
좀 더 개선한다면 TestCase 구현 클래스들 내부에서 테스트 메소드들을 자동으로 추출하고 실행하도록 할수도 있겠죠?  
여기선 범위가 과하단 생각에 기존 코드를 살려두었습니다.

현재까지를 다이어그램으로 표현하면 아래와 같습니다.

![스냅샷6](./images/스냅샷6.png)

### 7. 정리

여기까지 오시느라 수고하셨습니다!  
저희가 만든 JUnit 프레임워크의 전체 다이어그램이 아래와 같이 확장되었습니다.  

![스냅샷7](./images/스냅샷7.png)

> 여기서 PluggableSelector & Adapter 패턴은 초반에 생성한 Method.invoke 부분이라고 보시면 됩니다.  
간단한 부분이라 초반에 생략하고 지나갔는데 좀 더 자세히 보고 싶으시다면 [[번역]JUnit A Cook’s Tour](https://bluepoet.me/2016/12/03/%EB%B2%88%EC%97%ADjunit-a-cooks-tour/) 의 3.4 챕터를 보시면 됩니다.

굉장히 많은 패턴이 적용되었습니다.  
하지만 저희가 기존에 객체지향적으로 코딩하기 위해 한번씩 고민해보고 해결한 방식이여서 크게 이해하는데 어렵지 않았으리라고 생각합니다.  
아래와 같이 차근차근 패턴을 적용하면서 많은 공부가 되었습니다.

![스토리보드](./images/스토리보드.png)

(스토리보드)  
  
이후에도 기능 추가가 필요하다면 과감하게 추가할 수 있습니다.  
기준이 되는 테스트 코드들이 이미 존재하고, 각각의 모듈들이 별도로 분리되어있기 때문입니다.  
  
사실 제가 한게 진짜 제대로 된것인지는 저도 잘모르겠습니다^^;  
다만 좀 더 객체지향과 디자인패턴을 이해하는데 이 방식이 정말 큰 도움이 되었다는 것입니다.  
  
이 글을 보시는 분들도 꼭 시간이 되신다면 직접 JUnit을 만들어보셨으면 합니다.  
기나긴 연휴가 슬슬 마무리 되어가는데, 다들 잘 마무리하시길 바랍니다.  
  
끝까지 읽어주셔서 고맙습니다.

## 참고

### 계기

* [테스팅 프레임워크는 직접 만들어 써보자 - 토비님 블로그](http://toby.epril.com/?p=424)
* [[번역]JUnit A Cook’s Tour](https://bluepoet.me/2016/12/03/%EB%B2%88%EC%97%ADjunit-a-cooks-tour/)

### 사용한 패턴

* [Command 패턴](http://javacan.tistory.com/entry/6)
* [Template Method 패턴](http://jdm.kr/blog/116)
* [Collecting Parameter 패턴](http://www.javajigi.net/display/SWD/Move+Accumulation+to+Collecting+Parameter)
* [Composite 패턴](https://ko.wikipedia.org/wiki/%EC%BB%B4%ED%8F%AC%EC%A7%80%ED%8A%B8_%ED%8C%A8%ED%84%B4)