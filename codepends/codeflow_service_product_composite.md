### Code Flow: `product-composite microservice`
> Oct 7, 2024


#### Update the `ProductCompositeService` in the ___api___ library project to define the ___`create`___ and ___`delete`___ product operations

```java
package com.david.microservices.alpha.api.composite.product;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;

public interface ProductCompositeService {
	
	/**
	 * Sample usage: "curl $HOST:$PORT/product-composite/1"
	 * 
	 * @param productId - id of the product
	 * @return the composite product info, if found, else null
	 * 
	 */
	
	@Operation(
			summary = "${api.product-composite.get-composite-product.description}",
			description = "${api.product-composite.get-composite-product.notes}")
	@ApiResponses(value = {
			@ApiResponse(responseCode = "200", description = "${api.responseCodes.ok.description}"),
			@ApiResponse(responseCode = "400", description = "${api.responseCodes.badRequest.description}"),
			@ApiResponse(responseCode = "404", description = "${api.responseCodes.notFound.description}"),
			@ApiResponse(responseCode = "422", description = "${api.responseCodes.unprocessableEntity.description}")
	})
	@GetMapping(value = "/product-composite/{productId}", produces = "application/json")
	ProductAggregate getProduct(@PathVariable int productId);
	
	@Operation(summary = "${api.product-composite.create-composite-product.description}", description = "{api.product-composite.create-composite-product.notes}")
	@ApiResponses(value = {
			@ApiResponse(responseCode = "400", description = "${api.responseCodes.badRequest.description}"),
			@ApiResponse(responseCode = "422", description = "${api.responseCodes.unprocessableEntity.description}")
	})
	@PostMapping(value = "/product-composite", consumes = "application/json")
	void createProduct(@RequestBody ProductAggregate body);
	
	@Operation(summary = "${api.product-composite.delete-product-composite.description}", description = "${api.product-composite.delete-product-composite.notes}")
	@DeleteMapping(value = "/product-composite/{productId}")
	void deleteProduct(@PathVariable int productId);
}

```


The api documentations are parameterized, Check them [here](./conf/config_yaml_composite.md) for more in-depth info.


The `create` and `delete` mappings have to be implemented in the concrete class ___`ProductCompositeServiceImpl`___ in the ```product-composite-service```


But first, let's first override methods for the other services (`Product`, `Recommendation` and `Review`) in the ___`ProductCompositeIntegration`___ class:

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

	@Override
	public Product getProduct(int productId) {
		try {
			String url = productServiceUrl + productId;
			
			LOG.debug("Will call getProduct API on URL: {}", url);
			
			Product product = restTemplate.getForObject(url, Product.class);
			LOG.debug("Found a product with id: {}", product.getProductId());
			return product;
		} catch (HttpClientErrorException ex) {
			switch(HttpStatus.resolve(ex.getStatusCode().value())) {
			case NOT_FOUND:
				throw new NotFoundException(getErrorMessage(ex));
				
			case UNPROCESSABLE_ENTITY:
				throw new InvalidInputException(getErrorMessage(ex));
				
			default:
				LOG.warn("Got an unexpected unexpected HTTP error: {}, will rethrow it", ex.getStatusCode());
				LOG.warn("Error body: {}", ex.getResponseBodyAsString());
				throw ex;
			}
		}
	}
	
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
	public List<Review> getReviews(int productId) {
		
		try {
			String url = reviewServiceUrl + productId;
			
			LOG.debug("Will call getReviews API on URL: {}", url);
			List<Review> reviews = restTemplate.exchange(url, GET, null, new ParameterizedTypeReference<List<Review>>() {}).getBody();
			
			LOG.debug("Found {} reviews for a product with id: {}", reviews.size(), productId);
			
			return reviews;
		} catch (Exception ex) {
			LOG.warn("Got an exception while requesting reviews, return zero reviews: {}", ex.getMessage());
			return new ArrayList<>();
		}
	}
	
	private String getErrorMessage(HttpClientErrorException ex) {
		try {
			return mapper.readValue(ex.getResponseBodyAsString(), HttpErrorInfo.class).getMessage();
		} catch (IOException ioex) {
			return ex.getMessage();
		}
	}

	@Override
	public Review createReview(Review body) {
		
		try {
			String url = reviewServiceUrl;
			LOG.debug("Will post a new review to URL: {}", url);
			
			Review review = restTemplate.postForObject(url, body, Review.class);
			LOG.debug("Created a review with id: {}", review.getProductId());
			
			return review;
		} catch (HttpClientErrorException ex) {
			throw handleHttpClientException(ex);
		}
	}

	@Override
	public void deleteReviews(int productId) {
		try {
			String url = reviewServiceUrl + "?productId=" + productId;
			LOG.debug("Will call the deleteReviews API on URL: {}", url);
			
			restTemplate.delete(url);
			
		} catch (HttpClientErrorException ex) {
			throw handleHttpClientException(ex);
		}
	}

	@Override
	public Product createProduct(Product body) {
		try {
			String url = productServiceUrl;
			LOG.debug("Will post a new product to URL: {}", url);
			
			Product product = restTemplate.postForObject(url, body, Product.class);
			LOG.debug("Created a product with id: {}", product.getProductId());
			
			return product;
		} catch (HttpClientErrorException ex) {
			throw handleHttpClientException(ex);
		}
	}

	@Override
	public void deleteProduct(int productId) {
		try {
			String url = productServiceUrl + "/" + productId;
			LOG.debug("Will call the deleteProduct API on URL: {}", url);
			
			restTemplate.delete(url);
			
		} catch (HttpClientErrorException ex) {
			throw handleHttpClientException(ex);
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


In the ___`ProductCompositeServiceImpl`___ class, which is a controller that implements definitions of the ``ProductCompositeService``, we override the ``createProduct`` and ``deleteProduct``:



