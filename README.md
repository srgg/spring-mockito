spring-mockito
==============

Significantly simplifies Spring bean mocking by providing [MockitoPropagatingFactoryPostProcessor](https://github.com/srgg/spring-mockito/blob/master/src/main/java/com/github/srgg/springmockito/MockitoPropagatingFactoryPostProcessor.java) that replace original beans with their mocks, therefore, as a result, Autowiring is working fine out of the box.

Simple step by step tutorial
============================

Let's assume that we have two beans:
   - errorHandler with method onError() which re-throws corresponding  exception,
   - integerConverter with method convert() which is returns either an integer value parsed from passed string or it calls autowired instance of errorHandler in case of error.

Here is a simple implementation of those beans:

```java

    public class ErrorHandler{
        public void onError(Throwable e) throws RuntimeException {
            throw new RuntimeException("Exception was generated upon request", e);
        }
    }

    public class IntegerConverter{
        @Autowired
        private ErrorHandler errorHandler;

        public int convert(String value){
            try {
                return Integer.parseInt(value);
            }catch (NumberFormatException e) {
                errorHandler.onError(e);
                return -1;
            }
        }
    }

```

and corresponding beans.xml:

```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    
        <bean id="errorHandler" class="org.test.ErrorHandler" />
        <bean id="integerConverter" class="org.test.IntegerConverter" />
    </beans>
```


Let's create a test to ensure that errorHandler will be called by integerConverter exactly one time in case of error. So, in Mockito's terms:

> we need to inject the mocked errorHandler and verify interactions count.

This might be done by adding MockitoPropagatingFactoryPostProcessor as follows:

```java

    @RunWith(SpringJUnit4ClassRunner.class)
    @ContextConfiguration(classes = AutowiredMockTest.MockedConfig.class)
    public class AutowiredMockTest {
        @Configuration
        @ImportResource("classpath:/beans.xml")  // Original configuration
        static class MockedConfig{
            @Mock
            private ExceptionGenerator exceptionGenerator;
    
            @Bean
            public ExceptionGenerator exceptionGenerator(){
                return exceptionGenerator;
            }
    
            @Bean
            public MockitoPropagatingFactoryPostProcessor postProcessor(){
                return new MockitoPropagatingFactoryPostProcessor(this);
            }
        }
    
        @Autowired
        private ExceptionGenerator eg;
    
        @Test
        public void mustNotRaiseExceptionIfItsWasMocked() throws Exception {
            assertNotNull("Spring doesn't properly configured", eg);
    
            // original bean always throws exception
            eg.generateException();
    
            verify(eg, Mockito.times(1)).generateException();
            Mockito.verifyNoMoreInteractions(eg);
        }
    }

```

