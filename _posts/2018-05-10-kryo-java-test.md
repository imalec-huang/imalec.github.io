---
layout: post
title:  "kryo高性能序列化组件"
date:   2018-05-10 16:19:00
categories: 应用组件
tags: java
---

* content
{:toc}

此前项目中需要用到对象序列化，对比了下jackson/kryo/jdk Serializ，总体表现json方式能在性能与稳定性上达到一个均衡
kryo性能极好，不过对于多层嵌套对对象表现很不稳定，java序列化稳定性比较好不过性能比较糟糕





## 包依赖
```java
			<dependency>
				<groupId>com.esotericsoftware</groupId>
				<artifactId>kryo</artifactId>
				<version>4.0.2</version>
			</dependency>
			<dependency>
```
## 测试
```java
package info.data.themis.core.function;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.util.HashMap;
import java.util.Map;

import org.junit.Test;
import org.objenesis.strategy.StdInstantiatorStrategy;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.KryoException;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;

public class KryoTest {
	@Test
	public void test() {

	}

	public void testxx() {
		try {
			long start = System.currentTimeMillis();
			setSerializableObjectJava();
			System.out.println("Java 序列化时间:" + (System.currentTimeMillis() - start) + " ms");
			start = System.currentTimeMillis();
			getSerializableObjectJava();
			System.out.println("Java 反序列化时间:" + (System.currentTimeMillis() - start) + " ms");
			start = System.currentTimeMillis();
			setSerializableObject();
			System.out.println("Kryo 序列化时间:" + (System.currentTimeMillis() - start) + " ms");
			start = System.currentTimeMillis();
			getSerializableObject();
			System.out.println("Kryo 反序列化时间:" + (System.currentTimeMillis() - start) + " ms");
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}

	}

	public static void setSerializableObject() throws FileNotFoundException {

		Kryo kryo = new Kryo();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(false);
		kryo.setInstantiatorStrategy(new StdInstantiatorStrategy());
		kryo.register(Simple.class);
		Output output = new Output(new FileOutputStream("/tmp/kro.bin"));
		for (int i = 0; i < 100000; i++) {
			Map<String, Integer> map = new HashMap<String, Integer>(2);
			map.put("zhang0", i);
			map.put("zhang1", i);
			kryo.writeObject(output, new Simple("zhang" + i, (i + 1), map));
		}
		output.flush();
		output.close();
	}

	public static void getSerializableObject() {
		Kryo kryo = new Kryo();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(false);
		kryo.setInstantiatorStrategy(new StdInstantiatorStrategy());
		Input input;
		try {
			input = new Input(new FileInputStream("/tmp/kro.bin"));
			@SuppressWarnings("unused")
			Simple simple = null;
			while ((simple = kryo.readObject(input, Simple.class)) != null) {
				// System.out.println(simple.getAge() + " " + simple.getName() + " " +
				// simple.getMap().toString());
			}

			input.close();
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		} catch (KryoException e) {

		}
	}

	public static void setSerializableObjectJava() throws IOException {

		FileOutputStream fo = new FileOutputStream("/tmp/java.bin");

		ObjectOutputStream so = new ObjectOutputStream(fo);

		for (int i = 0; i < 100000; i++) {
			Map<String, Integer> map = new HashMap<String, Integer>(2);
			map.put("zhang0", i);
			map.put("zhang1", i);
			so.writeObject(new Simple("zhang" + i, (i + 1), map));
		}
		so.flush();
		so.close();
	}

	public static void getSerializableObjectJava() {
		FileInputStream fi;
		try {
			fi = new FileInputStream("/tmp/java.bin");
			ObjectInputStream si = new ObjectInputStream(fi);

			@SuppressWarnings("unused")
			Simple simple = null;
			while ((simple = (Simple) si.readObject()) != null) {
				// System.out.println(simple.getAge() + " " + simple.getName());
			}
			fi.close();
			si.close();
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		} catch (IOException e) {
			// e.printStackTrace();
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}

	}
}

class Simple implements Serializable {
	private static final long serialVersionUID = -4914434736682797743L;
	private String name;
	private int age;
	private Map<String, Integer> map;

	public Simple() {

	}

	public Simple(String name, int age, Map<String, Integer> map) {
		this.name = name;
		this.age = age;
		this.map = map;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public Map<String, Integer> getMap() {
		return map;
	}

	public void setMap(Map<String, Integer> map) {
		this.map = map;
	}

}
```

## 工具类

