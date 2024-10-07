### Code Flow: `product microservice`
> Oct 6, 2024


Update the __`ProductService`__ in the __api__ (`com.david.microservices.alpha.api.core.product`) package
```java
package com.david.microservices.alpha.api.core.product;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

/**
 * Expose getProduct() API method, which will be extended
 * 
 */
public interface ProductService {
	
	/**
	 * Usage: "curl $HOST:$PORT/product/1
	 * 
	 * @param productId - id of the product
	 * @return the product if found, else null
	 */
	@GetMapping(value = "/product/{productId}", produces = "application/json")
	Product getProduct(@PathVariable int productId);
	
	@PostMapping(value = "/product", consumes = "application/json", produces = "application/json")
	Product createProduct(@RequestBody Product body);
	
	@DeleteMapping(value = "/product/{productid}")
	void deleteProduct(@PathVariable int productId);
}
```

Into the __`product-service`__ microservice project:

#### `application.yml`
```yaml
spring:
 application.name: product-service
 
 data.mongodb:
  host: localhost
  port: 27017
  database: product-db
  
server:
 port: 7001
 error.include-message: always
 

logging:
 level:
  root: INFO
  com.david.microservices.alpha: DEBUG
  org.springframework.data.mongodb.core.MongoTemplate: DEBUG

---
spring:
 config:
  activate:
   on-profile: docker
 
 data.mongodb.host: mongodb

server:
 port: 8080
```

#### Define ___`ProductEntity`___
```java
package com.david.microservices.alpha.product.persistence;

import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.Version;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "products")
public class ProductEntity {
	@Id 
	private String id;
	
	@Version 
	private Integer version;
	
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

	// Getters and setters
}
```

#### Define ___`ProductRepository`___
```java
package com.david.microservices.alpha.product.persistence;

import java.util.Optional;

import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.PagingAndSortingRepository;

public interface ProductRepository extends PagingAndSortingRepository<ProductEntity, String>, CrudRepository<ProductEntity, String>{
	
	Optional<ProductEntity> findByProductId(int productId);
}
```

#### Define `ProductMapper`
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

#### Complete the implementation for ___`ProductServiceImpl`___
```java
package com.david.microservices.alpha.product.services;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.web.bind.annotation.RestController;

import com.david.microservices.alpha.api.core.product.Product;
import com.david.microservices.alpha.api.core.product.ProductService;
import com.david.microservices.alpha.api.exceptions.InvalidInputException;
import com.david.microservices.alpha.api.exceptions.NotFoundException;
import com.david.microservices.alpha.product.persistence.ProductEntity;
import com.david.microservices.alpha.product.persistence.ProductRepository;
import com.david.microservices.alpha.util.http.ServiceUtil;

@RestController
public class ProductServiceImpl implements ProductService {
	
	private static final Logger LOG = LoggerFactory.getLogger(ProductServiceImpl.class);
	
	private final ServiceUtil serviceUtil;
	
	private final ProductRepository repo;
	private final ProductMapper mapper;
	
	@Autowired
	public ProductServiceImpl(ProductRepository repository, ProductMapper mapper, ServiceUtil serviceUtil) {
		this.repo = repository;
		this.mapper = mapper;
		this.serviceUtil = serviceUtil;
	}
	
	@Override
	public Product getProduct(int productId) {
		
		LOG.debug("/product return the found product for productId={}", productId);
		
		if(productId < 1) {
			throw new InvalidInputException("Invalid productId: " + productId);
		}
		
		ProductEntity entity = repo.findByProductId(productId).orElseThrow(() -> new NotFoundException("No product found for productId: " + productId));
		
		Product response = mapper.entityToApi(entity);
		response.setServiceAddress(serviceUtil.getServiceAddress());
		
		LOG.debug("getProduct: found productId: {}", response.getProductId());
		
		return response;
	}

	@Override
	public Product createProduct(Product body) {
		try {
			ProductEntity entity = mapper.apiToEntity(body);
			ProductEntity newEntity = repo.save(entity);
			
			LOG.debug("createProduct: entity created for productId: {}", body.getProductId());
			return mapper.entityToApi(newEntity);
		} catch (DuplicateKeyException dke) {
			throw new InvalidInputException("Duplicate key, Product Id: " + body.getProductId());
		}
	}

	@Override
	public void deleteProduct(int productId) {
		LOG.debug("deleteProduct: tries to delete an entity with productId: {}", productId);
		repo.findByProductId(productId).ifPresent(e -> repo.delete(e));
	}
}

```

#### Complete the implementation for ___``___