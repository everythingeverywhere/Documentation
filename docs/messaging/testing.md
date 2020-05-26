<a name="steeltoe-messaging-testing"></a>
# Testing Support

Writing integration for asynchronous applications is necessarily more complex than testing simpler applications.
This is made more complex when abstractions such as the `@RabbitListener` annotations come into the picture.
The question is how to verify that, after sending a message, the listener received the message as expected.

The framework itself has many unit and integration tests.
Some using mocks while, others use integration testing with a live RabbitMQ broker.
You can consult those tests for some ideas for testing scenarios.

<a name="steeltoe-messaging-spring-rabbit-test"></a>
## @SpringRabbitTest

Use this annotation to add infrastructure beans to the Spring test `ApplicationContext`.
This is not necessary when using, for example `@SpringBootTest` since Spring Boot's auto configuration will add the beans.

Beans that are registered are:

* `CachingConnectionFactory` (`autoConnectionFactory`). If `@RabbitEnabled` is present, its connectionn factory is used.
* `RabbitTemplate` (`autoRabbitTemplate`)
* `RabbitAdmin` (`autoRabbitAdmin`)
* `RabbitListenerContainerFactory` (`autoContainerFactory`)

In addition, the beans associated with `@EnableRabbit` (to support `@RabbitListener`) are added, as the following JUnit5 example shows:

```Java
@SpringJunitConfig
@SpringRabbitTest
public class MyRabbitTests {

	@Autowired
	private RabbitTemplate template;

	@Autowired
	private RabbitAdmin admin;

	@Autowired
	private RabbitListenerEndpointRegistry registry;

	@Test
	void test() {
        ...
	}

	@Configuration
	public static class Config {

        ...

	}

}
```

With JUnit4, replace `@SpringJunitConfig` with `@RunWith(SpringRunnner.class)`.

<a name="steeltoe-messating-mockito-answer"></a>
## Mockito `Answer<?>` Implementations

There are currently two `Answer<?>` implementations to help with testing.

The first, `LatchCountDownAndCallRealMethodAnswer`, provides an `Answer<Void>` that returns `null` and counts down a latch.
The following example shows how to use `LatchCountDownAndCallRealMethodAnswer`:

```Java
LatchCountDownAndCallRealMethodAnswer answer = new LatchCountDownAndCallRealMethodAnswer(2);
doAnswer(answer)
    .when(listener).foo(anyString(), anyString());

...

assertTrue(answer.getLatch().await(10, TimeUnit.SECONDS));
```

The second, `LambdaAnswer<T>`, provides a mechanism to optionally call the real method and provides an opportunity
to return a custom result, based on the `InvocationOnMock` and the result (if any).

Consider the following POJO:

```Java
public class Thing {

    public String thing(String thing) {
        return thing.toUpperCase();
    }

}
```

The following class tests the `Thing` POJO:

```Java
Thing thing = spy(new Thing());

doAnswer(new LambdaAnswer<String>(true, (i, r) -> r + r))
    .when(thing).thing(anyString());
assertEquals("THINGTHING", thing.thing("thing"));

doAnswer(new LambdaAnswer<String>(true, (i, r) -> r + i.getArguments()[0]))
    .when(thing).thing(anyString());
assertEquals("THINGthing", thing.thing("thing"));

doAnswer(new LambdaAnswer<String>(false, (i, r) ->
    "" + i.getArguments()[0] + i.getArguments()[0])).when(thing).thing(anyString());
assertEquals("thingthing", thing.thing("thing"));
```

Starting with version 2.2.3, the answers capture any exceptions thrown by the method under test.
Use `answer.getExceptions()` to get a reference to them.


<a name="steeltoe-messaging-test-harness"></a>
## `@RabbitListenerTest` and `RabbitListenerTestHarness`

Annotating one of your `@Configuration` classes with `@RabbitListenerTest` causes the framework to replace the
standard `RabbitListenerAnnotationBeanPostProcessor` with a subclass called `RabbitListenerTestHarness` (it also enables
`@RabbitListener` detection through `@EnableRabbit`).

The `RabbitListenerTestHarness` enhances the listener in two ways.
First, it wraps the listener in a `Mockito Spy`, enabling normal `Mockito` stubbing and verification operations.
It can also add an `Advice` to the listener, enabling access to the arguments, result, and any exceptions that are thrown.
You can control which (or both) of these are enabled with attributes on the `@RabbitListenerTest`.
The latter is provided for access to lower-level data about the invocation.
It also supports blocking the test thread until the async listener is called.

