### Code Flow: `recommendation microservice`
> Oct 9, 2024

#### The _POJO_: __Recommendation__

```java
package com.david.microservices.alpha.api.core.recommendation;

public class Recommendation {
	private int productId;
	private int recommendationId;
	private String author;
	private int rate;		// Rating -+- Stars
	private String content;
	private String serviceAddress;	//Determine which instance responded to a request
	
	public Recommendation(int productId, int recommendationId,  String author, int rate, String content, String serviceAddress) {
		this.productId = productId;
		this.recommendationId = recommendationId;
		this.author = author;
		this.rate = 0;
		this.content = content;
		this.serviceAddress = serviceAddress;
	}

	public Recommendation() {
		this.productId = 0;
		this.recommendationId = 0;
		this.author = null;
		this.rate = 0;
		this.content = null;
		this.serviceAddress = null;
	}
	
	// Getters and setters
}

```

#### The Service interface: `RecommendationService`
```java
package com.david.microservices.alpha.api.core.recommendation;

import java.util.List;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;

public interface RecommendationService {
	
	@GetMapping(value= "/recommendations", produces = "application/json")
	List<Recommendation> getRecommendations(@RequestParam(value = "productId", required = true) int productId);
	
	@PostMapping(value = "/recommendation", consumes = "application/json", produces = "application/json")
	Recommendation createRecommendation(@RequestBody Recommendation body);
	
	@DeleteMapping(value = "/recommendation")
	void deleteRecommendations(@RequestParam(value = "productId", required = true) int productId);
}

```

#### Update the `ProductCompositeIntegration` class to implement the above methods:
```java
package com.david.microservices.alpha.composite.product.services;

import static org.springframework.http.HttpMethod.GET;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.RestTemplate;

import com.david.microservices.alpha.api.core.product.Product;
import com.david.microservices.alpha.api.core.product.ProductService;
import com.david.microservices.alpha.api.core.recommendation.Recommendation;
import com.david.microservices.alpha.api.core.recommendation.RecommendationService;
import com.david.microservices.alpha.api.core.review.Review;
import com.david.microservices.alpha.api.core.review.ReviewService;
import com.david.microservices.alpha.api.exceptions.InvalidInputException;
import com.david.microservices.alpha.api.exceptions.NotFoundException;
import com.david.microservices.alpha.util.http.HttpErrorInfo;
import com.fasterxml.jackson.databind.ObjectMapper;

@Component
public class ProductCompositeIntegration implements ProductService, RecommendationService, ReviewService {
	
	private final Logger LOG = LoggerFactory.getLogger(ProductCompositeIntegration.class);
	
	private final RestTemplate restTemplate;
	private final ObjectMapper mapper;
	
	private final String productServiceUrl;
	private final String recommendationServiceUrl;
	private final String reviewServiceUrl;
	
	@Autowired
	public ProductCompositeIntegration(RestTemplate restTemplate, ObjectMapper mapper, 
			@Value("${app.product-service.host}") 
			String productServiceHost, 
			
			@Value("${app.product-service.port}") 
			String productServicePort,
			
			@Value("${app.recommendation-service.host}") 
			String recommendationServiceHost, 
			
			@Value("${app.recommendation-service.port}") 
			String recommendationServicePort,
			
			@Value("${app.review-service.host}") 
			String reviewServiceHost, 
			
			@Value("${app.review-service.port}") 
			String reviewServicePort) {
		
		this.restTemplate = restTemplate;
		this.mapper = mapper;
		
		this.productServiceUrl = "http://" + productServiceHost + ":" + productServicePort + "/product/";
		this.recommendationServiceUrl = "http://" + recommendationServiceHost + ":" + recommendationServicePort + "/recommendations?productId=";
		this.reviewServiceUrl = "http://" + reviewServiceHost + ":" + reviewServicePort + "/reviews?productId=";
	}

	// Omitted other override methods for brevity
	
	@Override
	public List<Recommendation> getRecommendations(int productId) {
		
		try {
			String url = recommendationServiceUrl + productId;
			
			LOG.debug("Will call getRecommendations API on URL: {}", url);
			
			List<Recommendation> recommendations = restTemplate.exchange(url, GET, null, new ParameterizedTypeReference<List<Recommendation>>() {}).getBody();
			
			LOG.debug("Found {} recommendations for a product with id: {}", recommendations.size(), productId);
			return recommendations;
		} catch (Exception ex) {
			LOG.warn("Got an exception while requesting recommendations, return zero recommendations: {}", ex.getMessage());
			return new ArrayList<>();
		}
	}

	@Override
	public Recommendation createRecommendation(Recommendation body) {
		
		try {
			String url = recommendationServiceUrl;
			LOG.debug("Will post a new recommendation to URL: {}", url);
			
			Recommendation rec = restTemplate.postForObject(url, body, Recommendation.class);
			LOG.debug("Created a recommendation with id: {}", rec.getProductId());
			
			return rec;
		} catch (HttpClientErrorException ex) {
			throw handleHttpClientException(ex);
		}
	}

	@Override
	public void deleteRecommendations(int productId) {
		try {
			String url = recommendationServiceUrl + "?productId=" + productId;
			LOG.debug("Will call deleteRecommendations API on URL: {}", url);
			
			restTemplate.delete(url);
		} catch (HttpClientErrorException ex) {
			throw handleHttpClientException(ex);
		}
	}
	
	
	private String getErrorMessage(HttpClientErrorException ex) {
		try {
			return mapper.readValue(ex.getResponseBodyAsString(), HttpErrorInfo.class).getMessage();
		} catch (IOException ioex) {
			return ex.getMessage();
		}
	}
	
	private RuntimeException handleHttpClientException(HttpClientErrorException ex) {
		switch(HttpStatus.resolve(ex.getStatusCode().value())) {
		case NOT_FOUND:
			return new NotFoundException(getErrorMessage(ex));
			
		case UNPROCESSABLE_ENTITY:
			return new InvalidInputException(getErrorMessage(ex));
			
		default:
			LOG.warn("Got an unexpected HTTP error: {}, will rethrow it", ex.getStatusCode());
			LOG.warn("Error body: {}", ex.getResponseBodyAsString());
			return ex;
		}
	}

}

```

