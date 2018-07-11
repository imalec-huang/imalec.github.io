---
layout: post
title:  "Elasticsearch DSL使用说明"
date:   2018-06-14 17:19:00
categories: 服务中间件
tags: elasticsearch java
---

* content
{:toc}

elasticsearch dsl提供了强大复杂的查询功能，使用过程中如果长时间不接触，就会忘记，所以记录一篇基本的使用文章，便于查询回忆





## 总体结构
query
	bool
		must
		must_not
		should
		filter
match
term
range

## query与filter
query：文档与查询文档的匹配程度，会设计到文档的打分
filter：文档是否匹配