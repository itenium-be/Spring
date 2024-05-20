# Virtual threads

Why we use Virtual Threads in Spring?
Virtual threads offer a scalable solution without the need for additional hardware. By decoupling from platform threads, applications can spawn thousands or even millions of virtual threads to perform tasks concurrently, significantly improving scalability.

Let’s create two Spring Boot applications together to explore how virtual threads work in practice.

## The first application represents a blocking scenario, simulating HTTP communication with delays, highlighting the challenges of traditional threads in handling such operations.

### Spring Initializr

Open a new project and create a spring boot 3.2 application in java 21
group id: be.itenium
artifact: virtual-threads-blocking

add spring-boot-starter-web as dependency and create the application

### Configuring Application Properties

configure the server port as follows in “application.properties”

```
server.port=8081
```

### Creating a Controller

Now, let’s create a new Java class named “BlockController” to set up blocking endpoints and run the program.

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class BlockController {

    private static final Logger log = LoggerFactory.getLogger(BlockController.class);

    @GetMapping("/block/{seconds}")
    public void block(@PathVariable Integer seconds) throws InterruptedException{
        Thread.sleep(seconds*1000);
        log.info("Sleep for {} seconds ",seconds);
    }
}
```

Using Thread.sleep(seconds * 1000) adds a specified waiting time after calling the endpoint, where the desired duration is represented in seconds.
This application has just created an endpoint with a delay, simulating either network latency or processing delay. In real-world scenarios, when a client initiates an API request, the response is not instantaneous, and there is typically a delay before receiving the result.

## In the second application, we’ll experience the enhanced capabilities of virtual threads by enabling them in Spring Boot 3.2.

Generate and open the project using the same approach as in the first application.

### Configuring Application Properties

```
server.port=8082
```

### Creating a Controller

Create a Java class named “HomeController” to define endpoints. Subsequently, implement a GET request to invoke another HTTP request from a separate service. This setup enables us to call our first application using this GET method.

Within this method, we utilize the RestClient in Spring Boot to make an HTTP call. This RestClient functions as a synchronous HTTP client, making it suitable for a multi-threaded program.

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestClient;

@RestController
public class HomeController {

    private final static Logger logger = LoggerFactory.getLogger(HomeController.class);

    private final RestClient  restClient;
    public HomeController(RestClient.Builder builder){
        this.restClient=builder.baseUrl("http://localhost:8081").build();

    }
    @GetMapping("/block/{seconds}")
    public String home(@PathVariable Integer seconds){
        ResponseEntity<Void> result = restClient.get()
                .uri("/block/"+seconds)
                .retrieve()
                .toBodilessEntity();

        logger.info("{} on {}",result.getStatusCode(),Thread.currentThread());
        return Thread.currentThread().toString();

    }


}
```

In our machines, resources are limited, and we cannot allocate an unlimited amount to any application. Therefore, it is essential to impose restrictions on the resources accessible to the application. As a measure, we set a maximum limit of 10 threads for the server to utilize.

We are adding another configuration to the “application.properties”.

```
server.port=8082
server.tomcat.threads.max=10
```

I‘m using Apache Benchmark to concurrently send requests to the server and obtain a performance benchmark. This is how we send 60 requests, 20 concurrently, with a 2-second delay for each request. We expect the execution of all 60 requests to take 12 seconds. This is because if we send 20 requests concurrently, the server can’t process all 20 requests at once. This limitation is due to restricting it to 10 threads.

download [httpd-2.4.59-240404-win64-VS17.zip](https://www.apachelounge.com/download/#google_vignette) 
unzip the folder    
Move into the Apache24/bin directory and copy both the "ab" and "abs" executable files to your project directory

```
ab -n 60 -c 20 http://localhost:8082/block/2
```

You can see the following result in the IDE console for each reqeust.

```
2024-01-06T02:34:49.410+05:30  INFO 8888 --- [nio-8082-exec-2] be.itenium.thread.HomeController  : 200 OK on Thread[#38,http-nio-8082-exec-2,5,main]
```

Look at the Time taken for tests.

Let’s enable virtual threads and observe the performance gain from this existing feature in Spring Boot 3.2. To do that, go to the “application.properties ” file and make the following changes.
```
server.port=8082
spring.threads.virtual.enabled=true
```

Run the same benchmark test as earlier. If your virtual threads are enabled successfully, you can see the log for each request in the IDE console as follows.

```
2024-01-06T02:39:34.204+05:30  INFO 22348 --- [omcat-handler-0] be.itenium.thread.HomeController  : 200 OK on VirtualThread[#40,tomcat-handler-0]/runnable@ForkJoinPool-1-worker-1
```

You can clearly see the difference in how virtual threads enhance performance.

Before enable virual threads — 14.286 seconds
After enable virual threads — 8.340 seconds