```
package info.data.themis.component.mq.serializer;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.Serializable;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.codec.binary.Base64;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;
import com.esotericsoftware.kryo.pool.KryoFactory;
import com.esotericsoftware.kryo.pool.KryoPool;
import com.esotericsoftware.kryo.serializers.CollectionSerializer;
import com.esotericsoftware.kryo.serializers.JavaSerializer;
import com.esotericsoftware.kryo.serializers.MapSerializer;

public class KryoSerializerUtils {

	private static JavaSerializer javaSerializer = new JavaSerializer();
	private static KryoPool pool;

	static {
		KryoFactory factory = () -> new Kryo();
		pool = new KryoPool.Builder(factory).build();
	}

	public static <T extends Serializable> String serializationObject(T obj) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);

		ByteArrayOutputStream baos = new ByteArrayOutputStream();
		Output output = new Output(baos);
		kryo.writeClassAndObject(output, obj);
		output.flush();
		output.close();

		byte[] b = baos.toByteArray();
		try {
			baos.flush();
			baos.close();
		} catch (IOException e) {
			e.printStackTrace();
		}

		return new String(new Base64().encode(b));
	}

	@SuppressWarnings("unchecked")
	public static <T extends Serializable> T deserializationObject(String obj, Class<T> clazz) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);

		ByteArrayInputStream bais = new ByteArrayInputStream(new Base64().decode(obj));
		Input input = new Input(bais);
		return (T) kryo.readClassAndObject(input);
	}

	public static <T extends Serializable> String serializationList(List<T> obj, Class<T> clazz) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(true);

		CollectionSerializer serializer = new CollectionSerializer();
		serializer.setElementClass(clazz, javaSerializer);
		serializer.setElementsCanBeNull(false);

		kryo.register(clazz, javaSerializer);
		kryo.register(ArrayList.class, serializer);

		ByteArrayOutputStream baos = new ByteArrayOutputStream();
		Output output = new Output(baos);
		kryo.writeObject(output, obj);
		output.flush();
		output.close();

		byte[] b = baos.toByteArray();
		try {
			baos.flush();
			baos.close();
		} catch (IOException e) {
			e.printStackTrace();
		}

		return new String(new Base64().encode(b));
	}

	@SuppressWarnings("unchecked")
	public static <T extends Serializable> List<T> deserializationList(String obj, Class<T> clazz) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(true);

		CollectionSerializer serializer = new CollectionSerializer();
		serializer.setElementClass(clazz, javaSerializer);
		serializer.setElementsCanBeNull(false);

		kryo.register(clazz, javaSerializer);
		kryo.register(ArrayList.class, serializer);

		ByteArrayInputStream bais = new ByteArrayInputStream(new Base64().decode(obj));
		Input input = new Input(bais);
		return (List<T>) kryo.readObject(input, ArrayList.class, serializer);
	}

	public static <T extends Serializable> String serializationMap(Map<String, T> obj, Class<T> clazz) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(true);

		MapSerializer serializer = new MapSerializer();
		serializer.setKeyClass(String.class, javaSerializer);
		serializer.setKeysCanBeNull(false);
		serializer.setValueClass(clazz, javaSerializer);
		serializer.setValuesCanBeNull(true);

		kryo.register(clazz, javaSerializer);
		kryo.register(HashMap.class, serializer);

		ByteArrayOutputStream baos = new ByteArrayOutputStream();
		Output output = new Output(baos);
		kryo.writeObject(output, obj);
		output.flush();
		output.close();

		byte[] b = baos.toByteArray();
		try {
			baos.flush();
			baos.close();
		} catch (IOException e) {
			e.printStackTrace();
		}

		return new String(new Base64().encode(b));
	}

	@SuppressWarnings("unchecked")
	public static <T extends Serializable> Map<String, T> deserializationMap(String obj, Class<T> clazz) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(true);

		MapSerializer serializer = new MapSerializer();
		serializer.setKeyClass(String.class, javaSerializer);
		serializer.setKeysCanBeNull(false);
		serializer.setValueClass(clazz, javaSerializer);
		serializer.setValuesCanBeNull(true);

		kryo.register(clazz, javaSerializer);
		kryo.register(HashMap.class, serializer);

		ByteArrayInputStream bais = new ByteArrayInputStream(new Base64().decode(obj));
		Input input = new Input(bais);
		return (Map<String, T>) kryo.readObject(input, HashMap.class, serializer);
	}

	public static <T extends Serializable> String serializationSet(Set<T> obj, Class<T> clazz) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(true);

		CollectionSerializer serializer = new CollectionSerializer();
		serializer.setElementClass(clazz, javaSerializer);
		serializer.setElementsCanBeNull(false);

		kryo.register(clazz, javaSerializer);
		kryo.register(HashSet.class, serializer);

		ByteArrayOutputStream baos = new ByteArrayOutputStream();
		Output output = new Output(baos);
		kryo.writeObject(output, obj);
		output.flush();
		output.close();

		byte[] b = baos.toByteArray();
		try {
			baos.flush();
			baos.close();
		} catch (IOException e) {
			e.printStackTrace();
		}

		return new String(new Base64().encode(b));
	}

	@SuppressWarnings("unchecked")
	public static <T extends Serializable> Set<T> deserializationSet(String obj, Class<T> clazz) {
		Kryo kryo = pool.borrow();
		kryo.setReferences(false);
		kryo.setRegistrationRequired(true);

		CollectionSerializer serializer = new CollectionSerializer();
		serializer.setElementClass(clazz, javaSerializer);
		serializer.setElementsCanBeNull(false);

		kryo.register(clazz, javaSerializer);
		kryo.register(HashSet.class, serializer);

		ByteArrayInputStream bais = new ByteArrayInputStream(new Base64().decode(obj));
		Input input = new Input(bais);
		return (Set<T>) kryo.readObject(input, HashSet.class, serializer);
	}
}
```
## Kryo rabbitmq MessageConverter实现例子

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.Serializable;
import java.io.UnsupportedEncodingException;
import java.util.ArrayList;

