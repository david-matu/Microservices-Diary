```sh
[dave@localhost api]$ tree ./
./
├── bin
│   ├── main
│   │   └── com
│   │       └── david
│   │           └── microservices
│   │               └── alpha
│   │                   └── api
│   │                       ├── composite
│   │                       │   └── product
│   │                       │       ├── ProductAggregate.class
│   │                       │       ├── ProductCompositeService.class
│   │                       │       ├── RecommendationSummary.class
│   │                       │       ├── ReviewSummary.class
│   │                       │       └── ServiceAddresses.class
│   │                       ├── core
│   │                       │   ├── product
│   │                       │   │   ├── Product.class
│   │                       │   │   └── ProductService.class
│   │                       │   ├── recommendation
│   │                       │   │   ├── Recommendation.class
│   │                       │   │   └── RecommendationService.class
│   │                       │   └── review
│   │                       │       ├── Review.class
│   │                       │       └── ReviewService.class
│   │                       └── exceptions
│   │                           ├── BadRequestException.class
│   │                           ├── InvalidInputException.class
│   │                           └── NotFoundException.class
│   └── test
│       └── com
│           └── david
│               └── microservices
│                   └── alpha
│                       └── api
├── build
│   ├── classes
│   │   └── java
│   │       └── main
│   │           └── com
│   │               └── david
│   │                   └── microservices
│   │                       └── alpha
│   │                           └── api
│   │                               ├── composite
│   │                               │   └── product
│   │                               │       ├── ProductAggregate.class
│   │                               │       ├── ProductCompositeService.class
│   │                               │       ├── RecommendationSummary.class
│   │                               │       ├── ReviewSummary.class
│   │                               │       └── ServiceAddresses.class
│   │                               ├── core
│   │                               │   ├── product
│   │                               │   │   ├── Product.class
│   │                               │   │   └── ProductService.class
│   │                               │   ├── recommendation
│   │                               │   │   ├── Recommendation.class
│   │                               │   │   └── RecommendationService.class
│   │                               │   └── review
│   │                               │       ├── Review.class
│   │                               │       └── ReviewService.class
│   │                               └── exceptions
│   │                                   ├── InvalidInputException.class
│   │                                   └── NotFoundException.class
│   ├── generated
│   │   └── sources
│   │       ├── annotationProcessor
│   │       │   └── java
│   │       │       └── main
│   │       └── headers
│   │           └── java
│   │               └── main
│   ├── libs
│   │   └── api-1.0.0-SNAPSHOT.jar
│   └── tmp
│       ├── compileJava
│       │   ├── compileTransaction
│       │   │   ├── backup-dir
│       │   │   └── stash-dir
│       │   │       └── RecommendationService.class.uniqueId0
│       │   └── previous-compilation-data.bin
│       └── jar
│           └── MANIFEST.MF
├── build.gradle
├── HELP.md
├── settings.gradle
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── david
    │   │           └── microservices
    │   │               └── alpha
    │   │                   └── api
    │   │                       ├── composite
    │   │                       │   └── product
    │   │                       │       ├── ProductAggregate.java
    │   │                       │       ├── ProductCompositeService.java
    │   │                       │       ├── RecommendationSummary.java
    │   │                       │       ├── ReviewSummary.java
    │   │                       │       └── ServiceAddresses.java
    │   │                       ├── core
    │   │                       │   ├── product
    │   │                       │   │   ├── Product.java
    │   │                       │   │   └── ProductService.java
    │   │                       │   ├── recommendation
    │   │                       │   │   ├── Recommendation.java
    │   │                       │   │   └── RecommendationService.java
    │   │                       │   └── review
    │   │                       │       ├── Review.java
    │   │                       │       └── ReviewService.java
    │   │                       └── exceptions
    │   │                           ├── BadRequestException.java
    │   │                           ├── InvalidInputException.java
    │   │                           └── NotFoundException.java
    │   └── resources
    └── test
        └── java
            └── com
                └── david
                    └── microservices
                        └── alpha
                            └── api

74 directories, 48 files
```