>IMPORTANT: `final` `@RabbitListener` methods cannot be spied or advised.
Also, only listeners with an `id` attribute can be spied or advised.

Consider some examples.

The following example uses spy:

```Java
@Configuration
@RabbitListenerTest
public class Config {

    @Bean
    public Listener listener() {
        return new Listener();
    }

    ...

}

public class Listener {

    @RabbitListener(id="foo", queues="#{queue1.name}")
    public String foo(String foo) {
        return foo.toUpperCase();
    }

    @RabbitListener(id="bar", queues="#{queue2.name}")
    public void foo(@Payload String foo, @Header("amqp_receivedRoutingKey") String rk) {
        ...
    }

}

public class MyTests {

    @Autowired
    private RabbitListenerTestHarness harness; <1>

    @Test
    public void testTwoWay() throws Exception {
        assertEquals("FOO", this.rabbitTemplate.convertSendAndReceive(this.queue1.getName(), "foo"));

        Listener listener = this.harness.getSpy("foo"); <2>
        assertNotNull(listener);
        verify(listener).foo("foo");
    }

    @Test
    public void testOneWay() throws Exception {
        Listener listener = this.harness.getSpy("bar");
        assertNotNull(listener);

        LatchCountDownAndCallRealMethodAnswer answer = new LatchCountDownAndCallRealMethodAnswer(2); <3>
        doAnswer(answer).when(listener).foo(anyString(), anyString()); <4>

        this.rabbitTemplate.convertAndSend(this.queue2.getName(), "bar");
        this.rabbitTemplate.convertAndSend(this.queue2.getName(), "baz");

        assertTrue(answer.getLatch().await(10, TimeUnit.SECONDS));
        verify(listener).foo("bar", this.queue2.getName());
        verify(listener).foo("baz", this.queue2.getName());
    }

}
```

<1> Inject the harness into the test case so we can get access to the spy.

<2> Get a reference to the spy so we can verify it was invoked as expected.
Since this is a send and receive operation, there is no need to suspend the test thread because it was already
suspended in the `RabbitTemplate` waiting for the reply.

<3> In this case, we're only using a send operation so we need a latch to wait for the asynchronous call to the listener
on the container thread.
We use one of the link:#mockito-answer[Answer<?>] implementations to help with that.

<4> Configure the spy to invoke the `Answer`.

The following example uses the capture advice:

```Java
@Configuration
@ComponentScan
@RabbitListenerTest(spy = false, capture = true)
public class Config {

}

@Service
public class Listener {

    private boolean failed;

    @RabbitListener(id="foo", queues="#{queue1.name}")
    public String foo(String foo) {
        return foo.toUpperCase();
    }

    @RabbitListener(id="bar", queues="#{queue2.name}")
    public void foo(@Payload String foo, @Header("amqp_receivedRoutingKey") String rk) {
        if (!failed && foo.equals("ex")) {
            failed = true;
            throw new RuntimeException(foo);
        }
        failed = false;
    }

}

public class MyTests {

    @Autowired
    private RabbitListenerTestHarness harness; <1>

    @Test
    public void testTwoWay() throws Exception {
        assertEquals("FOO", this.rabbitTemplate.convertSendAndReceive(this.queue1.getName(), "foo"));

        InvocationData invocationData =
            this.harness.getNextInvocationDataFor("foo", 0, TimeUnit.SECONDS); <2>
        assertThat(invocationData.getArguments()[0], equalTo("foo"));     <3>
        assertThat((String) invocationData.getResult(), equalTo("FOO"));
    }

    @Test
    public void testOneWay() throws Exception {
        this.rabbitTemplate.convertAndSend(this.queue2.getName(), "bar");
        this.rabbitTemplate.convertAndSend(this.queue2.getName(), "baz");
        this.rabbitTemplate.convertAndSend(this.queue2.getName(), "ex");

        InvocationData invocationData =
            this.harness.getNextInvocationDataFor("bar", 10, TimeUnit.SECONDS); <4>
        Object[] args = invocationData.getArguments();
        assertThat((String) args[0], equalTo("bar"));
        assertThat((String) args[1], equalTo(queue2.getName()));

        invocationData = this.harness.getNextInvocationDataFor("bar", 10, TimeUnit.SECONDS);
        args = invocationData.getArguments();
        assertThat((String) args[0], equalTo("baz"));

        invocationData = this.harness.getNextInvocationDataFor("bar", 10, TimeUnit.SECONDS);
        args = invocationData.getArguments();
        assertThat((String) args[0], equalTo("ex"));
        assertEquals("ex", invocationData.getThrowable().getMessage()); <5>
    }

}
```

