---
layout: post
title:  "Hbase基本操作方法"
date:   2018-05-10 17:19:00
categories: DevOps
tags: hbase java
---

* content
{:toc}

hbase-client二次封装，核心解决hbase中列与实体对象对映射，简化对对象对处理

![](https://hbase.apache.org/images/hbase_logo_with_orca_large.png)




## 包依赖
```java
<!-- https://mvnrepository.com/artifact/org.apache.hbase/hbase-client -->
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>2.1.0</version>
</dependency>
```

## Template类实现
```java
package info.data.themis.component.hbase;

import java.io.IOException;
import java.math.BigInteger;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.NamespaceDescriptor;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.Admin;
import org.apache.hadoop.hbase.client.BufferedMutator;
import org.apache.hadoop.hbase.client.BufferedMutatorParams;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;
import org.apache.hadoop.hbase.client.Delete;
import org.apache.hadoop.hbase.client.Get;
import org.apache.hadoop.hbase.client.Mutation;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.client.ResultScanner;
import org.apache.hadoop.hbase.client.Scan;
import org.apache.hadoop.hbase.client.Table;
import org.apache.hadoop.hbase.io.compress.Compression;
import org.apache.hadoop.hbase.io.encoding.DataBlockEncoding;
import org.apache.hadoop.hbase.util.Bytes;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.util.Assert;

public class HbaseTemplate {

	private static final Logger LOGGER = LoggerFactory.getLogger(HbaseTemplate.class);

	private Configuration configuration;

	private volatile Connection connection;

	private Admin admin;

	private Boolean createTable;

	private Boolean disable;

	private static final String SPLIT_CHAR = ":";

	public HbaseTemplate(Configuration configuration, Boolean disable, Boolean createTable) {
		this.configuration = configuration;
		this.disable = disable;
		this.createTable = createTable;
		if (!disable) {
			this.getConnection();
		}
		Assert.notNull(configuration, " a valid configuration is required");
	}

	public <T> T execute(String tableName, TableCallback<T> action) {
		Assert.notNull(action, "Callback object must not be null");
		Assert.notNull(tableName, "No table specified");
		try (Table table = this.connection.getTable(TableName.valueOf(tableName))) {
			return action.doInTable(table);
		} catch (IOException e) {
			throw new HbaseSystemException(e);
		}
	}

	public <T> List<T> find(String tableName, String family, final RowMapper<T> action) {
		Scan scan = new Scan();
		scan.setCaching(5000);
		scan.addFamily(Bytes.toBytes(family));
		return this.find(tableName, scan, action);
	}

	public <T> List<T> find(String tableName, String family, String qualifier, final RowMapper<T> action) {
		Scan scan = new Scan();
		scan.setCaching(5000);
		scan.addColumn(Bytes.toBytes(family), Bytes.toBytes(qualifier));
		return this.find(tableName, scan, action);
	}

	public <T> List<T> find(String tableName, final Scan scan, final RowMapper<T> action) {
		return this.execute(tableName, new TableCallback<List<T>>() {

			public List<T> doInTable(Table table) throws HbaseSystemException {
				int caching = scan.getCaching();
				// 如果caching未设置(默认是1)，将默认配置成5000
				if (caching == 1) {
					scan.setCaching(5000);
				}
				try (ResultScanner scanner = table.getScanner(scan)) {
					List<T> rs = new ArrayList<>();
					for (Result result : scanner) {
						rs.add(action.result2object(result));
					}
					return rs;
				} catch (IOException | IllegalAccessException | InstantiationException e) {
					throw new HbaseSystemException(e);
				}
			}
		});
	}

	public <T> T get(String tableName, String rowName, final RowMapper<T> mapper) {
		return this.get(tableName, rowName, null, null, mapper);
	}

	public <T> T get(String tableName, String rowName, String familyName, final RowMapper<T> mapper) {
		return this.get(tableName, rowName, familyName, null, mapper);
	}

	public <T> T get(String tableName, final String rowName, final String familyName, final String qualifier,
			final RowMapper<T> mapper) {
		return this.execute(tableName, new TableCallback<T>() {
			public T doInTable(Table table) throws HbaseSystemException {
				Get get = new Get(Bytes.toBytes(rowName));
				if (StringUtils.isNotBlank(familyName)) {
					byte[] family = Bytes.toBytes(familyName);
					if (StringUtils.isNotBlank(qualifier)) {
						get.addColumn(family, Bytes.toBytes(qualifier));
					} else {
						get.addFamily(family);
					}
				}
				Result result;
				try {
					result = table.get(get);
					return mapper.result2object(result);
				} catch (IOException | IllegalAccessException | InstantiationException e) {
					throw new HbaseSystemException(e);
				}
			}
		});
	}

	public void execute(String tableName, MutatorCallback action) {
		Assert.notNull(action, "Callback object must not be null");
		Assert.notNull(tableName, "No table specified");
		long bufferSize = 3 * Long.valueOf(1024) * 1024;
		BufferedMutatorParams mutatorParams = new BufferedMutatorParams(TableName.valueOf(tableName));
		try (BufferedMutator mutator = this.connection.getBufferedMutator(mutatorParams.writeBufferSize(bufferSize))) {
			action.doInMutator(mutator);
		} catch (IOException e) {
			throw new HbaseSystemException(e);
		}
	}

	public void saveOrUpdate(String tableName, final Mutation mutation) {
		this.execute(tableName, new MutatorCallback() {
			public void doInMutator(BufferedMutator mutator) throws HbaseSystemException {
				try {
					mutator.mutate(mutation);
				} catch (IOException e) {
					throw new HbaseSystemException(e);
				}
			}
		});
	}

	public void saveOrUpdates(String tableName, final List<Mutation> mutations) {
		this.execute(tableName, new MutatorCallback() {
			public void doInMutator(BufferedMutator mutator) throws HbaseSystemException {
				try {
					mutator.mutate(mutations);
				} catch (IOException e) {
					throw new HbaseSystemException(e);
				}
			}
		});
	}

	private HTableDescriptor getTableDesc(String fullTableName, String family) {
		TableName tname = TableName.valueOf(fullTableName);
		HTableDescriptor table = new HTableDescriptor(tname);
		HColumnDescriptor column = new HColumnDescriptor(family);
		column.setDataBlockEncoding(DataBlockEncoding.NONE);// 编码
		column.setCompressionType(Compression.Algorithm.SNAPPY);// 压缩
		column.setBlocksize(32 * 1024);// 默认64M，scan为主适当增大，get为主适当减小
		table.addFamily(column);
		return table;
	}

	public boolean createTable(String tableName, String family) throws IOException {
		return createTable(tableName, family, null, null, 0);
	}

	public boolean createTable(String fullTableName, String family, String startKey, String endKey, int numRegions)
			throws IOException {
		if (fullTableName.indexOf(SPLIT_CHAR) > -1) {
			String[] items = fullTableName.split(SPLIT_CHAR);
			if (items.length == 2 && !this.isExistsNamespace(items[0])) {
				this.createNamespace(items[0]);
			}
		}
		HTableDescriptor table = getTableDesc(fullTableName, family);
		if (getAdmin().tableExists(table.getTableName())) {
			return false;
		}
		if (numRegions == 0) {
			getAdmin().createTable(table);
		} else {
			getAdmin().createTable(table, getHexSplits(startKey, endKey, numRegions));
		}
		return true;
	}

	/**
	 * 判断命名空间是否存在
	 * 
	 * @param strNamespace
	 *            命名空间
	 * @return true-存在,false-不存在
	 * @throws IOException
	 */
	public boolean isExistsNamespace(String strNamespace) throws IOException {
		NamespaceDescriptor[] namespaces = getAdmin().listNamespaceDescriptors();
		for (int i = 0; i < namespaces.length; i++) {
			if (strNamespace.equals(namespaces[i].getName())) {
				return true;
			}
		}
		return false;
	}

	/**
	 * 创建命名空间
	 * 
	 * @param strNamespace
	 *            命名空间
	 * @return true-创建成功,false-存在该namespace
	 * @throws IOException
	 */
	public boolean createNamespace(String strNamespace) throws IOException {
		if (isExistsNamespace(strNamespace)) {
			return false;
		} else {
			getAdmin().createNamespace(NamespaceDescriptor.create(strNamespace).build());
			return true;
		}
	}

	/**
	 * 判断表是否存在
	 * 
	 * @param strTableName
	 *            表名
	 * @return true-存在,false-不存在
	 * @throws IOException
	 */
	public boolean isExistsTable(String strTableName) throws IOException {
		TableName tableName = TableName.valueOf(strTableName);
		return getAdmin().tableExists(tableName);
	}

	/**
	 * 根据rowkey删除行
	 * 
	 * @param strTableName
	 *            表名
	 * @param rowkey
	 *            行名
	 * @throws IOException
	 */
	public void deleteByRowKey(String strTableName, String rowkey) throws IOException {
		Table table = connection.getTable(TableName.valueOf(strTableName));
		Delete delete = new Delete(Bytes.toBytes(rowkey));
		table.delete(delete);
	}

	/**
	 * 删除行
	 * 
	 * @param strTableName
	 *            表名
	 * @param list
	 *            删除数据集合
	 * @throws IOException
	 */
	public void deleteRow(String strTableName, List<Delete> list) throws IOException {
		Table table = connection.getTable(TableName.valueOf(strTableName));
		table.delete(list);
	}

	/**
	 * 截断表
	 * 
	 * @param strTableName
	 *            表名
	 * @throws IOException
	 */
	public void truncateTable(String strTableName) throws IOException {
		TableName tableName = TableName.valueOf(strTableName);
		getAdmin().disableTable(tableName);
		getAdmin().truncateTable(tableName, true);
	}

	/**
	 * 删除表
	 * 
	 * @param strTableName
	 *            表名
	 * @throws IOException
	 */
	public void deleteTable(String strTableName) throws IOException {
		TableName tableName = TableName.valueOf(strTableName);
		getAdmin().disableTable(tableName);
		getAdmin().deleteTable(tableName);
	}

	private byte[][] getHexSplits(String startKey, String endKey, int numRegions) {
		byte[][] splits = new byte[numRegions - 1][];
		BigInteger lowestKey = new BigInteger(startKey, 16);
		BigInteger highestKey = new BigInteger(endKey, 16);
		BigInteger range = highestKey.subtract(lowestKey);
		BigInteger regionIncrement = range.divide(BigInteger.valueOf(numRegions));
		lowestKey = lowestKey.add(regionIncrement);
		for (int i = 0; i < numRegions - 1; i++) {
			BigInteger key = lowestKey.add(regionIncrement.multiply(BigInteger.valueOf(i)));
			byte[] b = String.format("%016x", key).getBytes();
			splits[i] = b;
		}
		return splits;
	}

	public Connection getConnection() {
		if (null == this.connection) {
			synchronized (this) {
				if (null == this.connection) {
					try {
						ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(200, Integer.MAX_VALUE, 60L,
								TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
						poolExecutor.prestartCoreThread();// INIT pool
						this.connection = ConnectionFactory.createConnection(configuration, poolExecutor);
						this.admin = this.connection.getAdmin();
					} catch (Exception e) {
						LOGGER.error("hbase connection资源池创建失败", e);
					}
				}
			}
		}
		return this.connection;
	}

	public Admin getAdmin() {
		return this.admin;
	}

	public Boolean getCreateTable() {
		return createTable;
	}

	public Boolean getDisable() {
		return disable;
	}
}
```

## 默认mapper实现

```java
package info.data.themis.mapper;

import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;

import org.apache.hadoop.hbase.client.Mutation;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.util.Bytes;

import info.data.common.json.JsonWrap;
import info.data.themis.component.hbase.RowMapper;

public class DefaultMapper<T> implements RowMapper<T> {
	private String columnFamily;
	private Map<String, Field> colums = new HashMap<>();
	private Class<T> clazz;

	public DefaultMapper(Class<T> clazz) {
		super();
		this.clazz = clazz;
		map();
	}

	@Override
	public T result2object(Result result) throws IllegalAccessException, InstantiationException, IOException {
		if (result.size() == 0) {
			return null;
		}
		T obj = clazz.newInstance();
		for (Entry<String, Field> entry : colums.entrySet()) {
			Field f = entry.getValue();
			byte[] val = result.getValue(Bytes.toBytes(columnFamily), Bytes.toBytes(entry.getKey()));
			if (val == null) {
				continue;
			}
			switch (f.getName()) {
			case "Integer":
				f.setInt(obj, Bytes.toInt(val));
				break;
			case "Long":
				f.setLong(obj, Bytes.toLong(val));
				break;
			case "Boolean":
				f.setBoolean(obj, Bytes.toBoolean(val));
				break;
			case "String":
				f.set(obj, Bytes.toString(val));
				break;
			default:
				f.set(obj, JsonWrap.readAsObject(Bytes.toString(val), f.getType()));
				break;
			}
		}
		return obj;
	}

	@Override
	public List<Mutation> object2mutation(T t) throws IllegalAccessException, IOException {
		List<Mutation> list = new ArrayList<>();
		Field rowkeyField = colums.get("rowkey");
		String rowkey = rowkeyField.get(t).toString();
		for (Entry<String, Field> entry : colums.entrySet()) {
			Put put = new Put(Bytes.toBytes(rowkey));
			String conlumn = entry.getKey();
			Object val = entry.getValue().get(t);
			if (val != null) {
				put.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(conlumn),
						Bytes.toBytes(JsonWrap.writeAsString(val)));
				list.add(put);
			}
		}
		return list;
	}

	/**
	 * 获取所有字段 字段名称与hbase一一对应，存在一个final+static字段为列簇字段名称
	 * 
	 * @throws IllegalAccessException
	 * @throws IllegalArgumentException
	 */
	private void map() {
		Field[] fs = clazz.getDeclaredFields();// 得到类中的所有属性集合
		for (int i = 0; i < fs.length; i++) {
			Field f = fs[i];
			f.setAccessible(true); // 设置些属性是可以访问的
			int m = f.getModifiers();
			if (!Modifier.isFinal(m) && !Modifier.isStatic(m)) {
				colums.put(f.getName(), f);
			} else if ("FAMILY".equals(f.getName())) {
				try {
					columnFamily = f.get(clazz).toString();
				} catch (IllegalArgumentException | IllegalAccessException e) {
					columnFamily = "f0";
				}
			}
		}
	}

}
```

> 核心等两个类,其他接口自行创建
