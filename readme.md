# Spring Boot 3.X 

Welcome to the Spring boot 3.X training

We have a few exercises in this repo.

1. Virtual threads with Spring Boot 3.2 and Java 21 and using the new RestClient
	See readme in the virtual threads folder
2. Docker compose with Spring Boot 3.1 or higher
	See readme in the docker compose folder
3. Docker compose and test containers: https://github.com/SimonVerhoeven/sbldi?tab=readme-ov-file#testcontainers-at-development-time
4. Observability
	See below
5. Native compilation: https://github.com/SimonVerhoeven/ssss
6. Restclient demo: https://github.com/SimonVerhoeven/restclient-demo

## Observability

Checkout this project to start

### Create a greeting record

```java
public record Greeting(String name) {}
```

### Create a controller

```java
import be.itenium.observability.model.Greeting;
import be.itenium.observability.service.GreetingService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

@RestController
public class GreetingController {
    private final GreetingService service;

    GreetingController(GreetingService service) {
        this.service = service;
    }

    @GetMapping("/hello/{name}")
    public Mono<Greeting> greeting(@PathVariable("name") String name) {
        return service
                .greeting(name);
    }
}
```

### Create a greeting service

here is where the magic happens with micrometer where tyou observe your methode greeting with Mono of reactor and you can tag the data it will produce

```java
import be.itenium.observability.model.Greeting;
import io.micrometer.observation.ObservationRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import reactor.core.observability.micrometer.Micrometer;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Random;
import java.util.function.Supplier;

@Service
@Slf4j
public class GreetingService {

    private final Supplier<Long> latency = () -> new Random().nextLong(500);

    private final ObservationRegistry registry;

    GreetingService(ObservationRegistry registry) {
        this.registry = registry;
    }

    public Mono<Greeting> greeting(String name) {
        Long lat = latency.get();
        return Mono
                .just(new Greeting(name))
                .delayElement(Duration.ofMillis(lat))
                .name("greeting.call")
                .tag("latency", lat > 250 ? "high" : "low")
                .tap(Micrometer.observation(registry))
                ;
    }
}
```

### Docker containers

1. grafana
2. prometheus
3. loki -> logging
4. tempo -> tracing

you can run the appplication and see if it works on http://localhost:8787/hello/spring-boot-3

You can open Grafana and open the dashboard that is included in this project on http://localhost:3000/ 

If you refresh the URL multiple times you will start seeing data in the dashboard.

