---
layout: post
title:  "电商检索"
date:   2015-07-10 19:20:36
categories: ecommerce
---

# 电商检索

## 需求：

* 支持商品全文搜索
* 支持排序
* 多字段查询(品牌，名称，分类)
* 相关性

## 可选方案

最原始外包使用的是mysql like方案，查询效果肯定不佳。

因为现在是基于java开发，所以基本都基于lucene，常用的是solr和elasticsearch。

大致查看了网络的评论和对比，最终选择了elasticsearch。

过程中不断尝试优化检索效果，效果还是比较满意的。

## 技术细节

### schema

商品基础信息：
不同的属性有不同的类型，方便排序与比较。

```       
XContentBuilder mapping = jsonBuilder().startObject().startObject("sku").startObject("properties")        
        .startObject("marketPrice").field("type", "double").endObject()
        .startObject("categoryId").field("type", "long").endObject()
        .startObject("blockId").field("type", "long").endObject()
        .startObject("cityId").field("type", "long").endObject()
        .startObject("brandName").field("type", "string").field("analyzer", "ik").endObject()
        .startObject("name").field("type", "string").field("analyzer", "ik").endObject()
        .endObject()
        .endObject()
        .endObject();
```

### 自定义字典检索

作为电商，必然会有相当多的商品，品牌。这些关键词难以被准确分词，这种情况下，需要我们自建词典。

ik/custom/mydict.dic

```
生抽
老抽
海天
东古
李锦记
鸡精
鸡粉
...
```

### 同义词检索

* 考虑以下的情形：

	有商品“海天生抽”，“海天老抽”，“海天精品酱油”。
	
	用户搜索“酱油”，缺省情况下，肯定只能搜索到“海天精品酱油”。尽管生抽，老抽属于酱油，但这种情况下，就无法被检索到。

* 解决方案引入同义词

**定义同义词analyzer**

```
index:
  analysis:
    analyzer:
      ik:
          alias: [ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider
      ik_max_word:
          type: ik
          use_smart: false
      ik_smart:
          type: ik
          use_smart: true
      synonym:
          type: custom
          tokenizer: ik_tok
          filter: [synonym]
    tokenizer:
      ik_tok:
        type: ik
        use_smart: true
    filter:
      synonym:
        type: synonym
        synonyms_path: analysis/synonym.txt
```
**analysis/synonym.txt**

```
酱油=>酱油,生抽,老抽
辣椒=>辣椒,泡椒,剁椒
...
```

**检索时采用 synonym analyzer**

```
    final MultiMatchQueryBuilder queryBuilder = QueryBuilders.multiMatchQuery(productSearchCriteria.getQuery(), "name", "brandName");
    queryBuilder.operator(MatchQueryBuilder.Operator.AND);
    queryBuilder.analyzer("synonym");
    builder.setQuery(queryBuilder);

```

### O2O检索

因为商品检索带有一定的地区搜索的属性，每个商品可能只覆盖周边一定的区域，只展示在部分城市的部分区域中(上面代码中的blockId)。

在构建商品索引时，将商品的区块信息也一起构建。

```
    document.put("name", product.getName());
    document.put("description", product.getDescription());
    
    document.put("blockId", blockIds);

    ...
    return document;
```

这样在查询时通过用户的地理区块信息，就可以获取相应的商品。

```
    final SearchRequestBuilder builder = client.prepareSearch(indexAlias).setTypes("sku");

    if (StringUtils.isNotBlank(productSearchCriteria.getQuery())) {
        final MultiMatchQueryBuilder queryBuilder = QueryBuilders.multiMatchQuery(productSearchCriteria.getQuery(), "name", "brandName");
        queryBuilder.operator(MatchQueryBuilder.Operator.AND);
        queryBuilder.analyzer("synonym");
        builder.setQuery(queryBuilder);
    }

    List<FilterBuilder> filters = new ArrayList<FilterBuilder>();
    
    final Long blockId = productSearchCriteria.getBlockId();
    if (blockId != null && blockId != 0l) {
        filters.add(FilterBuilders.inFilter("blockId", new long[]{blockId.longValue()}));
    }


    builder.addFields("id", "name", "categoryId", "cityId", "blockId", "brandId", "brandName", "marketPrice");

    builder.setPostFilter(FilterBuilders.andFilter(filters.toArray(new FilterBuilder[filters.size()])));
```


### 索引切换

一般在商品变更时会同时更新索引，但有时会有一些后台操作导致数据与索引不匹配，这种情况下需要手动重建索引。

如果没有索引无缝切换，那么在重建索引的过程中，将会无法进行检索。

elasticsearch通过别名，提供了索引无缝切换的方案。

```
	final String newIndex = "product_v_" + System.currentTimeMillis();
	client.admin().indices().prepareCreate(newIndex).execute().actionGet();
	
	// rebuild new index
	// ...
	
	
	// find out all old index
	List<String> oldIndexes = new ArrayList<>();
	
	if (client.admin().indices().prepareAliasesExist(indexAlias).execute().actionGet().exists()) {
	    final UnmodifiableIterator<String> iterator = client.admin().indices().prepareGetAliases(indexAlias).execute().actionGet().getAliases().keysIt();
	
	    while (iterator.hasNext()) {
	        oldIndexes.add(iterator.next());
	    }
	}
	
	// swap alias
	final IndicesAliasesRequestBuilder indicesAliasesRequestBuilder = client.admin().indices().prepareAliases();
	if (!oldIndexes.isEmpty()) {
	    indicesAliasesRequestBuilder.removeAlias(oldIndexes.toArray(new String[oldIndexes.size()]), indexAlias);
	}
	indicesAliasesRequestBuilder.addAlias(newIndex, indexAlias);
	indicesAliasesRequestBuilder.execute().actionGet();
	
	// remove old indexes
	if (!oldIndexes.isEmpty()) {
	    client.admin().indices().prepareDelete(oldIndexes.toArray(new String[oldIndexes.size()])).execute().actionGet();
	}
	
	client.admin().indices().prepareRefresh().execute().actionGet();


```













        