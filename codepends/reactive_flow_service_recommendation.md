#### Reactive Recommendation Service
> Oct 1, 2024

##### The Repo
```java
package com.david.microservices.alpha.recommendation.persistence;

import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import reactor.core.publisher.Flux;

public interface RecommendationRepository extends ReactiveCrudRepository<RecommendationEntity, String>{
	Flux<RecommendationEntity> findByProductId(int productId);
}
```

##### The `RecommendationServiceImpl`
```java
package com.david.microservices.alpha.recommendation.services;


import static java.util.logging.Level.FINE;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.web.bind.annotation.RestController;

import com.david.microservices.alpha.api.core.recommendation.Recommendation;
import com.david.microservices.alpha.api.core.recommendation.RecommendationService;
import com.david.microservices.alpha.api.exceptions.InvalidInputException;
import com.david.microservices.alpha.recommendation.persistence.RecommendationEntity;
import com.david.microservices.alpha.recommendation.persistence.RecommendationRepository;
import com.david.microservices.alpha.util.http.ServiceUtil;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
public class RecommendationServiceImpl implements RecommendationService {
	
	private static final Logger LOG = LoggerFactory.getLogger(RecommendationServiceImpl.class);
	
	private final ServiceUtil serviceUtil;
	
	private final RecommendationRepository repo;
	private final RecommendationMapper mapper;
	
	@Autowired
	public RecommendationServiceImpl(RecommendationRepository repository, RecommendationMapper mapper, ServiceUtil serviceUtil) {
		this.serviceUtil = serviceUtil;
		this.repo = repository;
		this.mapper = mapper;
	}

	@Override
	public Flux<Recommendation> getRecommendations(int productId) {
		
		if(productId < 1) {
			throw new InvalidInputException("Invalid productId: " + productId);
		}
		
		LOG.debug("Will get recommendations for product with id={}", productId);
		
		return repo.findByProductId(productId)
				.log(LOG.getName(), FINE)
				.map(e -> mapper.entityToApi(e))
				.map(e -> setServiceAddress(e));
	}
	
	@Override
	public Mono<Recommendation> createRecommendation(Recommendation body) {
		
		if(body.getProductId() < 1) {
			throw new InvalidInputException("Invalid productId: " + body.getProductId());
		}
		
		RecommendationEntity entity = mapper.apiToEntity(body);
		Mono<Recommendation> newEntity = repo.save(entity)
				.log(LOG.getName(), FINE)
				.onErrorMap(
						DuplicateKeyException.class, 
						ex -> new InvalidInputException("Duplicate key, Product Id: " + body.getProductId() + ", Recommendation Id: " + body.getRecommendationId()))
				.map(e -> mapper.entityToApi(e));
		
		return newEntity;
	}

	@Override
	public Mono<Void> deleteRecommendations(int productId) {
		if(productId < 1) {
			throw new InvalidInputException("Invalid productId: " + productId);
		}
		
		LOG.debug("deleteRecommendations: tries to delete recommendations for the product with productId: {}", productId);
		return repo.deleteAll(repo.findByProductId(productId));
	}

	private Recommendation setServiceAddress(Recommendation e) {
		e.setServiceAddress(serviceUtil.getServiceAddress());
		return e;
	}
	
}
```

Upto this juncture, the persistence layer of the Recommendation service is non-blocking


##### Update `ProductCompositeIntegration`

```java
	@Override
	public Flux<Recommendation> getRecommendations(int productId) {
		String url = recommendationServiceUrl + productId;
		
		LOG.debug("Will call getRecommendations API on URL: {}", url);
		
		// If something goes wrong when fetching recommendations, return a partial response
		return webClient.get().uri(url)
				.retrieve().bodyToFlux(Recommendation.class)
				.log(LOG.getName(), FINE)
				.onErrorResume(error -> empty());
	}
	
	private Mono<Recommendation> empty() {
		return Mono.empty();
	}
```


