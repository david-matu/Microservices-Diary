#### `application.yml`
###### Forked from ___```src/main/resources```___ of the __`product-composite-service`__

```yaml
server:
 port: 7000
 error.include-message: always
 
spring:
 application.name: product-composite-service

# This configuration (below) will be replaced by a service discovery mechanism
app:
 product-service:
  host: localhost
  port: 7001
  
 recommendation-service:
  host: localhost
  port: 7002
  
 review-service:
  host: localhost
  port: 7003

api:
 common:
  title: Product Service API
  description: This is a Product Service (demo) API where a group of cooperating microservices are aggregated to return information about a given product
  version: 1.0.0
  
  termsOfService: https://david-matu.github.io/alpha-microservices/terms-of-service
  license: General Usage License
  licenseUrl: https://david-matu.github.io/alpha-microservices/license
  
  externalDocDesc: Find out the public info about this API
  externalDocUrl: https://david-matu.github.io/alpha-microservices
  
  contact:
   name: David Matu
   url: https://david-matu.github.io
   email: davidmatu817@gmail.com
   
  responseCodes:
   ok.description: OK
   badRequest.description: Bad Request, invalid format of the request. See response message for more information 
   notFound.description: Not found. The specified id does not exist 
   unprocessableEntity.description: Uprocessable entity, input parameters caused the processing to fail. See response message for more information
 
 product-composite: 
  get-composite-product:
   description: Returns a composite view of the specified product id
   notes: |
    # Normal response
    If the requested product id is found, the method will return information regarding:
    1. Base product information
    2. Reviews
    3. Recommendations
    4. Service Addresses\n(technical information regarding the addresses pf the microservices that created the response)
    
    # Expected partial and error responses
    
    ## Product id 113
    200 - OK, but ni recommendations will be returned
    
    ## Product id 213
    200 - OK, but no reviews will be returned
    
    ## Non-numerical product-id
    400 - A **Bad Request** error will be returned
    
    ## Product id 13
    404 - A **Not FOund** error will be returned
    
    ## Negative product ids
    422 - An **Unprocessable Entity** error will be returned
  
  create-composite-product:
   description: Creates a composite product
   notes: |
    # Normal response
    The composite product information posted to the API will be split up and stored as separate product-info, recommendation and review entities.
    
    # Expected error responses
    1. If a product with the same productId as specified in the posted information already exists, a **422** - Unprocessable Entity** error with a "duplicate key" error message will be returned
    
  delete-product-composite:
   description: Deletes a product composite
   notes: |
    # Normal response
    Entities for product information, recommendations and reviews related to te specified productId will be deleted.
    The implementation of the delete method is indempotent -- it can be called several times with the same response.
    
    This means that a delete request for a non-existing product will return**200 Ok**.
   
springdoc:
 swagger-ui.path: /openapi/swagger-ui.html
 api-docs.path: /openapi/v3/api-docs
 packagesToScan: com.david.microservices.alpha.composite.product
 pathsToMatch: /**
 
---
spring:
 config.activate.on-profile: docker

server.port: 8080

app:
 product-service:
  host: product
  port: 8080
  
 recommendation-service:
  host: recommendation
  port: 8080
  
 review-service:
  host: review
  port: 8080

```