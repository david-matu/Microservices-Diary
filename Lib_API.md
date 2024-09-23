### This is the `api` library documentation
> Sep 23, 2024

Project Structure:
```sh
cd /home/dave/Documents/Workspace-Microservices-alpha/api
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
###### `com.david.microservices.alpha.api.core.product`
1. Product.java [:link:](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/product/Product.java)
2. ProductService.java (interface) [:link:](https://github.com/david-matu/product-microservices/blob/main/api/src/main/java/com/david/microservices/alpha/api/core/product/ProductService.java)


###### `com.david.microservices.alpha.api.core.recommendation`
1. Recommendation.java
2. RecommendationService.java


###### `com.david.microservices.alpha.api.core.review`
1. Review.java
2. ReviewService.java


###### `com.david.microservices.alpha.api.exceptions`
1. BadRequestException.java
2. InvalidInputException.java
3. NotFoundException.java


###### `com.david.microservices.alpha.api.composite.ProductService`
1. ProductAggregate.java
2. ProductCompositeService.java
3. RecommendationSummary.java
4. ReviewSummary.java
5. ServiceAddresses.java



