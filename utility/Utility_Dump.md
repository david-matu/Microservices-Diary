Utility_Dump.md

```java
	@Override
	public Mono<Void> deleteProduct(int productId) {
		try {
			LOG.info("Will delete a product aggregate for product.id: {}", productId);
			
			
			return Mono.zip(
					r -> "",
					integration.deleteProduct(productId),
					integration.deleteRecommendations(productId),
					integration.deleteReviews(productId))
					.doOnError(ex -> LOG.warn("delete failed: {}", ex.toString()))
					.log(LOG.getName(), FINE).then();
					
			
			/*
			 * Use this in cases when the methods i.e. integration.deleteReviews(productId) don't return Mono<Void> type 
			return Mono.zip(
			        integration.deleteProduct(productId),
			        Mono.fromRunnable(() -> integration.deleteRecommendations(productId)), // Wrap in Mono
			        Mono.fromRunnable(() -> integration.deleteReviews(productId)))         // Wrap in Mono
			        .map(tuple -> "") // Use map to transform the zipped results
			        .doOnError(ex -> LOG.warn("delete failed: {}", ex.toString()))
			        .log(LOG.getName(), FINE).then();
			 */

		} catch (RuntimeException rex) {
			LOG.warn("deleteCompositeProduct failed: {}", rex.toString());
		}
		return Mono.empty();
	}

```