#### Update `RecommendationServiceImpl` in the ``recommendation-service`` to include implementations for ___``createRecommendation```___ and ___``deleteRecommendations``___

This is the actual code that does the operations!

```java
package com.david.microservices.alpha.recommendation.services;

import java.util.ArrayList;
import java.util.List;

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
	public List<Recommendation> getRecommendations(int productId) {
		
		if(productId < 1) {
			throw new InvalidInputException("Invalid productId: " + productId);
		}
		
		if(productId == 113) {
			LOG.debug("No recommendations found for productId: {}", productId);
			return new ArrayList<>();
		}
		
		List<Recommendation> list = new ArrayList<>();
		list.add(new Recommendation(productId, 1, "Author 1", 1, "Content R1", serviceUtil.getServiceAddress()));
		list.add(new Recommendation(productId, 2, "Author 2", 2, "Content R2", serviceUtil.getServiceAddress()));
		list.add(new Recommendation(productId, 3, "Author 3", 3, "Content R3", serviceUtil.getServiceAddress()));
		
		LOG.debug("/recommendations response size: {}", list.size());
		
		return list;
	}

	@Override
	public Recommendation createRecommendation(Recommendation body) {
		try {
			RecommendationEntity entity = mapper.apiToEntity(body);
			RecommendationEntity newEntity = repo.save(entity);
			
			LOG.debug("createRecommendation: created a recommendation entity: {}/{}", body.getProductId(), body.getRecommendationId());
			return mapper.entityToApi(newEntity);
		} catch (DuplicateKeyException dke) {
			throw new InvalidInputException("Duplicate key, Product Id: " + body.getProductId() + ", Recommendation Id: " + body.getRecommendationId());
		}
	}

	@Override
	public void deleteRecommendations(int productId) {
		LOG.debug("deleteRecommendations: tries to delete recommendations for the product with productId: {}", productId);
		repo.deleteAll(repo.findByProductId(productId));
	}

}

```
