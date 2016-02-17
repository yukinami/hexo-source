title: JPA fetch join & N+1
date: 2016-02-17 14:33:44
tags:
- Spring Data
- JPA
---

JPA对于关联属性的加载时机，可以通过`FetchType`来定义`EAGER`或者`LAZY`，但是对于加载的方式，是使用SELECT、SUBSELEC、JOIN则是由provider来决定的。JPA并没有提供相关的定义。例如，在使用Hibernate作为proider时，即便定义OneToMany的`FetchType`为`EAGER`, Hibernate确实会在加载对象的时候同时加载关联对象，但是却是通过SELECT方式来实现的，也就是我们常说的N+1问题，对性能有较大的影响。

<!--more-->

## 解决办法

通过JPQL

```
@Query("SELECT u FROM User u JOIN FETCH u.roles r")
```

或者Creteria API

```
new Specification<User>() {
    @Override
    public Predicate toPredicate( Root<User> root, CriteriaQuery<?> criteria, CriteriaBuilder builder ) {
        root.fetch( User_.roles );
        return null;
    }
}
```

## Spring Data JPA中的分页问题

但是如果上面两个查询，用于在Spring Data JPA中的Pagination的分页查询的话，又报如下的错误

```
query specified join fetching, but the owner of the fetched association was not present in the select list
```

意思是说，查询指定了join fetching，但是需要被fetched的关联的owner没有在select列表中出现。就比如说，我们指定了需要fetch User的roles属性，但是select中连user本身都没有出现，这个fetch就变的多余了。但是我们确实取了User对象

SimpleJpaRepository中对Query的构建

```
protected TypedQuery<T> getQuery(Specification<T> spec, Sort sort) {

	CriteriaBuilder builder = em.getCriteriaBuilder();
	CriteriaQuery<T> query = builder.createQuery(getDomainClass());

	Root<T> root = applySpecificationToCriteria(spec, query);
	query.select(root);

	if (sort != null) {
		query.orderBy(toOrders(sort, root, builder));
	}

	return applyRepositoryMethodMetadata(em.createQuery(query));
}
```

原因在于我们查询的是Page对象，Spring除了会使用差个查询获取数据外，还会使用这个查询进行count查询，但是那个查询中没有没指定User对象。

解决办法是如果发现针对count查询，不指定fetch


```
new Specification<User>() {
    @Override
    public Predicate toPredicate( Root<User> root, CriteriaQuery<?> criteria, CriteriaBuilder builder ) {
    	Class clazz = cq.getResultType();
    	if (clazz.equals(User.class)) {
    		root.fetch(User_.roles);
    	}
        
        return null;
    }
}
```

## 分页Fetch的性能问题

我们知道，理想的分页是过程在数据库查询阶段就指定条件来限制返回结果的件数的。 但是我们同时又需要去JOIN子表，因为每条主表记录关联子表的记录条数是不确定的，这也就导致需要取多少条数据库返回记录变得不确定了。最终会产生如下的警告

```
7 juin 2011 09:52:37 org.hibernate.hql.ast.QueryTranslatorImpl list ATTENTION: firstResult/maxResults specified with collection fetch; applying in memory!
```

也就是说，分页的操作不能在数据库端进行了，只能查询数据库的所有记录，然后在内存中进行分页，这在数据量庞大的主表的情况是不能接受的（通常我们需要分页，也就意味着主表的记录较多）

所以结论上讲，类似OneToMany这样会导致结果条数不确定的JOIN，在需要分页时，是不能使用的。 所以我们只能选择SELECT或者SUBSELECT。好在就算我们使用的JPA，Hibernate的FetchMode的还是可以使用的，为了避免N+1问题，只能使用SUBSELECT。

NOTES：在使用Hibernate作为provider的时候，FetchMode的JOIN并没有效果，原因未知。

另外，指定了SUBSELECT只是指定了FetchMode，FetchType本身仍可以指定为EAGER，但是不推荐直接修改定义关联注解上的属性，因为并不能保证每个查询User的地方都希望fetch roles属性。推荐EntityGraph来实现，针对不同的查询自定义知否需要eager fetch。

NOTES：在使用Hibernate作为provider的时候，通过EntityGraph来指定某个关联需要eager fetch时，会导致Hibernate使用JOIN来关联而忽视定义的FetchMode。所以分页的情况，仍不能使用EntityGraph。


[DATAJPA-35]: https://jira.spring.io/browse/DATAJPA-35
[DATAJPA-278]: https://jira.spring.io/browse/DATAJPA-278
[org-hibernate-hql-ast-querytranslatorimpl-list-attention-firstresult-maxresults]: http://stackoverflow.com/questions/6258556/org-hibernate-hql-ast-querytranslatorimpl-list-attention-firstresult-maxresults