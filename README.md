# brave-webmvc-example #

Example Spring Web MVC service implementation that shows the use of the client and server side interceptors from the
`brave-spring-web-servlet-interceptor`  and `brave-spring-resttemplate-interceptors` modules and the usage of [brave](https://github.com/kristofa/brave) in general.

## What is it about? ##

On one hand there is the service resource which can be found in following class: 
`brave.webmvc.ExampleController`

Next to that there is an integration test: `brave.webmvc.ITWebMvcExample` that sets up
an embedded Jetty server at port `8081` which deploys our service resource at context path http://localhost:8081.

The resource (WebExampleController) makes available 2 URI's

*   GET http://localhost:8081/a
*   GET http://localhost:8081/b


The test code (ITWebMvcExample) sets up our endpoint, does a http GET request to http://localhost:8081/a
The code that is triggered through
this URI will make a new call to the other URI: http://localhost:8081/b

For both requests our client and server side interceptors that use the Brave api are executed.  This results in 2 spans being logged.
The test uses `LoggingReporter` which simply logs the spans as json:

```
Dec 08, 2016 3:55:18 PM com.github.kristofa.brave.LoggingReporter report
INFO: {"traceId":"6551d3d189b43c84","id":"05fe4e63b5ce342e","name":"get","parentId":"6551d3d189b43c84","timestamp":1481183718619000,"duration":87378,"annotations":[{"timestamp":1481183718619000,"value":"cs","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}},{"timestamp":1481183718706378,"value":"cr","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}}],"binaryAnnotations":[{"key":"http.url","value":"http://localhost:8081/b","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}}]}
Dec 08, 2016 3:55:18 PM com.github.kristofa.brave.LoggingReporter report
INFO: {"traceId":"6551d3d189b43c84","id":"05fe4e63b5ce342e","name":"get","parentId":"6551d3d189b43c84","annotations":[{"timestamp":1481183718622000,"value":"sr","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}},{"timestamp":1481183718706000,"value":"ss","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}}],"binaryAnnotations":[{"key":"http.status_code","value":"200","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}},{"key":"http.url","value":"/b","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}}]}
Dec 08, 2016 3:55:18 PM com.github.kristofa.brave.LoggingReporter report
INFO: {"traceId":"6551d3d189b43c84","id":"6551d3d189b43c84","name":"get","timestamp":1481183717947000,"duration":823013,"annotations":[{"timestamp":1481183717947000,"value":"sr","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}},{"timestamp":1481183718770013,"value":"ss","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}}],"binaryAnnotations":[{"key":"http.status_code","value":"200","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}},{"key":"http.url","value":"/a","endpoint":{"serviceName":"brave-webmvc-example","ipv4":"192.168.1.10"}}]}
```

The spans are logged in reverse order:

1.  The last log line logs the client part of the first span the is initiated in the test code. 
    You notice that is has no parent spanid and it logs cs (client send), cr (client received) annotations as well as the http code return code as binary annotation.
2.  The 3rd log line logs the server part of the first span. So it has the same trace id, span id and null parent span id. 
    It logs sr (server received) and ss (server send) annotations.
3.  The 2nd log line logs the client request from URI a to b. 
    It has the same trace id but different span id and refers to the our first span as parent span id. 
    As it is a client request it again logs cs, cr and http return code annotations.
4.  The 1st log line logs the server side part of b. 
    So it has sr and ss annotations and same trace information as previous span log.

So the 2 server side logs are generated by having `ServletHandlerInterceptor` available.
The client side logs are generated by the `BraveClientHttpRequestInterceptor`.

## How is it all hooked together? ##

### src/main/webapp/WEB-INF/web.xml ###

We use Spring to wire everything together via web.xml.

We use the Spring DispatcherServlet and choose to use annotation based configuration (AnnotationConfigWebApplicationContext) instead of
XML configuration.

We set up our context by with the following configuration classes:

*   brave.webmvc.ExampleController : This sets up the web controller and rest template and has no tracing configuration
*   brave.webmvc.WebTracingConfiguration : This adds tracing by configuring brave, server and client tracing filters.

### Jetty ###

Jetty is embedded and started in the setup method of our test (ITWebMvcExample) and stopped in the tearDown method.

It is configured to use src/main/webapp/WEB-INF/web.xml so that is how Spring are being set up.

## Running against Zipkin ##

First, run [Zipkin](http://zipkin.io/), which stores and queries traces reported by the above services.

```bash
wget -O zipkin.jar 'https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec'
java -jar zipkin.jar
```

Now, edit WebTracingConfiguration, so that it sends spans to zipkin instead of the console.

```java
  @Bean Reporter<Span> reporter() {
    return AsyncReporter.builder(sender()).build();
  }
```

Next, from this directory, run `mvn verify`, which will run the test scenario (ITWebMvcExample)

Finally, browse zipkin for the traces the test created: http://localhost:9411/
