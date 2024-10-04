## Microservices with Spring Boot 3.3 / Gradle 8.8 / Docker / Kubernetes

GitHub: [https://github.com/david-matu/product-microservices](https://github.com/david-matu/product-microservices)

Featuring microservices architecture, the walk aims to build an end-to-end Product Service supported by modern stacks: Spring Boot, Gradle, Docker, Kubernetes, monitoring and logging tools which I'll list below.  
This repo contains codebase for the Product Service comprising:

* `api`  the library that defines interfaces and plain old Java objects that the Product Service implements in the concrete services
* `util` this library defines protocol information such as Services Addresses and error handling classes
* `product-service` this is the core microservice features the Product itself
* `recommendation-service` entails recommendations related to the Product
* `review-service` serves reviews given by authors related to the Product
* `product-composite-service` aggregates the three microservices above such that they sit behind this service.  
The composite makes the actual communication to the microservices and delivers an aggregate response to the requesting client  

### Project structure
Current directory structure:
```sh
[dave@localhost Workspace-Microservices-alpha]$ ll
total 36
drwxr-xr-x. 7 dave dave  165 Jul 28 14:19 api
-rw-r--r--. 1 dave dave 1164 Aug 25 08:46 docker-compose.yml
drwxr-xr-x. 3 dave dave   21 Jul 16 00:44 gradle
-rwxr-xr-x. 1 dave dave 8706 Jul 16 00:45 gradlew
-rw-r--r--. 1 dave dave 2918 Jul 16 00:45 gradlew.bat
drwxr-xr-x. 7 dave dave  147 Jul 25 07:26 microservices
-rw-r--r--. 1 dave dave 1885 Aug 19 08:18 README.md
-rw-r--r--. 1 dave dave  211 Jul 26 06:18 settings.gradle
-rwxrwxrwx. 1 dave dave 3804 Aug 14 07:59 test-em-all.bash
drwxr-xr-x. 8 dave dave 4096 Jul 27 08:57 util

```

### Including libraries in the project (featuring the `product microservice`)
Ensure the following is in the `settings.gradle` file of the project:
```sh
rootProject.name = 'product-service'
include('api')
include('util')
include project(':api').projectDir = new File(settingsDir, '../../api')
include project(':util').projectDir = new File(settingsDir, '../../util')
```

The complete ``build.gradle`` content:
```sh
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.3.0'
	id 'io.spring.dependency-management' version '1.1.5'
}

group = 'com.david.microservices.alpha.product'
version = '1.0.0-SNAPSHOT'

java {
	toolchain {
		languageVersion = JavaLanguageVersion.of(17)
	}
}

repositories {
	mavenCentral()
}

jar {
	enabled = false
}

dependencies {
	implementation(project(':api'))
	implementation(project(':util'))
	
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
	implementation 'org.springframework.boot:spring-boot-starter-webflux'
	// Skipping explicit specification of Netty, thinks loaded by default
	
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'io.projectreactor:reactor-test'
	// testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
	useJUnitPlatform()
}

```

### Add Dockerfiles
Use the following ``Dockerfile`` content:
```sh
FROM eclipse-temurin:17.0.5_8-jre-focal as builder
WORKDIR /extracted
ADD ./build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17.0.5_8-jre-focal
WORKDIR application
COPY --from=builder /extracted/dependencies/ ./
COPY --from=builder /extracted/spring-boot-loader/ ./
COPY --from=builder /extracted/snapshot-dependencies/ ./
COPY --from=builder /extracted/application/ ./
EXPOSE 8080
ENTRYPOINT [ "java", "org.springframework.boot.loader.launch.JarLauncher" ]
```

## The Orchestration

[`api`](./lib_api.md) library project has definitions for core classes: `Product`, `Recommendation`, `Review`, `ProductAggregate` and as well, exceptions classes `InvalidInputException`, `NotFoundException`


[`util`](./lib_util.md) library project, purposed to handle protocol level information, defines utility classes namely `GlobalControllerExceptionHandler`, `HttpErrorInfo`, `ServiceUtil`


[`microservices`](./microservices.md) subprojects have concrete implementation to deliver info regarding a product

![View UML](./architecture/Microservices-Dave-Alpha.drawio.svg)

## Adding Persistence to the microservices
> Aug_27_2024  
Featuring **MongoDB** and **MySQL** databases


##### Installing MongoDB
- [x] Setup Mongo DB image
- [ ] Setup MySQL image

`cd /home/dave/Documents/Workspace-Microservices-alpha`

Open the ``docker-compose.yml`` file and add the following entries for ``services``:  

```yaml
services:
   mongodb:
   image: mongo:6.0.4
   mem_limit: 512m
   ports:
    - "27017:27017"
   command: mongod
   healthcheck:
    test: "mongostat -n 1"
    interval: 5s
    timeout: 2s
    retries: 60

  mysql:
   image: mysql:8.0.32
   mem_limit: 512m
   ports:
    - "3306-3306"
   environment:
    - MYSQL_ROOT_PASSWORD=rootpwd
    - MYSQL_DATABASE=review_db
    - MYSQL_USER=user
    - MYSQL_PASSWORD=mypwd
   healthcheck:
    test: "/usr/bin/mysql --user=user --password=mypwd --execute \"SHOW DATABASES;\""
    interval: 5s
    timeout: 2s
    retries: 60
```

The `product` and `recommendation` microservices, will use **MongoDB** while the `review` microservice will use **MySQL**

### 1. Add dependencies
##### Product and Review microservices
In the `build.gradle` file of both projects, add the following:
```sh
//Mapstruct generates implementation of the bean mappings at compile time by processing MapStruct annotations
ext {
	mapstructVersion = "1.5.3.Final"
}

dependencies {
	
	// ... omitted other dependencies previously shown
	
	// Add Persistence capabilities for MongoDB
	implementation "org.mapstruct:mapstruct:${mapstructVersion}"
	
	annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
	testAnnotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
	compileOnly "org.mapstruct:mapstruct-processor:${mapstructVersion}"
	
	implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
	
	
	// Use Testcontainers and its support for JUnit5 to enable use of MongoDB when running automated integration tests 
	implementation platform('org.testcontainers:testcontainers-bom:1.15.2')
	testImplementation 'org.testcontainers:testcontainers'
	testImplementation 'org.testcontainers:junit-jupiter'
	testImplementation 'org.testcontainers:mongodb'
}
```

### 2. Add the codes for Persistence and Mapping 
##### Product microservice  
Create `ProductEntity`
```java
@Document(collection = "products")
public class ProductEntity {
	@Id private String id;
	
	@Version private Integer version;
	
	@Indexed(unique = true)
	private int productId;
	
	private String name;
	private int weight;
	
	public ProductEntity() {}
	
	public ProductEntity(int productId, String name, int weight) {
		this.productId = productId;
		this.name = name;
		this.weight = weight;
	}
	...	
}
```

> // SEP_23_2024


Create `ProductRepository`

```java
package com.david.microservices.alpha.product.persistence;

import java.util.Optional;

import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.PagingAndSortingRepository;

public interface ProductRepository extends PagingAndSortingRepository<ProductEntity, String>, CrudRepository<ProductEntity, String>{
	
	Optional<ProductEntity> findByProductId(int productId);
}
```

Create `ProductMapper`

This interface provides conversion of an api(core) object into entity object that is understandable by the persistence spectrum, and vise-versa. 


You can see that the `@Mappings` annotations specifying fields to ignore during conversion. For example, the `id` and `version` attributes of 
`ProductEntity` will be ignored 


```java
package com.david.microservices.alpha.product.services;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;

import com.david.microservices.alpha.api.core.product.Product;
import com.david.microservices.alpha.product.persistence.ProductEntity;

@Mapper(componentModel = "spring")
public interface ProductMapper {
	
	@Mappings({
		@Mapping(target = "serviceAddress", ignore = true)
	})
	Product entityToApi(ProductEntity entity);
	
	@Mappings({
		@Mapping(target = "id", ignore = true), 
		@Mapping(target = "version", ignore = true)
	})
	ProductEntity apiToEntity(Product api);
}
```

##### Recommendation microservice

Create `RecommendationEntity`
```java
package com.david.microservices.alpha.recommendation.persistence;

import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.Version;
import org.springframework.data.mongodb.core.index.CompoundIndex;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection="recommendation")
@CompoundIndex(name = "prod-rec-id", unique = true, def = "{'prodId': 1, 'recommendationId': 1}")
public class RecommendationEntity {
	@Id
	private String id;
	
	@Version
	private Integer version;
	
	private int productId;
	private int recommendationId;
	private String author;
	private int rating;
	private String content;
	
	public RecommendationEntity() {
		super();
	}
	
	public RecommendationEntity(int productId, int recommendationId, String author, int rating, String content) {
		this.productId = productId;
		this.recommendationId = recommendationId;
		this.author = author;
		this.rating = rating;
		this.content = content;
	}

	// ...
}

```


Create the `RecommendationRepository`:
```java
package com.david.microservices.alpha.recommendation.persistence;

import java.util.List;

import org.springframework.data.repository.CrudRepository;

public interface RecommendationRepository extends CrudRepository<RecommendationEntity, String>{
	
	List<RecommendationEntity> findByProductId(int productId);
}
```


Create `RecommendationMapper`:
```java
package com.david.microservices.alpha.recommendation.services;

import java.util.List;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;

import com.david.microservices.alpha.api.core.recommendation.Recommendation;
import com.david.microservices.alpha.recommendation.persistence.RecommendationEntity;

@Mapper(componentModel = "spring")
public interface RecommendationMapper {
	
	@Mappings({
		@Mapping(target = "rate", source = "entity.rating"),
		@Mapping(target = "serviceAddress", ignore = true)
	})
	Recommendation entityToApi(RecommendationEntity entity);
	
	@Mappings({
		@Mapping(target = "rating", source = "api.rate"),
		@Mapping(target = "id", ignore = true),
		@Mapping(target = "version", ignore = true)
	})
	RecommendationEntity apiToEntity(Recommendation api);
	
	List<Recommendation> entityListToApiList(List<RecommendationEntity> entity);
	
	List<RecommendationEntity> apiListToEntityList(List<Recommendation> api);
}
```

##### Review microservice
Create `ReviewEntity`
```java
package com.david.microservices.alpha.review.persistence;

import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.Version;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Index;
import jakarta.persistence.Table;

@Entity
@Table(name = "reviews", indexes = { @Index(name = "reviews_unique_idx", unique = true, columnList = "productId,reviewId") })
public class ReviewEntity {
	
	@Id
	@GeneratedValue
	private int id;
	
	@Version
	private int version;
	
	private int productId;
	private int reviewId;
	private String author;
	private String subject;
	private String content;
	
	public ReviewEntity() {}
	
	public ReviewEntity(int productId, int reviewId, String author, String subject, String content) {
		this.productId = productId;
		this.reviewId = reviewId;
		this.author = author;
		this.subject = subject;
		this.content = content;
	}

	// ...
	
}

```

Create `ReviewRepository`:
```java
package com.david.microservices.alpha.review.persistence;

import java.util.List;

import org.springframework.data.repository.CrudRepository;
import org.springframework.transaction.annotation.Transactional;

public interface ReviewRepository extends CrudRepository<ReviewEntity, Integer>{
	
	@Transactional(readOnly = true)
	List<ReviewEntity> findByProductId(int productId);
}

```


Create `ReviewMapper` interface:

```java
package com.david.microservices.alpha.review.services;

import java.util.List;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;

import com.david.microservices.alpha.api.core.review.Review;
import com.david.microservices.alpha.review.persistence.ReviewEntity;

@Mapper(componentModel = "spring")
public interface ReviewMapper {
	
	@Mappings({
		@Mapping(target = "serviceAddress", ignore = true)
	})
	Review entityToApi(ReviewEntity entity);
	
	@Mappings({
		@Mapping(target = "id", ignore = true),
		@Mapping(target = "version", ignore = true)
	})
	ReviewEntity apiToEntity(Review api);
	
	List<Review> entityListToApiList(List<ReviewEntity> entity);
	
	List<ReviewEntity> apiListToEntityList(List<Review> api);
}
```


___


##### Writing Tests for Persistence
> Oct 3, 2024

# Writing Automated Tests that focus on persistence:

@DataMongoTest

@DataJpaTest


#### Review Microservice Persistence

> Abstract class: ``MySqlTestBase``
> ``PersistenceTests`` extends ``MySqlTest``
> ``ReviewServiceApplicationTests`` extends ``MySqlTestBase``

```
class ReviewServiceApplicationTests extends MySqlTestBase {
	...
}
```


Provide a ``Logback`` configuration file under ```src/test/resources``` with the content:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<configuration>
	<include resource="org/springframework/boot/logging/logback/defaults.xml"/>
	<include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
	<root level="INFO">
		<appender-ref ref="CONSOLE" />
	</root>
</configuration>

```

The above config file defines default values and a log appender that can write log events to the console.

The log output is limited to the **INFO** log level. This will discard the ___DEBUG___ and ___TRACE___ log records that will be emitted by Testcontainers library 


**`ReviewServiceApplicationTests.java`**
```java


import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.test.web.reactive.server.WebTestClient;

import static org.springframework.http.HttpStatus.*;
import static reactor.core.publisher.Mono.just;
import static org.springframework.http.MediaType.APPLICATION_JSON;

import com.david.microservices.alpha.api.core.review.Review;
import com.david.microservices.alpha.review.persistence.ReviewRepository;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static  org.springframework.boot.test.context.SpringBootTest.WebEnvironment.RANDOM_PORT;
import static org.springframework.http.HttpStatus.BAD_REQUEST;
import static org.springframework.http.HttpStatus.UNPROCESSABLE_ENTITY;
import static org.springframework.http.MediaType.APPLICATION_JSON;

@SpringBootTest(webEnvironment = RANDOM_PORT)
class ReviewServiceApplicationTests extends MySqlTestBase {
	
	/* writing off these to add persistence tests
	private static int PRODUCT_ID_OK = 1;
	private static int PRODUCT_ID_NOT_FOUND = 213;
	private static int PRODUCT_ID_INVALID = -1;
	*/
	
	@Autowired
	private WebTestClient client;
	
	@Autowired
	private ReviewRepository repo;
	
	@BeforeEach
	void setupDb() {
		repo.deleteAll();
	}
	
	// ... upcoming tests discussion below
}
```

Talking of the ``postAndVerifyReview()`` method, we create a new ```Review``` object and direct the test client (``WebClient``) to post and assert the response gotten. The status returned should be ``OK`` and the content type as application/json.


> Oct 4, 2024

##### `getReviewsByProductId()` test

```java
@Test
	void getReviewsByProductId() {
		int productId = 1;
		
		// Acertain that there are zero reviews for any product now that the reviews db has been cleared before test
		assertEquals(0, repo.findByProductId(productId).size());
		
		// Post the reviews against product (1)
		postAndVerifyReview(productId, 1, OK);
		postAndVerifyReview(productId, 2, OK);
		postAndVerifyReview(productId, 3, OK);
		
		// Assert that 3 reviews have been inserted successfully against a productId
		assertEquals(3, repo.findByProductId(productId).size());
		
		getAndVerifyReviewsByProductId(productId, OK)
			.jsonPath("$.length()").isEqualTo(3)
			.jsonPath("$[2].productId").isEqualTo(3)
			.jsonPath("$[2].reviewId").isEqualTo(3);
	}

	private WebTestClient.BodyContentSpec getAndVerifyReviewsByProductId(int productId, HttpStatus expectedStatus) {
		return client.get()
				.uri("/review")
				.accept(APPLICATION_JSON)
				.exchange()
				.expectStatus().isEqualTo(expectedStatus)
				.expectHeader().contentType(APPLICATION_JSON)
				.expectBody();
	}

	private WebTestClient.BodyContentSpec postAndVerifyReview(int productId, int reviewId, HttpStatus expectedStatus) {
		Review review = new Review(productId, reviewId, "Author " + reviewId, "Subject " + reviewId, "Content" + reviewId, "SA");
		
		return client.post()
				.uri("/review")
				.body(just(review), Review.class)
				.accept(APPLICATION_JSON)
				.exchange()
				.expectStatus().isEqualTo(expectedStatus)
				.expectHeader().contentType(APPLICATION_JSON)
	}
```

##### `duplicateError` test
In this test, we assert that initially, the repository has zero reviews. Then we post a Review with ```productId=1``` and ```reviewId=1``` and confirm that the returned json object contains the respective productId and reviewId, and also the database contains exactly one review record.

We then post another Review with same product id and review id. This should return a message containing ```Duplicate key```

We ascertain that the database has maintained the earlier and only one record. There shouldn't be a duplicate record indeed!
```java
@Test
	void duplicateError() {
		int productId = 1;
		int reviewId = 1;
		
		assertEquals(0, repo.count());
		
		postAndVerifyReview(productId, reviewId, OK)
			.jsonPath("$.productId").isEqualTo(productId)
			.jsonPath("$.reviewId").isEqualTo(reviewId);
		
		assertEquals(1, repo.count());
		
		postAndVerifyReview(productId, reviewId, UNPROCESSABLE_ENTITY)
			.jsonPath("$.path").isEqualTo("/review")
			.jsonPath("$.message").isEqualTo("Duplicate key, Product Id: 1, Review Id:1");
		
		assertEquals(1, repo.count());
	}
```

##### `deleteReviews` test
