---
title: Spring Cache Learning Note
tags:
 - Learning
 - Note
 - Redis
 - Spring_Cache
 - Spring 
create_time: 2026-02-10
---

# Introduction
- "Spring Cache" is a framework that implements caching functionality based on **annotations**. All you need to do is simply add an annotation, and the caching function can be achieved.
- `Spring Cache` provides an abstraction layer, and at the underlying level, different caches can be switched to achieve this, for example:
	- `EHCache`
	- `Caffeine`
	- `Redis`

> [!warning] Notice
> If you choose `Redis`, you must ensure your entity class that implement `Serializable`.
> (Unless you configure a custom JSON Serializer)

- **Introduce dependencies**
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-cache</artifactId>
	<version>2.7.3</version>
</dependency>
```

---

## Common Annotations
| Annotation | Statement |
|---|---|
|`@EnableCaching`|Enable the cache annotation feature, usually added to the startup class.|
|`@Cacheable`|Before executing the method, first check if there is any data in the cache. If there is data, directly return the cached data. If there is no cached data, call the method and store the return value of the method in the cache.|
|`@CachePut`|Put the return value of the method into the cache.|
|`@CacheEvict`|Delete one or more pieces of data from the cache.|

---

## Business Scenario

> [!warning] Notice:
> Remind yourself to add `@EnableCaching` to your project startup class.

### `@CachePut` Simple Example
```Java
/* UserController.java */
@PostMapping
// compute cache key is `user::{user.id}`
@CachePut(cacheNames = "user", key = "#result.id")
public User save(User user) {
	userService.save(user); // useGenerateKey="true", keyProperties="id"
	return user;
}
```
> [!info] ℹ Information
> The `key` value can be computed by `SpEL` expression. (from Spring framework)
> Also, `#user` refers to the method parameter.


> [!tip] 💡 Extension
>  You can use `#result` to use returned value.(only in @CachePut or @CacheEvict)
> Such as:
> ```Java
> @CachePut(cacheName="user", key="#result.id")
> ```
> Other forms for accessing parameters:
> ```Java
> // obtain the first object of the params(if params is not unique)
> @CachePut(cacheName="user", key="#p0.id")
> @CachePut(cacheName="user", key="#a0.id") // same as above
> @CachePut(cacheName="user", key="#root.args[0].id") // same as above
> ```
> **PS**: you must notice that the params array index begin from zero.

### `@CacheEvict` Simple Example
```Java
/* UserController.java */
@DeleteMapping
@CacheEvict(cacheNames="user", key="#id") // only delete one data.
public void deleteById(Long id) {
	userService.deleteById(id);
}

@DeleteMapping("/deleteAll")
@CacheEvict(cacheNames="user", allEntries=true) // delete all user cache.
public void deleteAll() {
	userService.deleteAll();
}
```

---

## Tip
> [!info] About Cache Annotation
> Usually add to `ServiceImpl` Layer.
> More, you need to add public method. Be aware that **internal method calls** (calling a method within the same class) will cause Cache Annotations to fail due to AOP proxy limitations.

> [!tip] About `@Cacheable`
> This annotation's key can't use `#result`
> The `key` generation happens **before** the method execution (to check if the cache exists). Therefore, the `#result` (which is produced **after** execution) is not available for generating the key.