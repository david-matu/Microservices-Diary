## `util` library documentation
> Sep 23, 2024

This library project is based on Gradle and uses the `api` project.

#### `settings.gradle`
To resolve the dependency on `api` project, the following `settings.gradle` content has been observed:
```sh
rootProject.name = 'util'
include('api')
project(':api').projectDir = new File(settingsDir, '../api')
```

#### `build.gradle`
```sh
plugins {
	id 'io.spring.dependency-management' version '1.1.5'
	id 'java'
}

group = 'com.david.microservices.alpha.util'
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

    implementation(project(':api'))
    
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
	useJUnitPlatform()
}
```

### Project Structure
#### `com.david.microservices.alpha.util.http`
* [GlobalControllerExceptionHandler.class](https://github.com/david-matu/product-microservices/blob/main/util/src/main/java/com/david/microservices/alpha/util/http/GlobalControllerExceptionHandler.java)
* [HttpErrorInfo.class](https://github.com/david-matu/product-microservices/blob/main/util/src/main/java/com/david/microservices/alpha/util/http/HttpErrorInfo.java)
* [ServiceUtil.class](https://github.com/david-matu/product-microservices/blob/main/util/src/main/java/com/david/microservices/alpha/util/http/ServiceUtil.java)
