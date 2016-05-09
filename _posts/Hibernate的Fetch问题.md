title: Hibernate的Fetch问题
date: 2016-04-21 15:40:04
tags:
- JPA
- Spring Data JPA
- Hibernate
---

下面的内容基于JPA的Hibernate实现进行讨论。

JPA的关联关系中的single-ended的关联（也就是ToOne）的模式fetch类型是的eager的。 在使用`EntityMangerd#find`方法进行查找时，这些关联会通过JOIN的方式被查询出来。但是如果使用Criteria API或者JPQL进行查询时，会发现针对这些关联产生了额外的SELECT查询，也就是通常我们所说的N+1的问题。

<!-- more -->

## Hibernate Fetching strategies

这里需要先讨论下Hibernate的fetch策略的问题。在Hibernate的映射文件（或者注解）中可以通过fetch属性来告诉Hibernate如何fetch关联对象。

```
<set name="permissions"
            fetch="join">
    <key column="userId"/>
    <one-to-many class="Permission"/>
</set
<many-to-one name="mother" class="Cat" fetch="join"/>
```

上面的例子中，使用join的方式来fetch mother属性，也间接的表明mother关联是eager的。 但是这里声明的fetch策略仅仅对下面几种情况有效

- 通过`get()`或者`load()`方法获取
— 当被作为关联引用到的时候产生的隐式获取
- `Criteria`查询
- 使用HQL查询并且声明的是subselect fetching

既然映射文件中声明的fetch策略对Criteria查询有效，但是为何使用JPA的Criteria接口时又没有使用JOIN来关联呢？

### Hibernate对JPA Criteria接口的实现

Hibernate对JPA的Criteria接口返回的`TypedQuery`实现类是`CriteriaQueryTypeQueryAdapter`，可以看到Hibernate对JPA返回的Criteria对象的编译的过程实际是转换成JPQL的查询。

```
public List<X> getResultList() {
	return jpqlQuery.getResultList();
}
```

而Hibernate对于JPQL的实现则最终又是通过HQL来实现了。这就可以解释其原因了。那么对于使用JPA接口的情况下，只有3中情况可以使用到映射文件中定义的fetch策略。

### Eager对HQL的影响

对于声明成eager的关联，那么他们最终一定会去获取到这个关联。使用HQL的情况，如果HQL中通过FETCH JOIN来进行了关联，那并没有什么问题。 但是如果没有关联，那么Hibernate则会额外产生一条SELECT语句来对eager的关联进行进行查找。 也就是说如果实体中有eager关联的属性，为了避免产生N+1问题，不管我们是否需要这个关联属性必须显示的FETCH这个关联。

## EntityGraph

EntityGraph可以运行时动态的覆盖映射文件中定义的FetchType。实际加载EntityGraph有两种方式

- FETCH 对于EntityGraph声明的属性使用eager，未声明的属性使用lazy
- LOAD  对于EntityGraph声明的属性使用eager，未声明的属性使用他们在映射文件中声明的默认的fetch类型

Hibernate对于EntityGraph的实现是使用了FetchProfile。但是FetchProfile本身实在EntityGraph之前诞生的，它用于覆盖映射文件的中fetchMode，而不是fetchType。 也就导致了使用Hibernate作为JPA实现时，FETCH和LOAD两种方式是一样的，并不能覆盖原来在映射文件中的eager为lazy。作为原因就是上面提到的JPQ中无论是Criteria API还是JPQL都是使用HQL来实现的，而HQL是无法实现覆盖eager关联的。 [这里][1]是相关问题的讨论。

最终在EntityGraph中声明的属性相当于声明了它需要使用JOIN来进行fetch。

## EAGER fetching不一致

通过上面的内容我们可以发现，对于eager的关联，在不用的使用场景下（find，JPQL），会产生不一致的结果。所以eager join看起来是一个代码中的[坏味道][2]。 所以推荐在global fetch plan（映射文件）中，定义所有的关联为lazy，而在每个具体的查询中来定义fetch策略。

下面是用Hibernate官方文档中的一段引用

> Usually, the mapping document is not used to customize fetching. Instead, we keep the default behavior, and override it for a particular transaction, using left join fetch in HQL.

## Spring Data JPA中的实现

Spring Data JPA可以通过接口的Query Method方法签名自动的来实现JPA的调用逻辑，它的调用是通过JPA Criteria API来实现的，所以最终也是无法使用映射文件文件的fetch策略的。推荐的方式是使用@Query的话直接在JPQL的声明需要fetch的关联；使用Spring Data JPA自动生成的时候使用EntityGraph来fetch。


[1]: https://hibernate.atlassian.net/browse/HHH-8776
[2]: https://dzone.com/articles/eager-fetching-code-smell