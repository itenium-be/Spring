# Docker Compose

Typically, we’d run docker-compose up to start and docker-compose down to stop our containers based on a docker-compose.yml. We can now delegate those Docker Compose commands to Spring Boot 3. While the Spring Boot application starts or stops, it’ll also manage our containers.

Furthermore, it has in-built management for multiple services, such as SQL databases, MongoDB, Cassandra, etc. Therefore, we might not need configuration classes or properties to duplicate in our application resource file.

Finally, we’ll see that we can use this support with custom Docker images and Docker Compose profiles.

## Create a new application

Open a new project and create a spring boot 3.2 application in java 21
group id: be.itenium
artifact: docker-compose

add spring-boot-starter-web,spring-boot-starter-data-mongodb,spring-boot-starter-actuator and spring-boot-docker-compose as dependency and create the application

## Create a docker file

Adding a docker file to resources

```
version: '3.8'
services:
  db:
    image: mongo:latest
    ports:
      - '27017:27017'
    volumes:
      - db:/data/db
volumes:
  db:
    driver:
      local
```
## adding spring properties for docker compose

```
spring.docker.compose.enabled=true
spring.docker.compose.file=docker-compose.yml
```

## Database configuration

We don’t need a database configuration. The Docker Compose Support will create a default one. However, we can still add our MongoDB configuration using a profile.

Here is an example of that config class with a profile:

```
/**
 * This profile is active for non docker-compose profile and will set up a MongoClient.
 * When docker-compose profile is active, the application will boot with application-docker-compose.yml and the Docker Compose support will start a default configuration
 */
@Configuration
@Profile("!docker-compose")
public class MongoConfig extends AbstractMongoClientConfiguration {

    @Override
    protected String getDatabaseName() {
        return "test";
    }

    @Override
    public MongoClient mongoClient() {
        ConnectionString connectionString = new ConnectionString("mongodb://localhost:27017/test");
        MongoClientSettings mongoClientSettings = MongoClientSettings.builder()
                .applyConnectionString(connectionString)
                .build();

        return MongoClients.create(mongoClientSettings);
    }

    @Override
    public Collection<String> getMappingBasePackages() {
        return Collections.singleton("com.baeldung.dockercompose");
    }
}
```

This way, we can choose whether to use the Docker Compose support. If we don’t go with a profile, the application will expect a database already running.

## create the model

```
@Document("item")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Item {

    @Id
    private String id;
    private String name;
    private int quantity;
    private String category;
}
```

## create the REST Controller

Lets just add CRUD operations

```
@RestController
@RequestMapping("/item")
@RequiredArgsConstructor
public class ItemController {

    private final ItemRepository itemRepository;

    @PostMapping(consumes = APPLICATION_JSON_VALUE)
    public ResponseEntity<Item> save(final @RequestBody Item item) {
        return ResponseEntity.ok(itemRepository.save(item));
    }

    @GetMapping(produces = APPLICATION_JSON_VALUE)
    public Item findByName(@RequestParam final String name) {
        return itemRepository.findItemByName(name);
    }

    @GetMapping(value = "category", produces = APPLICATION_JSON_VALUE)
    public List<Item> findByCategory(@RequestParam final String category) {
        return itemRepository.findAllByCategory(category);
    }

    @GetMapping(value = "count", produces = APPLICATION_JSON_VALUE)
    public long count() {
        return itemRepository.count();
    }
}
```

## Create the repository

```
import java.util.List;

import be.itenium.docker.model.Item;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;


public interface ItemRepository extends MongoRepository<Item, String> {

    @Query("{name:'?0'}")
    Item findItemByName(String name);

    @Query(value = "{category:'?0'}")
    List<Item> findAllByCategory(String category);

    long count();
}
```

## Application test

Make sure you have docker installed and open up docker desktop. Start the application with the profile docker-compose. Now we can see that Spring boot start the container before starting the application and that it is running in docker desktop. 

Stop your application and you will see the container also being stopped by Spring boot.

Interestingly, Spring Boot 3 will automatically check for container readiness. Containers can take some time to become fully ready. Therefore, this feature frees us to use the healthcheck command to see whether a container is ready.

## final thaughts

In a production environment, our docker services can spread across different instances. Therefore, in that case, we might not need this support. However, we can activate a Spring profile that loads from a docker-compose.yml definition for local development only.

This support integrates nicely with our IDE, and we won’t jump back and forth on a command line to start and stop the Docker services.