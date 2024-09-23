# Microservices with Spring Boot 3.3 / Gradle 8.8 / Docker / Kubernetes

[GitHub: https://github.com/david-matu/product-microservices](https://github.com/david-matu/product-microservices)

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

`api` lib [here](./project_structure_lib_api.md)


## Adding Persistence to the microservices
> Aug_27_2024  
Featuring **MongoDB** and **MySQL** databases

:sunny:

##### Installing MongoDB
- [x] Setup Mongo DB image
- [ ] Setup MySQL image

`cd /home/dave/Documents/Workspace-Microservices-alpha`

Open the ``docker-compose.yml`` file and add the following entries under ``services``:  

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
##### product microservice  
Create `product entity`
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

// SEP_23_2024


Create `product repository`

```java
package com.david.microservices.alpha.product.persistence;

import java.util.Optional;

import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.PagingAndSortingRepository;

public interface ProductRepository extends PagingAndSortingRepository<ProductEntity, String>, CrudRepository<ProductEntity, String>{
	
	Optional<ProductEntity> findByProductId(int productId);
}
```

Create `product mapper`
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

##### recommendation microservice

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

##### review microservice
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


Create `ReviewMapper`:
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