import org.objenesis.strategy.StdInstantiatorStrategy;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageProperties;
import org.springframework.amqp.support.converter.MessageConversionException;
import org.springframework.amqp.support.converter.WhiteListDeserializingMessageConverter;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;

import info.data.common.protocol.GeneralDto;
import info.data.common.protocol.LocationDto;
import info.data.common.protocol.MediaDto;
import info.data.common.protocol.OriginDto;

/**
 * KRYO序列化
 */
public class KryoSerializerMessageConverter extends WhiteListDeserializingMessageConverter {

	public static final String DEFAULT_CHARSET = "UTF-8";

	private volatile String defaultCharset = DEFAULT_CHARSET;

	private static Kryo kryo = new Kryo();

	public KryoSerializerMessageConverter() {
		kryo.setReferences(true);// 对象内部循环引用当时候，必须设置为true，默认true
		kryo.setRegistrationRequired(true);// 是否要求强制注册
		kryo.register(GeneralDto.class);
		kryo.register(ArrayList.class);
		kryo.register(LocationDto.class);
		kryo.register(MediaDto.class);
		kryo.register(OriginDto.class);
		((Kryo.DefaultInstantiatorStrategy) kryo.getInstantiatorStrategy())
				.setFallbackInstantiatorStrategy(new StdInstantiatorStrategy());// 很重要，估计是个bug，未设置无法创建arraylist类型对象反序列化
	}

	public byte[] serialization(Object obj) throws IOException {
		ByteArrayOutputStream baos = new ByteArrayOutputStream();
		Output output = new Output(baos);
		kryo.writeObject(output, obj);
		output.close();
		output.toBytes();
		byte[] b = baos.toByteArray();
		return b;
	}

	public <T extends Serializable> T deserialization(byte[] bytes, Class<T> clazz) {
		ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
		Input input = new Input(bais);
		T t = kryo.readObject(input, clazz);
		input.close();
		return t;
	}

	/**
	 * Converts from a AMQP Message to an Object.
	 */
	@Override
	public Object fromMessage(Message message) throws MessageConversionException {
		Object content = null;
		MessageProperties properties = message.getMessageProperties();
		if (properties != null) {
			String contentType = properties.getContentType();
			if (contentType != null && contentType.startsWith("text")) {
				String encoding = properties.getContentEncoding();
				if (encoding == null) {
					encoding = this.defaultCharset;
				}
				try {
					content = new String(message.getBody(), encoding);
				} catch (UnsupportedEncodingException e) {
					throw new MessageConversionException("failed to convert text-based Message content", e);
				}
			} else if (contentType != null && contentType.equals(MessageProperties.CONTENT_TYPE_SERIALIZED_OBJECT)) {
				content = this.deserialization(message.getBody(), GeneralDto.class);
			}
		}
		if (content == null) {
			content = message.getBody();
		}
		return content;
	}

	/**
	 * Creates an AMQP Message from the provided Object.
	 */
	@Override
	protected Message createMessage(Object object, MessageProperties messageProperties)
			throws MessageConversionException {
		byte[] bytes;
		if (object instanceof String) {
			try {
				bytes = ((String) object).getBytes(this.defaultCharset);
			} catch (UnsupportedEncodingException e) {
				throw new MessageConversionException("failed to convert Message content", e);
			}
			messageProperties.setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN);
			messageProperties.setContentEncoding(this.defaultCharset);
		} else if (object instanceof byte[]) {
			bytes = (byte[]) object;
			messageProperties.setContentType(MessageProperties.CONTENT_TYPE_BYTES);
		} else {
			try {
				bytes = this.serialization(object);
			} catch (IOException e) {
				throw new MessageConversionException("Cannot convert object to bytes", e);
			}
			messageProperties.setContentType(MessageProperties.CONTENT_TYPE_SERIALIZED_OBJECT);
		}
		if (bytes != null) {
			messageProperties.setContentLength(bytes.length);
		}
		return new Message(bytes, messageProperties);
	}

}
```

> kroy非常快，但是多层嵌套的复杂类型会出现数据错位
