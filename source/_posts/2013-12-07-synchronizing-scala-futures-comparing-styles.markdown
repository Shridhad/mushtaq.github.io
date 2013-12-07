---
layout: post
title: "Synchronizing Scala Futures: Comparing Styles"
date: 2013-03-09 16:02
comments: true
categories: [scala, futures]
---

You deal with [futures](http://docs.scala-lang.org/overviews/core/futures.html) all the time if you are using [play framework](http://www.playframework.com/). For example, call to external webservices or mongodb via [reactivemongo](http://reactivemongo.org/) driver returns future. Consider the signatures of two apis from repository classes wrapping mongodb access:

``` scala
productRepo.getProduct(name: String): Future[Product]

metadataRepo.getProductMetadata(productId: String): Future[Metadata]
```

You will have to use the result of the first call to obtain the productId required for the second call. But the return type for both the calls are futures, so how to use the product value inside a future without blocking it. Well, using higher order function flatMap defined on futures, you can do something like:

``` scala
val metadata: Future[Metadata] = 
	productRepo.getProduct("watch").flatMap { product =>
		metadataRepo.getProductMetadata(product.id)
	}
```

There are a lot of other useful combinators defined for futures like map, filter, recover etc. Here, flatMap ensure that call to getProductMetadata will not happen unless future returned by getProduct is successful. An important insight is that composing futures using combinators like flatMap is essentially synchronizing futures without blocking. 

We know that flatMap operations can also be represented as for comprehension. Like this:

``` scala
val metadata: Future[Metadata] = for {
	product <- productRepo.getProduct("watch")
	metadata <- metadataRepo.getProductMetadata(product.id)
} yield metadata
```

Unlike flatMap invloves for does not do invert the program flow. Hence, in general code using for is more readable than directly using flatMap. This may not be the case always. For example if we want to throw exception if product is not available in the store, for approach is more complicated compared to flatMap:

``` scala
val metadata: Future[Metadata] = for {
	product <- productRepo.getProduct("watch")
	metadata <- {
		if (product.isNotAvailable)
			throw new ProductNotAvailableException("product not found")
		metadataRepo.getProductMetadata(product.id)
	}
} yield metadata
```

There is an alternate way to synchronize futures using continuations also known as dataflow style. We will have to use [akka's dataflow library](http://doc.akka.io/docs/akka/2.1.0/scala/dataflow.html) which provides a flow block inside which call to apply() on futures returns the contained element (it does not actually return but it does look like that). This style is somewhat similar to for comprehensions but allows mixing imperative code with dataflow style very easy. The above program then can be written as:

``` scala
val metadata: Future[Metadata] = flow {
	val product = productRepo.getProduct("watch")()
	if (product.isNotAvailable)
		throw new ProductNotAvailableException("product not found")
	metadataRepo.getProductMetadata(product.id)()
}
```

We have been using dataflow style with futures a lot in our project and it has worked very well so far. Continuations bring in their own constraints on the structure of your code. Compiler generated error messages when you violate those constraints are intimidating. We will discuss how to deal with them in a future post.

