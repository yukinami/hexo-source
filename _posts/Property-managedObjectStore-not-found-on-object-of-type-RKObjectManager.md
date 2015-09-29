title: "Property 'managedObjectStore' not found on object of type'RKObjectManager *'"
date: 2015-09-29 14:34:03
tags:
- iOS
- Restkit
- 经验错误
---

问题
----
使用Restkit时，访问`RKObjectManger`的`managedObjectStore`属性是提示
> Property 'managedObjectStore' not found on object of type'RKObjectManager *'

<!--more-->

原因
----
看了下`RKObjectManger`的头文件中关于`managedObjectStore`属性的定义是这样的

```
#ifdef RKCoreDataIncluded
/**
 A Core Data backed object store for persisting objects that have been fetched from the Web
 */
@property (nonatomic, strong) RKManagedObjectStore *managedObjectStore;

/**
 An array of `RKFetchRequestBlock` blocks used to map `NSURL` objects into corresponding `NSFetchRequest` objects.
 
 When searched, the blocks are iterated in the reverse-order of their registration and the first block with a non-nil return value halts the search.
 */
@property (nonatomic, readonly) NSArray *fetchRequestBlocks;

/**
 Adds the given `RKFetchRequestBlock` block to the manager.
 
 @param block A block object to be executed when constructing an `NSFetchRequest` object from a given `NSURL`. The block has a return type of `NSFetchRequest` and accepts a single `NSURL` argument.
 */
- (void)addFetchRequestBlock:(NSFetchRequest *(^)(NSURL *URL))block;
#endif
```

同时这个宏变量是这样定义的

```
#ifdef _COREDATADEFINES_H
#if __has_include("RKCoreData.h")
#define RKCoreDataIncluded
#endif
```

`_COREDATADEFINES_H`这个宏是在CoreData的`CoreDataDefines.h`头文件中定义的

CoreDataDefines.h

```
#ifndef _COREDATADEFINES_H
#define _COREDATADEFINES_H
```

解决
----
需要在import Restkit，这样在import`RKObjectManger.h`时，才能正确定义宏以及属性。

```
#include <CoreData/CoreData.h>
```
总结
----
OC中的头文件引入顺序非常重要，会影响到一些实际代码特别是一些宏的定义。