<1> Inject the harness into the test case so we can get access to the spy.

<2> Use `harness.getNextInvocationDataFor()` to retrieve the invocation data - in this case since it was a request/reply
scenario there is no need to wait for any time because the test thread was suspended in the `RabbitTemplate` waiting
for the result.

<3> We can then verify that the argument and result was as expected.

<4> This time we need some time to wait for the data, since it's an async operation on the container thread and we need
to suspend the test thread.

<5> When the listener throws an exception, it is available in the `throwable` property of the invocation data.
====

<a name="steeltoe-messaging-test-template"></a>
## Using `TestRabbitTemplate`

The `TestRabbitTemplate` is provided to perform some basic integration testing without the need for a broker.
When you add it as a `@Bean` in your test case, it discovers all the listener containers in the context, whether declared as `@Bean` or `<bean/>` or using the `@RabbitListener` annotation.
It currently only supports routing by queue name.
The template extracts the message listener from the container and invokes it directly on the test thread.
Request-reply messaging (`sendAndReceive` methods) is supported for listeners that return replies.

The following test case uses the template:

```Java
@RunWith(SpringRunner.class)
public class TestRabbitTemplateTests {

    @Autowired
    private TestRabbitTemplate template;

    @Autowired
    private Config config;

    @Test
    public void testSimpleSends() {
        this.template.convertAndSend("foo", "hello1");
        assertThat(this.config.fooIn, equalTo("foo:hello1"));
        this.template.convertAndSend("bar", "hello2");
        assertThat(this.config.barIn, equalTo("bar:hello2"));
        assertThat(this.config.smlc1In, equalTo("smlc1:"));
        this.template.convertAndSend("foo", "hello3");
        assertThat(this.config.fooIn, equalTo("foo:hello1"));
        this.template.convertAndSend("bar", "hello4");
        assertThat(this.config.barIn, equalTo("bar:hello2"));
        assertThat(this.config.smlc1In, equalTo("smlc1:hello3hello4"));

        this.template.setBroadcast(true);
        this.template.convertAndSend("foo", "hello5");
        assertThat(this.config.fooIn, equalTo("foo:hello1foo:hello5"));
        this.template.convertAndSend("bar", "hello6");
        assertThat(this.config.barIn, equalTo("bar:hello2bar:hello6"));
        assertThat(this.config.smlc1In, equalTo("smlc1:hello3hello4hello5hello6"));
    }

    @Test
    public void testSendAndReceive() {
        assertThat(this.template.convertSendAndReceive("baz", "hello"), equalTo("baz:hello"));
    }

    @Configuration
    @EnableRabbit
    public static class Config {

        public String fooIn = "";

        public String barIn = "";

        public String smlc1In = "smlc1:";

        @Bean
        public TestRabbitTemplate template() throws IOException {
            return new TestRabbitTemplate(connectionFactory());
        }

        @Bean
        public ConnectionFactory connectionFactory() throws IOException {
            ConnectionFactory factory = mock(ConnectionFactory.class);
            Connection connection = mock(Connection.class);
            Channel channel = mock(Channel.class);
            willReturn(connection).given(factory).createConnection();
            willReturn(channel).given(connection).createChannel(anyBoolean());
            given(channel.isOpen()).willReturn(true);
            return factory;
        }

        @Bean
        public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory() throws IOException {
            SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
            factory.setConnectionFactory(connectionFactory());
            return factory;
        }

        @RabbitListener(queues = "foo")
        public void foo(String in) {
            this.fooIn += "foo:" + in;
        }

        @RabbitListener(queues = "bar")
        public void bar(String in) {
            this.barIn += "bar:" + in;
        }

        @RabbitListener(queues = "baz")
        public String baz(String in) {
            return "baz:" + in;
        }

        @Bean
        public SimpleMessageListenerContainer smlc1() throws IOException {
            SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory());
            container.setQueueNames("foo", "bar");
            container.setMessageListener(new MessageListenerAdapter(new Object() {

                @SuppressWarnings("unused")
                public void handleMessage(String in) {
                    smlc1In += in;
                }

            }));
            return container;
        }

    }

}
```

<a name="steeltoe-messaging-junit-rules"></a>
## JUnit4 `@Rules`

