---
title: Cache-Aside模式【翻译】
urlname: cache-aside-pattern
mathjax: true
date: 2023-03-09 10:25:00
toc: true
tags:
- 翻译
- MSDN
- cache
---

>翻译系列第一篇，MSDN一篇介绍缓存模式的文档，原文链接：[Cache-Aside Pattern](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn589799(v=pandp.10))

<!-- more -->

按需将数据从数据存储加载到缓存中。使用这种模式可以提高性能，并且还有助于使缓存中的数据与底层数据存储之间保持一致性。

## 背景和问题
应用程序使用缓存来优化对数据存储中的数据的重复访问。 但是，期望缓存数据始终与数据存储中的数据完全一致通常是不切实际的。 应用程序应该采取某种策略，以尽可能使缓存中的数据保持最新，而当缓存中的数据过期时也可以检测和处理。

## 解决方案
许多商业缓存系统提供`read-through`和`write-through/write-behind`操作。 在这样的系统中，应用程序通过缓存来检索数据。 如果数据不在缓存中，它会从数据存储中检索数据并添加到缓存中，这个过程对应用程序是透明的。 对缓存中的数据进行任何修改也会自动写回到数据存储中。

对于不提供此功能的缓存，使用缓存的应用程序负责维护缓存中的数据。

应用程序可以使用`Cache-Aside`策略来模拟`read-through`缓存操作。 该策略能够高效地将数据按需加载到缓存中。 图 1 总结了这个过程中的步骤。
![图 1 - 使用 Cache-Aside 模式将数据存储在缓存中](/images/cache-aside-pattern/dn589799.55b56fd8930e405c5bf9580e455a16c1(en-us,pandp.10).png)

如果一个应用程序要更新数据，它可以像下面这样模拟`write-through`策略：
1. 对数据存储进行修改
2. 使缓存中的对应项失效

当下次访问该数据时，`Cache-Aside`策略会从数据存储中检索最新的数据并将其添加回缓存中。

## 问题和注意事项
在实现此模式时，需要考虑以下几点：
- **缓存数据的生命周期。** 许多缓存使用过期机制，缓存数据在一段时间内没被访问就会失效并从缓存中删除。 要使`Cache-Aside`模式有效，需要确保过期机制与数据的访问模式相匹配。 不要将过期时间设置得太短，因为这会导致应用程序不断从数据存储中检索数据并将其添加到缓存中。 同样，不要将过期时间设置得太长，以免缓存的数据过时。 请记住，缓存对于相对静态的数据或频繁读取的数据最有效。
- **缓存淘汰。** 与数据存储相比，大多数缓存的大小比较小，必要时会淘汰缓存数据。 大多数缓存采用最近最少使用（LRU）的策略来选择要淘汰的数据，不过一般是可以配置的。 可以通过配置缓存的全局过期属性和其他属性，以及每个缓存项的过期属性，来提高缓存的成本效益。 有时候将全局淘汰策略应用于缓存中的每个项目可能并不总是合适的。 例如，如果一个缓存项目从数据存储中检索起来非常昂贵，那么可能牺牲其它访问更频繁但成本较低的项目从而将这个项目保留在缓存中比较好。
- **缓存预热。** 许多解决方案都是将应用程序可能需要的数据预先填充到缓存中，作为启动过程的一部分。即使其中一些数据过期或被淘汰，`Cache-Aside`模式仍然有用。
- **一致性。** 使用`Cache-Aside`模式并不能保证数据存储和缓存之间的一致性。 数据存储中的一个项目可能在任何时候被外部进程修改，而这个改变可能不会反映在缓存中，直到下一次该项目被加载到缓存中。在一个跨数据存储区复制数据的系统中，如果同步发生得非常频繁，这个问题可能会变得特别严重。
- **本地（内存中）缓存。** 缓存可以是应用程序实例的本地缓存，并存储在内存中。 如果应用程序频繁访问相同的数据，则`Cache-Aside`模式在此环境中很有用。 但是，本地缓存是私有的，因此不同的应用程序实例可以各自拥有相同缓存数据的副本。 此数据可能会在缓存之间很快变得不一致，因此可能有必要使本地缓存中保存的数据更频繁地过期和更新。 在这些情况下，可能更适合考虑使用共享或分布式缓存。

## 什么时候使用此模式
在以下情况下使用此模式：
- 缓存系统本身不提供`read-through`和`write-through`操作。
- 资源需求是不可预测的。 此模式让应用程序按需加载数据，它不会预先假设应用程序需要哪些数据。

