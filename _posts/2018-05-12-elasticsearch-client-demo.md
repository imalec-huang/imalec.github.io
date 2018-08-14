---
layout: post
title:  "elasticsearch不同客户端连接实现"
date:   2018-05-12 16:19:00
categories: DevOps
tags: elasticsearch java
---

* content
{:toc}

elasticsearch官方提供了三种java-client实现方式，总体对比下来transport功能最为强大，主要还是解决了DSL语法组装对繁琐，提供了诸多builder实现，rest实现基本对http方式对api操作，high-client一种基于rest请求方式，transport参数方式，官方推荐

![](https://www.elastic.co/assets/blt244a845f141977c3/elastic-logo.svg)





## 包依赖
```java
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>transport</artifactId>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>elasticsearch-rest-high-level-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.elasticsearch.client</groupId>
			<artifactId>elasticsearch-rest-client-sniffer</artifactId>
		</dependency>
```

## Template
```
package info.data.elasticsearch;

import java.io.Closeable;
import java.io.IOException;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.commons.lang3.StringUtils;
import org.apache.http.HttpHost;
import org.elasticsearch.client.Response;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.transport.client.PreBuiltTransportClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import info.data.common.exception.BusinessException;
import info.data.elasticsearch.response.ResponseReader;

public abstract class EsTemplateAdmin implements Closeable {
	protected static final Logger logger = LoggerFactory.getLogger(EsTemplateAdmin.class);

	/**
	 * 健康状态
	 * 
	 * @return
	 */
	public ResponseReader health() throws BusinessException {
		try {
			Response r = restClient.performRequest("GET", "_cluster/health", Collections.<String, String>emptyMap());
			return new ResponseReader(r);
		} catch (IOException e) {
			throw new BusinessException("索引监控检测失败", e);
		}
	}

	/**
	 * 判断索引是否存在
	 */
	public boolean isExists(String index) throws BusinessException {
		try {
			Response response = restClient.performRequest("HEAD", index, Collections.<String, String>emptyMap());
			return response.getStatusLine().getStatusCode() == 200;
		} catch (IOException e) {
			throw new BusinessException("索引存在判断失败", e);
		}
	}

	/**
	 * 删除索引
	 */
	public boolean removeIndex(String index) throws BusinessException {
		try {
			Response response = restClient.performRequest("DELETE", index, Collections.<String, String>emptyMap());
			return response.getStatusLine().getStatusCode() == 200;
		} catch (IOException e) {
			throw new BusinessException("索引删除失败", e);
		}
	}

	/**
	 * 判断索引主分片存储大小
	 */
	public Double size(String index) throws BusinessException {
		Map<String, String> params = new HashMap<>();
		params.put("filter_path", "_all.primaries.store.size_in_bytes");
		try {
			Response r = restClient.performRequest("GET", StringUtils.join(index, "/_stats"), params);
			if (r.getStatusLine().getStatusCode() == 200) {
				ResponseReader rr = new ResponseReader(r);
				String line = rr.responseAsString();
				String pattern = "\\d+";
				Pattern pc = Pattern.compile(pattern);
				Matcher m = pc.matcher(line);
				if (m.find()) {
					return Double.valueOf(m.group());
				}
				return 0.0d;
			}
			return 0.0d;
		} catch (IOException e) {
			throw new BusinessException("索引大小获取失败", e);
		}
	}

	/**
	 * 添加一个别名
	 * 
	 * @param index
	 * @param alias
	 * @return
	 * @throws Exception
	 */
	public ResponseReader addAlias(String index, String alias) throws BusinessException {
		try {
			Response r = restClient.performRequest("PUT", StringUtils.join(index, "/_alias/", alias),
					Collections.<String, String>emptyMap());
			return new ResponseReader(r);
		} catch (IOException e) {
			throw new BusinessException("索引别名获取失败", e);
		}
	}

	/**
	 * 移除所有别名
	 * 
	 * @param index
	 * @return
	 * @throws Exception
	 */
	public ResponseReader removeAlias(String index) throws BusinessException {
		return removeAlias(index, "*");
	}

	/**
	 * 移除指定别名
	 * 
	 * @param index
	 * @return
	 * @throws Exception
	 */
	public ResponseReader removeAlias(String index, String alias) throws BusinessException {
		try {
			Response r = restClient.performRequest("DELETE", StringUtils.join(index, "/_alias/", alias),
					Collections.<String, String>emptyMap());
			return new ResponseReader(r);
		} catch (IOException e) {
			throw new BusinessException("索引别名移除失败", e);
		}
	}

	/**
	 * 获取所有别名
	 * 
	 * @param index
	 * @return
	 * @throws Exception
	 */
	public ResponseReader alias(String index) throws BusinessException {
		try {
			Response r = restClient.performRequest("GET", StringUtils.join(index, "/_alias"),
					Collections.<String, String>emptyMap());
			return new ResponseReader(r);
		} catch (IOException e) {
			throw new BusinessException("索引别名获取失败", e);
		}
	}

	/**
	 * 创建索引
	 * 
	 * @param index
	 * @return
	 * @throws BusinessException
	 */
	public ResponseReader create(String index) throws BusinessException {
		try {
			Response r = restClient.performRequest("PUT", index, Collections.<String, String>emptyMap());
			return new ResponseReader(r);
		} catch (Exception e) {
			throw new BusinessException("创建索引出错", e);
		}
	}

	protected TransportClient transportClient;
	protected RestClient restClient;
	protected RestHighLevelClient restHighLevelClient;
	protected EsProperties prop;

	public void init() {
		String[] hosts = prop.getHosts().split(",");
		Settings settings = Settings.builder().put("cluster.name", prop.getClusterName())
				.put("client.transport.sniff", false).build();
		transportClient = new PreBuiltTransportClient(settings);
		HttpHost[] httpHosts = new HttpHost[hosts.length];
		for (int i = 0; i < hosts.length; i++) {
			httpHosts[i] = new HttpHost(hosts[i], 9200, "http");
			try {
				transportClient
						.addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(hosts[i]), 9300));
			} catch (UnknownHostException e) {
				logger.error("地址解析错误", e);
			}
		}
		restClient = RestClient.builder(httpHosts).build();
		restHighLevelClient = new RestHighLevelClient(restClient);
	}

	@Override
	public void close() throws IOException {
		restClient.close();
		transportClient.close();
	}

	protected static final char[] INVALID = new char[] { '"', '*', '\\', '<', '|', ',', '>', '/', '?' };
}
```