# Metrics and Tracing: Better Together

You've decided to put your talents to work in the service of humanity and - in the age of the pandemic, and having no other real skills to speak of besides software - you're going to build a web service that people can check for the availability of the highly vaunted Playstation 5 video game console, on your new website, `www.ps5ownersarebetterpeople.com.net`.

## It All Started so Auspiciously...
Go to the trusty [Spring Initializr](https://start.spring.io) and generate a new project (called `service`) using the latest version of Java (_natch_!) and add the `Reactive Web`, `Wavefront`, `Lombok`, `Sleuth`, and `Actuator` dependencies to the project. Click the `Generate` button to download a `.zip` file containing the code for a project you should open in your favorite IDE. 

Add the following configuration values to `application.properties`. We'll review them as they become relevant. For now, the critical thing to keep in mind is that we're running the `service` on port `8083`, and we've given this service a name with the `spring.application.name` property, `service`.

https://github.com/shakuzen/console-availability/tree/master/basics/service/src/main/resources/application.properties 
```properties
spring.application.name=service
server.port=8083
wavefront.application.name=console-availability
management.metrics.export.wavefront.source=my-cloud-server
```

Here's the Java code.

https://github.com/shakuzen/console-availability/blob/master/basics/service/src/main/java/server/ServiceApplication.java
```java
@Slf4j
@SpringBootApplication
public class ServiceApplication {

    public static void main(String[] args) {
        log.info("starting server");
        SpringApplication.run(ServiceApplication.class, args);
    }
}

@RestController
class AvailabilityController {

    private boolean validate(String console) {
        return StringUtils.hasText(console) &&
               Set.of("ps5", "ps4", "switch", "xbox").contains(console);
    }

    @GetMapping("/availability/{console}")
    Map<String, Object> getAvailability(@PathVariable String console) {
        return Map.of("console", console,
                "available", checkAvailability(console));
    }

    private boolean checkAvailability(String console) {
        Assert.state(validate(console), () -> "the console specified, " + console + ", is not valid.");
        return switch (console) {
            case "ps5" -> throw new RuntimeException("Service exception");
            case "xbox" -> true;
            default -> false;
        };
    }
}
```

Given a request for a particular type of console (`ps5`, `nintendo`, `xbox`, `ps4`), the API returns the console's availability (presumably sourced from local electronics outlets). Except for some reason, which for our demo will have to be _deus ex machina_ - there is no PlayStation 5 availability. Worse, the service itself is experiencing errors and blowing chunks every time someone even dares to inquire about the Playstation 5! We'll use this particular code path - asking about the availability of Playstation 5's in particular - to simulate an error in our system. Don't judge. You've probably made an error at some point, too. Probably. 

We want as much information as possible about individual microservices and their interactions, and we will most want that information when we're trying to flush out bugs in the system. Let's see how tracing and metrics can work together to provide an observability posture superior to using either metrics or tracing alone. 

We'll need a client to talk to the service and drive some traffic to it. Return to the [Spring Initializr](https://start.spring.io) and generate another project precisely like the `service`, but give this one the `spring.application.name` value of `client`. 

Here's the configuration file. 

https://github.com/shakuzen/console-availability/blob/master/basics/client/src/main/resources/application.properties 
```properties
spring.application.name=client
wavefront.application.name=console-availability
management.metrics.export.wavefront.source=my-cloud-server
```

The code uses the reactive, non-blocking `WebClient` to issue requests against the service. The entire application - both `client` and `service` - use reactive, non-blocking HTTP. You could just as easily use the traditional Servlet-based Spring MVC. Or you could avoid HTTP altogether and use messaging technologies. Or you could use both. Here's the Java code. 

https://github.com/shakuzen/console-availability/blob/master/basics/client/src/main/java/client/ClientApplication.java
```java
@Slf4j
@SpringBootApplication
public class ClientApplication {

    public static void main(String[] args) {
        log.info("starting client");
        SpringApplication.run(ClientApplication.class, args);
    }

    @Bean
    WebClient webClient(WebClient.Builder builder) {
        return builder.build();
    }

    @Bean
    ApplicationListener<ApplicationReadyEvent> ready(AvailabilityClient client) {
        return applicationReadyEvent -> {
            for (var console : "ps5,xbox,ps4,switch".split(",")) {
                Flux.range(0, 20).delayElements(Duration.ofMillis(100)).subscribe(i ->
                        client
                                .checkAvailability(console)
                                .subscribe(availability ->
                                        log.info("console: {}, availability: {} ", console, availability.isAvailable())));
            }
        };
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
class Availability {
    private boolean available;
    private String console;
}

@Component
@RequiredArgsConstructor
class AvailabilityClient {

    private final WebClient webClient;
    private static final String URI = "http://localhost:8083/availability/{console}";

    Mono<Availability> checkAvailability(String console) {
        return this.webClient
                .get()
                .uri(URI, console)
                .retrieve()
                .bodyToMono(Availability.class)
                .onErrorReturn(new Availability(false, console));
    }

}
```

Start the `service` application, and then start the `client` application. The `client` application generates a lot of demand directed at the service, some of which results in failed requests. We want to capture all of that information. 

## A Number By Any Other Name 

First, we want to have an aggregate view of all the data, _metrics_, that give us statistics about all the requests. Metrics are numbers, aggregations. Metrics can cover things like memory/thread usage, garbage collection, process metrics, etc. They also often encompass key performance indicators that the business might set, like how many orders were fulfilled, how many users authenticated, etc.

The `Actuator` starter, in turn, brings in [Micrometer](https://micrometer.io), which provides a simple facade over the instrumentation clients for the most popular monitoring systems, allowing you to instrument your JVM-based application code without vendor lock-in. Think SLF4J, but for metrics. 

The most straightforward possible use of Micrometer is to capture metrics and keep them in memory, which the Spring Boot Actuator will do. You can configure your application to show those metrics under an Actuator management endpoint - `/actuator/metrics/`. More commonly, though, you'll want to send these metrics to a time-series database like Graphite, Prometheus, Netflix Atlas, Datadog, or InfluxDB. Time series databases   store the evolving value of a metric over time, so you can see how it has changed.

## Follow the Data

Danger is the snack food of a true sleuth. -Mac Barnett

We also want to have detailed breakdowns of individual requests and traces to give us context around particular failed requests. The `Sleuth` starter brings in the [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth) distributed tracing abstraction, which provides a simple facade over distributed tracing systems like OpenZipkin and Google Cloud Stackdriver Trace and Wavefront. 

Micrometer and Sleuth give you the power of choice in metrics and tracing backends. We _could_ use these two different abstractions and separately stand up a dedicated cluster for our tracing and metrics aggregation systems. People do. Crazier things have happened. We're all about the idea that you shouldn't run what you can't charge for, so let's use an easy, turnkey, hosted, software-as-a-service (SaaS) offering and let someone else do that work. We don't envy the integration task that having such highly related data living in two different, unrelated backend systems otherwise implies. 

## To The Observability Cave, Statsman!

We will use VMware Tanzu's [excellent Wavefront observability platform](https://tanzu.vmware.com/observability), which understands both metrics and traces and can link them together for us. We already added the `Wavefront` starter to our build.

Start the `service` and then start the `client`. The `client` will generate a lot of traffic. Well, not a _lot_. Remember, [Reddit](https://en.wikipedia.org/wiki/Reddit) uses Wavefront successfully at their global scale. So, all things being equal, our data is _nothing_. But it's enough to see some core concepts in action. A Wavefront URL is printed out when our Spring Boot application starts up. This is the URL to access the _freemium_ Wavefront cluster. You already have a valid Wavefront configuration, and you didn't even have to sign up for an account! It will take a minute for metrics to publish to Wavefront. Wait for a minute and then visit the URL printed in the console in your browser.

That URL dumps you into the Wavefront dashboard for Spring Boot. There's a lot here, so we'll focus on a few key things in particular. 

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/1.png" width = "500" />

You can see that Wavefront comes fully loaded with a `Spring Boot` dashboard in the `Dashboards` menu at the top of the screen. The top of the dashboard shows that the `Source` is `my-cloud-server`, which comes from the configuration property `management.
.export.wavefront.source` (or use the default, which is the hostname of the machine). The `Application` we're interested in is `console-availability`, which comes from the configuration property `wavefront.application.name`.  _Application_  refers to the logical group of Spring Boot microservices, not any specific one. 

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/2.png" width = "500" />

Click on that, and you'll see everything about your application at a glance.  You can choose to look at information for either module - `client`  or `service`. Click `Jump To` to navigate to a particular set of graphs. We're interested in the data in the `HTTP` section. 


<img src = "https://github.com/shakuzen/console-availability/raw/master/images/4.png" width = "500" />

You can see a useful thing like the `Top Requests`, the `Top Failed Requests`, and `Top Exceptions` encountered in the code - mouse over a particular type of request to get some details associated with each entry. You can get information like the HTTP method (`GET`), service (`service`), status code (`500`), and URI (`/availability/{console}`) associated with the failing requests.

These kinds of at-a-glance numbers are metrics. Metrics are not based on sampled data; they're an aggregation of every single request. You should use metrics for your alerting because they ensure that you see _all_ your requests (and _all_ the errors, slow requests, etc.). Trace data, on the other hand, usually needs to be sampled at high volumes of traffic because the amount of data increases proportionally to the traffic volume.

We can see that the metrics collection has ignored the values for the `{console}` path variable in distinguishing the requests, which means that - as far as our data is concerned  - there's only one URI (`/availability/{console}`). This is by design. `{console}` is a path variable we're using to specify the console, but it could just as easily have been a user ID, an order ID, or some other thing for which there are likely many, possibly unbounded, values. It would be dangerous for the metrics system to record high cardinality metrics by default. Bounded cardinality metrics are cheap! Cost does not increase with traffic. Watch for cardinality in your metrics.

It is a little unfortunate because even though we know that `{console}` is a low-cardinality variable - there's a finite set of possible values - we can't drill down into the data any further to see at a glance which paths are failing. Metrics represent aggregated statistics, so even if we broke down the metrics in terms of the `{console}` variable, the metrics still lacks context around individual requests.

There's always the trace data! Click on the little breadcrumb/sandwich icon to the right of the words `Top Failed Requests`, and find the service by going to `Traces` > `console-availability`. 

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/5.png" width = "500" />

Here are all the traces collected for the application: good, bad, or otherwise. 

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/6.png" width = "500" />

Let's drill down into only the errant requests by adding an `Error` filter to the search. Then click `Search`. Now we can scrutinize individual errant requests. You can see how long was taken in each service call, the relationship between services, and where the error originated.

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/8.png" width = "500" />

Click on the `Expand` icon for the panel labeled `client: GET` in the screen's bottom right. You can see each hop in the request's journey: how long it took, the trace ID, the URL, and the path.

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/9.png" width = "500" />

Expand the `Tags` branch under a particular segment of the trace, and you can see the metadata collected on your behalf, automatically, by Spring Cloud Sleuth. A trace is made up of individual segments, called _spans_, that describe one hop in the request's journey.

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/10.png" width = "500" />

## Enriching the Data with Business/Domain Context 

We got a lot out of the default configuration. We didn't really do anything to the code to get the results we just saw except for adding the Spring Boot `Actuator` starter, the `Wavefront` starter, and the `Sleuth` starter and starting the application. See? It was easy! Very easy. As easy as falling off a log...ging-centric system and onto a real observability platform, even. We got trace information and metrics and a dashboard that we could consult for the details. We made precisely zero changes to our Java code to support any of this.

Let's take things a bit further and customize the metadata captured by Spring Cloud Sleuth and Micrometer to make it even easier to drill down by a domain-specific concept: the type of console requested. We can do this with the `{console}` path variable. The code already validates that the console's value falls within a set of well-known consoles. It is important that we validate the input before using it, which ensures the type of console is low cardinality. You should not use arbitrary input (like a path variable or query parameter) that could be high cardinality as a metric tag - though you could use high cardinality data as a trace tag. Now, instead of gleaning the type of console from the HTTP path in the trace data, we can use a tag on the metrics and traces.

We'll update the service to inject a `SpanCustomizer` to customize the trace information. We'll also update the service to configure a `WebFluxTagsContributor` to customize the tags captured by Spring Boot and given to Micrometer. Here's the new and updated code.

https://github.com/shakuzen/console-availability/blob/master/enhanced/service/src/main/java/server/ServiceApplication.java
```java
@Slf4j
@SpringBootApplication
public class ServiceApplication {

    @Bean
    WebFluxTagsContributor consoleTagContributor() {
        return (exchange, ex) -> {
            var console = "UNKNOWN";
            var consolePathVariable = ((Map<String,String>) exchange.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE)).get("console");
            if (AvailabilityController.validateConsole(consolePathVariable)) {
                console = consolePathVariable;
            }
            return Tags.of("console", console);
        };
    }

    public static void main(String[] args) {
        log.info("starting server");
        SpringApplication.run(ServiceApplication.class, args);
    }
}

@RestController
@AllArgsConstructor
class AvailabilityController {

    private final SpanCustomizer spanCustomizer;

    @GetMapping("/availability/{console}")
    Map<String, Object> getAvailability(@PathVariable String console) {
        Assert.state(validateConsole(console), () -> "the console specified, " + console + ", is not valid.");
        this.spanCustomizer.tag("console", console);
        return Map.of("console", console, "available", checkAvailability(console));
    }

    private boolean checkAvailability(String console) {
        return switch (console) {
            case "ps5" -> throw new RuntimeException("Service exception");
            case "xbox" -> true;
            default -> false;
        };
    }

    static boolean validateConsole(String console) {
        return StringUtils.hasText(console) &&
               Set.of("ps5", "ps4", "switch", "xbox").contains(console);
    }

}
```

Re-run the service with the above changes and then the client (same as before), and wait a minute for metrics to be published. Then open up the Wavefront console again; use that handy link printed in the console output!

You can now see different metrics broken down by the console. Click on `Dashboards` > `Spring Boot Dashboard`, and you'll note that the `Top Requests` and `Top Failed Requests` have more entries. This time, you may break down the results in terms of each console. Hover over them, and you'll see the details. 

Here are successful requests.

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/15.png" width = "500" /> 

Here are failed requests.

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/16.png" width = "500" /> 

It seems to us that the `ps5` console has a high coincidence with the failed requests. Let's look at the trace information. Click on `Applications` > `Traces`, to see the updated data. 

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/11.png" width = "500" /> 

Click on the critical path breakdown and expand the panel. Click on a particular segment, as shown here, and extend the `Tags` branch. You'll see all the tags associated with a particular request, including the `console` tag. The failing request depicted was after someone requested the availability of the `ps5` console. It'd sure be nice to filter based on the console, wouldn't it? Click the `+` icon next to the tag `console`, and Wavefront will add it to the search criteria. Click `Search` to see all the errant traces and find the culprits.

<img src = "https://github.com/shakuzen/console-availability/raw/master/images/12.png" width = "500" />

Our data breaks down both traces and metrics in terms of the console, our domain-specific concept.


## Metrics and Tracing Go Together like Chicken and Pickle Brine

What? You've _never_ tried chicken and pickle brine? It's good. It's [really good](https://www.eatthis.com/weird-food-combinations/). Can you imagine how much more free time once you've standardized on Spring and Wavefront and don't have to maintain as much undifferentiated infrastructure yourself? It's gonna be sweet. You'll have so much time. You'll have time enough to try chicken and pickle brine. 

You have seen a concrete example of using metrics and tracing together. Let's review some uses and anti-patterns for metrics and tracing. This hopefully makes clear why you want both metrics and tracing and how to use each for what. It may be useful to consider the framing set up in [Peter Bourgon's Metrics, tracing, and logging](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html) blog post.

Tracing and metrics overlap in providing insight on request-scoped interactions in our services. However, some information  provided by metrics and tracing is disjointed. Traces excel at showing the relationship between services as well as high-cardinality data about a specific request, such as a user ID associated with a request. Distributed tracing helps you pinpoint the source of an issue in your distributed system quickly. The tradeoff is that at high volumes and strict performance requirements, traces need to be sampled to control costs. This means that the specific request you are interested in may not be in the sampled tracing data. 

On the other side of the coin, metrics aggregate all measurements and export the aggregate at time intervals to define the time series data. All data is included in this aggregation and cost does not increase with traffic, so long as best practices on tag cardinality are followed. Therefore, a metric measuring the maximum latency of something will include the slowest request, and a calculation of error rate will be accurate, regardless of any sampling on tracing data.

Metrics can be used beyond the scope of requests for monitoring memory, CPU usage, garbage collection, and caches, to name a few. You will want to use metrics for your alerting, SLOs (service-level objectives), and dashboards. In the `console-availability` example, it would be an alert about an SLO violation that notifies us about a high error rate for our service. (You don't want to stare at dashboards 24/7 to detect issues, do you?)

Then, with both metrics and traces, we can jump from one to the other using common metadata available in each. Both metrics and trace information support capturing arbitrary key-value pairs with the data called tags. For example, given an alert notification about high latency on an HTTP-based service (based on metrics), you could link to a search of spans (trace data) that match the alert. You would search for spans with the same service, HTTP method, HTTP URI, and that are above a duration threshold to quickly get a sampling of traces matching the alert.

In conclusion, data is better than no data, and integrated data is better than non-integrated data. Micrometer and Spring Cloud Sleuth provides a solid observability posture, out of the box, but can be configured and adapted to your business/domain's context. And, finally, while you _could_ use Micrometer or Spring Cloud Sleuth with any number of other backends, we find Wavefront a convenient and powerful option.