Spring AMQP version 1.7 and later provide an additional jar called `spring-rabbit-junit`.
This jar contains a couple of utility `@Rule` instances for use when running JUnit4 tests.
See <a href="#steeltoe-messaging-junit5-conditions"></a> for JUnit5 testing.

### Using `BrokerRunning`

`BrokerRunning` provides a mechanism to let tests succeed when a broker is not running (on `localhost`, by default).

It also has utility methods to initialize and empty queues and delete queues and exchanges.

The following example shows its usage:

```Java
@ClassRule
public static BrokerRunning brokerRunning = BrokerRunning.isRunningWithEmptyQueues("foo", "bar");

@AfterClass
public static void tearDown() {
    brokerRunning.removeTestQueues("some.other.queue.too") // removes foo, bar as well
}
```

There are several `isRunning...` static methods, such as `isBrokerAndManagementRunning()`, which verifies the broker has the management plugin enabled.

<a name="steeltoe-messaging-brokerRunning-configure"></a>
====== Configuring the Rule

There are times when you want tests to fail if there is no broker, such as a nightly CI build.
To disable the rule at runtime, set an environment variable called `RABBITMQ_SERVER_REQUIRED` to `true`.

You can override the broker properties, such as hostname with either setters or environment variables:

The following example shows how to override properties with setters:


```Java
@ClassRule
public static BrokerRunning brokerRunning = BrokerRunning.isRunningWithEmptyQueues("foo", "bar");

static {
    brokerRunning.setHostName("10.0.0.1")
}

@AfterClass
public static void tearDown() {
    brokerRunning.removeTestQueues("some.other.queue.too") // removes foo, bar as well
}
```

You can also override properties by setting the following environment variables:

```Java
public static final String BROKER_ADMIN_URI = "RABBITMQ_TEST_ADMIN_URI";
public static final String BROKER_HOSTNAME = "RABBITMQ_TEST_HOSTNAME";
public static final String BROKER_PORT = "RABBITMQ_TEST_PORT";
public static final String BROKER_USER = "RABBITMQ_TEST_USER";
public static final String BROKER_PW = "RABBITMQ_TEST_PASSWORD";
public static final String BROKER_ADMIN_USER = "RABBITMQ_TEST_ADMIN_USER";
public static final String BROKER_ADMIN_PW = "RABBITMQ_TEST_ADMIN_PASSWORD";
```

These environment variables override the default settings (`localhost:5672` for amqp and `http://localhost:15672/api/` for the management REST API).

Changing the host name affects both the `amqp` and `management` REST API connection (unless the admin uri is explicitly set).

`BrokerRunning` also provides a `static` method called `setEnvironmentVariableOverrides` that lets you can pass in a map containing these variables.
They override system environment variables.
This might be useful if you wish to use different configuration for tests in multiple test suites.

>IMPORTANT: The method must be called before invoking any of the `isRunning()` static methods that create the rule instance.
Variable values are applied to all instances created after this invocation.
Invoke `clearEnvironmentVariableOverrides()` to reset the rule to use defaults (including any actual environment variables).

In your test cases, you can use those properties when creating the connection factory.
The following example shows how to do so:

```Java
@Bean
public ConnectionFactory rabbitConnectionFactory() {
    CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
    connectionFactory.setHost(brokerRunning.getHostName());
    connectionFactory.setPort(brokerRunning.getPort());
    connectionFactory.setUsername(brokerRunning.getUser());
    connectionFactory.setPassword(brokerRunning.getPassword());
    return connectionFactory;
}
```

### Using `LongRunningIntegrationTest`

`LongRunningIntegrationTest` is a rule that disables long running tests.
You might want to use this on a developer system but ensure that the rule is disabled on, for example, nightly CI builds.

The following example shows its usage:

```Java
@Rule
public LongRunningIntegrationTest longTests = new LongRunningIntegrationTest();
```

To disable the rule at runtime, set an environment variable called `RUN_LONG_INTEGRATION_TESTS` to `true`.

<a name="junit5-conditions"></a>
## JUnit5 Conditions

Version 2.0.2 introduced support for JUnit5.

### Using the `@RabbitAvailable` Annotation

This class-level annotation is similar to the `BrokerRunning` `@Rule` discussed in <a href="#steeltoe-messaging-junit-rules"></a>.
It is processed by `RabbitAvailableCondition`.

The annotation has three properties:

* `queues`: An array of queues that are declared (and purged) before each test and deleted when all tests are complete.
* `management`: Set this to `true` if your tests also require the management plugin installed on the broker.
* `purgeAfterEach`: (Since version 2.2) when `true` (default), the `queues` are purged between tests.

