### `api` library documentation
> Sep 23, 2024

Project Structure:
```sh
cd /home/dave/Documents/Workspace-api-Microservices/alpha
```

#### Build Settings `build.gradle`
```sh
plugins {
	id 'io.spring.dependency-management' version '1.1.5'
	id 'java'
}

group = 'com.david.microservices.alpha.api'
version = '1.0.0-SNAPSHOT'

java {
	toolchain {
		languageVersion = JavaLanguageVersion.of(17)
	}
}

repositories {
	mavenCentral()
}

ext {
    springBootVersion = '3.3.0'
}

dependencies {
	
	implementation platform("org.springframework.boot:spring-boot-dependencies:${springBootVersion}")
	implementation 'org.springframework.boot:spring-boot-starter-webflux'
	
	implementation 'org.springdoc:springdoc-openapi-starter-webflux-ui:2.0.2'
	
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
	useJUnitPlatform()
}

```

See entire structure of `api` [here](./project_structure_lib_api.txt)

What's notable from the api lib is the definition of the POJOs for the core entities (product, recommendation, review), composite objects and exceptions:
```sh
com.david.microservices.alpha.api.core.product
com.david.microservices.alpha.api.core.recommendation
com.david.microservices.alpha.api.core.review
com.david.microservices.alpha.api.exceptions
com.david.microservices.alpha.api.composite.product
```

For each:
##### `com.david.microservices.alpha.api.core.product`
1. [Product.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/product/Product.java)
2. [ProductService.java interface](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/product/ProductService.java)


##### `com.david.microservices.alpha.api.core.recommendation`
1. [Recommendation.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/recommendation/Recommendation.java)
2. [RecommendationService.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/recommendation/RecommendationService.java)


##### `com.david.microservices.alpha.api.core.review`
1. [Review.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/review/Review.java)
2. [ReviewService.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/review/ReviewService.java)


##### `com.david.microservices.alpha.api.exceptions`
1. BadRequestException.java
2. [InvalidInputException.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/exceptions/InvalidInputException.java)
3. [NotFoundException.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/exceptions/NotFoundException.java)


##### `com.david.microservices.alpha.api.composite.ProductService`
1. [ProductAggregate.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/composite/product/ProductAggregate.java)
2. [ProductCompositeService.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/composite/product/ProductCompositeService.java)
3. [RecommendationSummary.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/composite/product/RecommendationSummary.java)
4. [ReviewSummary.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/composite/product/ReviewSummary.java)
5. [ServiceAddresses.java](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/composite/product/ServiceAddresses.java)


To enable this library resolvable for import by other projects, the `settings.gradle` file looks like this:
```sh
rootProject.name = 'api'

```