这种模式可能不适用于：
- 缓存的数据集是静态的。 如果数据适用于可用的缓存空间，则应该在启动时用数据填充缓存并防止数据过期。
- 用于缓存 Web 应用程序的会话状态信息。在这种环境下，应该避免引入基于客户端-服务器亲和性的依赖关系。

## 例子
在 Microsoft Azure 中，您可以使用 Azure 缓存创建分布式缓存，该缓存可由应用程序的多个实例共享。 以下代码示例中的`GetMyEntityAsync`函数演示了基于 Azure 缓存的`Cache-Aside`模式的实现。 此函数使用`read-though`方法从缓存中检索对象。

每个对象用一个整数 ID 作为键来标识。 `GetMyEntityAsync`函数基于此键生成一个字符串值（Azure 缓存 API 使用字符串作为键值）并尝试从缓存中检索具有此键的项目。 如果找到匹配项，则将其返回。 如果缓存中没有匹配项，则`GetMyEntityAsync`函数从数据存储中检索对象，将其添加到缓存中，然后返回它（实际从数据存储中检索数据的代码已被省略，因为它依赖于数据存储）。 要注意的地方是，缓存的项目设置了过期时间，以防止它在其他地方被更新从而过时。
```C#
private DataCache cache;...public async Task<MyEntity> GetMyEntityAsync(int id) {    
    // Define a unique key for this method and its parameters.  
    var key = string.Format("StoreWithCache_GetAsync_{0}", id);  
    var expiration = TimeSpan.FromMinutes(3);  
    bool cacheException = false;  
    try {    
        // Try to get the entity from the cache.    
        var cacheItem = cache.GetCacheItem(key);    
        if (cacheItem != null) {      
            return cacheItem.Value as MyEntity;    
        }  
    } catch (DataCacheException) {   
        // If there is a cache related issue, raise an exception     
        // and avoid using the cache for the rest of the call.    
        cacheException = true;  
    }  
    // If there is a cache miss, get the entity from the original store and cache it.  
    // Code has been omitted because it is data store dependent.    
    var entity = ...;  
    if (!cacheException) {    
        try {      
            // Avoid caching a null value.      
            if (entity != null) {        
                // Put the item in the cache with a custom expiration time that         
                // depends on how critical it might be to have stale data.        
                cache.Put(key, entity, timeout: expiration);      
            }    
        } catch (DataCacheException) {      
            // If there is a cache related issue, ignore it      
            // and just return the entity.    
        }  
    }  
    return entity;
}
```
> **注意**
> 这些示例使用 Azure 缓存 API 访问存储并从缓存中检索信息。 有关 Azure 缓存 API 的更多信息，请参阅 MSDN 上的[使用 Microsoft Azure 缓存](https://learn.microsoft.com/en-us/previous-versions/azure/azure-services/hh914165(v=azure.100)?redirectedfrom=MSDN)。

下面的`UpdateEntityAsync`函数演示了如何在应用程序更改值时使缓存中的对象失效。 这是`write-through`方法的一个例子。 代码更新原始数据存储，然后通过调用`Remove`方法从缓存中删除对应键的缓存项（这部分功能的代码已被省略，因为它将依赖于数据存储）。

> **注意**
> 此过程中的**步骤顺序很重要**。 如果缓存项在数据更新之前被删除，则在数据存储中的项目被更改之前，客户端应用程序有一小段机会获取数据（因为它未在缓存中找到），从而过时的数据被添加到缓存中。
```C#
public async Task UpdateEntityAsync(MyEntity entity) {  
    // Update the object in the original data store  
    await this.store.UpdateEntityAsync(entity).ConfigureAwait(false);  
    // Get the correct key for the cached object.  
    var key = this.GetAsyncCacheKey(entity.Id);  
    // Then, invalidate the current cache object  
    this.cache.Remove(key);
} 
private string GetAsyncCacheKey(int objectId) {  
    return string.Format("StoreWithCache_GetAsync_{0}", objectId);
}
```

## 相关模式和指南
在实现该模式时，以下模式和指南也可能相关：
- [缓存指南](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn589802(v=pandp.10))。 本指南提供了有关如何在云解决方案中缓存数据的信息以及实现缓存时应考虑的问题。
- [数据一致性入门指南](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn589800(v=pandp.10))。 云应用程序通常使用分散在数据存储中的数据。 在此环境中管理和维护数据一致性可能成为系统的一个关键方面，特别是可能出现并发性和可用性方面的问题。 本指南介绍了与分布式数据一致性有关的问题，并总结了应用程序如何通过实现最终一致性来保持数据的可用性。

## 更多信息
MSDN 上的文章[使用 Microsoft Azure 缓存](https://msdn.microsoft.com/library/windowsazure/hh914165.aspx)。