It is used to check whether the broker is available and skip the tests if not.
As discussed in <a href="#steeltoe-messaging-brokerRunning-configure"></a>, the environment variable called `RABBITMQ_SERVER_REQUIRED`, if `true`, causes the tests to fail fast if there is no broker.
You can configure the condition by using environment variables as discussed in <a href="#steeltoe-messaging-brokerRunning-configure"></a>.

In addition, the `RabbitAvailableCondition` supports argument resolution for parameterized test constructors and methods.
Two argument types are supported:

* `BrokerRunningSupport`: The instance (before 2.2, this was a JUnit 4 `BrokerRunning` instance)
* `ConnectionFactory`: The `BrokerRunningSupport` instance's RabbitMQ connection factory

The following example shows both:

```Java
@RabbitAvailable(queues = "rabbitAvailableTests.queue")
public class RabbitAvailableCTORInjectionTests {

    private final ConnectionFactory connectionFactory;

    public RabbitAvailableCTORInjectionTests(BrokerRunningSupport brokerRunning) {
        this.connectionFactory = brokerRunning.getConnectionFactory();
    }

    @Test
    public void test(ConnectionFactory cf) throws Exception {
        assertSame(cf, this.connectionFactory);
        Connection conn = this.connectionFactory.newConnection();
        Channel channel = conn.createChannel();
        DeclareOk declareOk = channel.queueDeclarePassive("rabbitAvailableTests.queue");
        assertEquals(0, declareOk.getConsumerCount());
        channel.close();
        conn.close();
    }

}
```

The preceding test is in the framework itself and verifies the argument injection and that the condition created the queue properly.

A practical user test might be as follows:

```Java
@RabbitAvailable(queues = "rabbitAvailableTests.queue")
public class RabbitAvailableCTORInjectionTests {

    private final CachingConnectionFactory connectionFactory;

    public RabbitAvailableCTORInjectionTests(BrokerRunningSupport brokerRunning) {
        this.connectionFactory =
            new CachingConnectionFactory(brokerRunning.getConnectionFactory());
    }

    @Test
    public void test() throws Exception {
        RabbitTemplate template = new RabbitTemplate(this.connectionFactory);
        ...
    }
}
```

When you use a Spring annotation application context within a test class, you can get a reference to the condition's connection factory through a static method called `RabbitAvailableCondition.getBrokerRunning()`.

>IMPORTANT: Starting with version 2.2, `getBrokerRunning()` returns a `BrokerRunningSupport` object; previously, the JUnit 4 `BrokerRunnning` instance was returned.
The new class has the same API as `BrokerRunning`.

The following test comes from the framework and demonstrates the usage:

```Java
@RabbitAvailable(queues = {
        RabbitTemplateMPPIntegrationTests.QUEUE,
        RabbitTemplateMPPIntegrationTests.REPLIES })
@SpringJUnitConfig
@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
public class RabbitTemplateMPPIntegrationTests {

    public static final String QUEUE = "mpp.tests";

    public static final String REPLIES = "mpp.tests.replies";

    @Autowired
    private RabbitTemplate template;

    @Autowired
    private Config config;

    @Test
    public void test() {

        ...

    }

    @Configuration
    @EnableRabbit
    public static class Config {

        @Bean
        public CachingConnectionFactory cf() {
            return new CachingConnectionFactory(RabbitAvailableCondition
                    .getBrokerRunning()
                    .getConnectionFactory());
        }

        @Bean
        public RabbitTemplate template() {

            ...

        }

        @Bean
        public SimpleRabbitListenerContainerFactory
                            rabbitListenerContainerFactory() {

            ...

        }

        @RabbitListener(queues = QUEUE)
        public byte[] foo(byte[] in) {
            return in;
        }

    }

}
```

### Using the `@LongRunning` Annotation

Similar to the `LongRunningIntegrationTest` JUnit4 `@Rule`, this annotation causes tests to be skipped unless an environment variable (or system property) is set to `true`.
The following example shows how to use it:

```Java
@RabbitAvailable(queues = SimpleMessageListenerContainerLongTests.QUEUE)
@LongRunning
public class SimpleMessageListenerContainerLongTests {

    public static final String QUEUE = "SimpleMessageListenerContainerLongTests.queue";

...

}
```

By default, the variable is `RUN_LONG_INTEGRATION_TESTS`, but you can specify the variable name in the annotation's `value` attribute.