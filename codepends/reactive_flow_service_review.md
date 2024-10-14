#### Reactive Review Service
> Oct 14, 2024

##### The `ReviewRepository`
There is no change in base class and types. That's because currently there is no reactive support for JPA-based repositories

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

##### The `ReviewService` 
```java
package com.david.microservices.alpha.api.core.review;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface ReviewService {
	
	/**
	 * Example usage: "curl $HOST:$PORT/review?productId=1"
	 * 
	 * @param productId Id of the product
	 * @return the reviews of the product
	 */
	@GetMapping(value= "/reviews", produces = "application/json")
	Flux<Review> getReviews(@RequestParam(value = "productId", required = true) int productId);
	
	@PostMapping(value = "/reviews", consumes = "application/json", produces = "application/json")
	Mono<Review> createReview(@RequestBody Review body);
	
	@DeleteMapping(value = "/reviews")
	Mono<Void> deleteReviews(@RequestParam(value = "productId", required = true) int productId);
}
```

##### Update the `ReviewServiceImpl`
We'll first configure thread pools:

__`ReviewServiceApplication`__
```java
package com.david.microservices.alpha.review;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;

import reactor.core.scheduler.Scheduler;
import reactor.core.scheduler.Schedulers;

@SpringBootApplication
@ComponentScan("com.david.microservices.alpha")
public class ReviewServiceApplication {
	
	private static final Logger LOG = LoggerFactory.getLogger(ReviewServiceApplication.class);
	
	private final Integer threadPoolSize;
	private final Integer taskQueueSize;
	
	public ReviewServiceApplication(@Value("${app.threadPoolSize:10}") Integer threadPoolSize, @Value("${app.taskQueueSize:100}") Integer taskQueueSize) {
		this.threadPoolSize = threadPoolSize;
		this.taskQueueSize = taskQueueSize;
	}
	
	@Bean
	public Scheduler jdbcScheduler() {
		LOG.info("Creates a jdbcScheduler with thread pool size = {}", threadPoolSize);
		
		return Schedulers.newBoundedElastic(threadPoolSize, taskQueueSize, "jdbc-pool");
	}
	
	public static void main(String[] args) {
		//SpringApplication.run(ReviewServiceApplication.class, args);
		ConfigurableApplicationContext ctx = SpringApplication.run(ReviewServiceApplication.class, args);
		
		String mysqlUri = ctx.getEnvironment().getProperty("spring.datasource.url");
		LOG.info("Connected to MySQL: " + mysqlUri);
	}
}
```

In the REST controller: `ReviewServiceImpl`

```java
package com.david.microservices.alpha.review.services;

import java.util.List;

import static java.util.logging.Level.FINE;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.web.bind.annotation.RestController;

import com.david.microservices.alpha.api.core.review.Review;
import com.david.microservices.alpha.api.core.review.ReviewService;
import com.david.microservices.alpha.api.exceptions.InvalidInputException;
import com.david.microservices.alpha.review.persistence.ReviewEntity;
import com.david.microservices.alpha.review.persistence.ReviewRepository;
import com.david.microservices.alpha.util.http.ServiceUtil;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Scheduler;

@RestController
public class ReviewServiceImpl implements ReviewService {

	private static final Logger LOG = LoggerFactory.getLogger(ReviewServiceImpl.class);
	
	private final ServiceUtil serviceUtil;
	
	private final ReviewMapper mapper;
	private final ReviewRepository repo;
	
	private final Scheduler jdbcScheduler;
	
	/**
	 * 
	 * @param repo
	 * @param mapper
	 * @param serviceUtil
	 */
	@Autowired
	public ReviewServiceImpl(@Qualifier("jdbcScheduler") Scheduler jdbcScheduler, ReviewRepository repo, ReviewMapper mapper, ServiceUtil serviceUtil) {
		this.jdbcScheduler = jdbcScheduler;
		this.serviceUtil = serviceUtil;
		this.mapper = mapper;
		this.repo = repo;
	}

	@Override
	public Flux<Review> getReviews(int productId) {
		
		if(productId < 1) {
			throw new InvalidInputException("Invalid productId: " + productId);
		}
		
		LOG.info("Will get revies for product with id={}", productId);
		
		return Mono.fromCallable(() -> internalGetReviews(productId))
				.flatMapMany(Flux::fromIterable)
				.log(LOG.getName(), FINE)
				.subscribeOn(jdbcScheduler);
	}

	private List<Review> internalGetReviews(int productId) {
		List<ReviewEntity> entityList = repo.findByProductId(productId);
		List<Review> list = mapper.entityListToApiList(entityList);
		
		list.forEach(e -> e.setServiceAddress(serviceUtil.getServiceAddress()));
		
		LOG.debug("Response size: {}", list.size());
		
		return list;
	}

	@Override
	public Mono<Review> createReview(Review body) {
		/*
		
		*/
		
		if(body.getProductId() < 1) {
			throw new InvalidInputException("Invalid productId: " + body.getProductId());
		}
		
		return Mono.fromCallable(() -> internalCreateReview(body))
				.subscribeOn(jdbcScheduler);
	}
	
	private Review internalCreateReview(Review body) {
		try {
			ReviewEntity entity = mapper.apiToEntity(body);
			ReviewEntity newEntity = repo.save(entity);
			
			LOG.debug("createReview: created a review entity: {}/{}", body.getProductId(), body.getReviewId());
			return mapper.entityToApi(newEntity);
		} catch (DataIntegrityViolationException dive) {
			throw new InvalidInputException("Duplicate key, Product Id: " + body.getProductId() + ", Review Id: " + body.getReviewId());
		}		
	}

	@Override
	public Mono<Void> deleteReviews(int productId) {
		if(productId < 1) {
			throw new InvalidInputException("Invalid productId: " + productId);
		}
		
		return Mono.fromRunnable(() -> internalDeleteReviews(productId)).subscribeOn(jdbcScheduler).then();
	}

	private void internalDeleteReviews(int productId) {
		LOG.debug("deleteReviews: tries to delete reviews for the product with productId: {}", productId);
		
		repo.deleteAll(repo.findByProductId(productId));
	}
}

```

In the `ProductCompositeIntegration`
```java
	@Override
	public Flux<Review> getReviews(int productId) {
		
		String url = reviewServiceUrl + productId;
		
		return webClient.get().uri(url).retrieve().bodyToFlux(Review.class)
				.log(LOG.getName(), FINE)
				.onErrorResume(error -> emptyReviews());
	}
	
	private Flux<Review> emptyReviews() {
		return  Flux.empty();
	}


```

For the __`createReview()`__ and __`deleteReview()`__ which are supposed to be event-driven, we'll explore how to enable that here:


##### Asynchronous Messaging

In the __`build.gradle`__ file:

```sh
ext {
	springCloudVersion = "2022.0.1"
}

dependencies {
	implementation project(':api')
    implementation project(':util')
   
    // other ommitted

	implementation 'org.springframework.cloud:spring-cloud-starter-stream-rabbit'
	implementation 'org.springframework.cloud:spring-cloud-starter-stream-kafka'
	
	testImplementation 'org.springframework.cloud:spring-cloud-stream::test-binder'
}

dependencyManagement {
	imports {
		mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
	}
}
```
