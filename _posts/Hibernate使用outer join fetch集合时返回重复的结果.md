title: Hibernate使用outer join fetch集合时返回重复的结果
date: 2016-05-21 17:00:37
tags:
- Hibernate
- 经验错误
---

## 问题

如果我们有一个订单关联三个订单项，当进行下面的查询时

```
List result = session.createQuery("select o from Order o left join fetch o.lineItems").list();  
```

返回的结果集会是三个一样的订单对象，分别包含了三个订单项。

<!-- more -->

## 原因

首先在SQL层面LEFT JOIN会以左表为驱动表去关联从表的中的所有记录。以上面的的订单为例，就会返回三条纪录。而Hibernate默认会保留所有的驱动表中的记录，多余的两条记录会引用同一条订单记录。

## 解决

```
List result = session.createQuery("select o from Order o left join fetch o.lineItems")  
                      .setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY) // Yes, really!  
                      .list();  
```

或者

```
List result = session.createQuery("select distinct o from Order o left join fetch o.lineItems").list();  
```

后者的意义和SQL的distinct并不一样，这里它就是DISTINCT_ROOT_ENTITY result transformer的缩写。

https://developer.jboss.org/wiki/HibernateFAQ-AdvancedProblems#jive_content_id_Hibernate_does_not_return_distinct_results_for_a_query_with_outer_join_fetching_enabled_for_a_collection_even_if_I_use_the_distinct_keyword