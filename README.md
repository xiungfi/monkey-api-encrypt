# 使用

## 前言

前后端分离的开发方式，我们以接口为标准来进行推动，定义好接口，各自开发自己的功能，最后进行联调整合。无论是开发原生的APP还是webapp还是PC端的软件,只要是前后端分离的模式，就避免不了调用后端提供的接口来进行业务交互。

网页或者app，只要抓下包就可以清楚的知道这个请求获取到的数据，这样的接口对爬虫工程师来说是一种福音，要抓你的数据简直轻而易举。

数据的安全性非常重要，特别是用户相关的信息，稍有不慎就会被不法分子盗用，所以我们对这块要非常重视，容不得马虎。

monkey-api-encrypt是对基于Servlet的Web框架API请求进行统一加解密操作的框架。

## 功能点

- 支持所有基于Servlet的Web框架（Spring Boot, Spring Cloud Zuul等框架）
- 内置AES加密算法
- 支持用户自定义加密算法
- 使用简单，有操作示列

## 使用文档

第一步：增加项目依赖

```xml
<dependency>
  <groupId>com.cxytiandi</groupId>
  <artifactId>monkey-api-encrypt-core</artifactId>
  <version>1.2.2.RELEASE</version>
</dependency>
```

第二步：配置加解密过滤器(Spring Boot中配置方式)

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<EncryptionFilter> filterRegistration() {
    	EncryptionConfig config = new EncryptionConfig();
    	config.setKey("d86d7bab3d6ac01ad9dc6a897652f2d2");//1.2版本及以下key 16位，1.2以上key 32位
    	config.setRequestDecyptUriList(Arrays.asList("/save", "/decryptEntityXml"));
    	config.setResponseEncryptUriList(Arrays.asList("/encryptStr", "/encryptEntity", "/save", "/encryptEntityXml", "/decryptEntityXml"));
        FilterRegistrationBean<EncryptionFilter> registration = new FilterRegistrationBean<EncryptionFilter>();
        registration.setFilter(new EncryptionFilter(config));
        registration.addUrlPatterns("/*");
        registration.setName("EncryptionFilter");
        registration.setOrder(1);
        return registration;
    }

}
```

- EncryptionConfig EncryptionConfig是加解密的配置类，配置项目定义如下：

```java
public class EncryptionConfig {

	/**
	 * AES加密Key，长度必须16（1.2版本及以下key 16位，1.2以上key 32位）
	 */
	private String key = "d86d7bab3d6ac01ad9dc6a897652f2d2;
	
	/**
	 * 需要对响应内容进行加密的接口URI<br>
	 * 比如：/user/list<br>
	 * 不支持@PathVariable格式的URI
	 */
	private List<String> responseEncryptUriList = new ArrayList<String>();
	
	/**
	 * 需要对请求内容进行解密的接口URI<br>
	 * 比如：/user/list<br>
	 * 不支持@PathVariable格式的URI
	 */
	private List<String> requestDecyptUriList = new ArrayList<String>();

	/**
	 * 响应数据编码
	 */
	private String responseCharset = "UTF-8";
	
	/**
	 * 开启调试模式，调试模式下不进行加解密操作，用于像Swagger这种在线API测试场景
	 */
	private boolean debug = false;
}
```

## 原理文档

1.0版本用的`RequestBodyAdvice` 和 `ResponseBodyAdvice` 来对请求和响应内容进行加解密操作，后面考虑到通用性，决定基于Servlet底层来做处理。

1.1版本就是基于Servlet来实现的。

用的是`HttpServletRequestWrapper`和`HttpServletResponseWrapper`来实现的。

`HttpServletRequestWrapper`使用场景比较广泛，比如说通过`HttpServletRequestWrapper`可以重新session的实现逻辑，将session存入数据库或者redis。

只要能够获取到请求和响应的内容，剩下的就简单了，加解密而已。

核心代码在encrypt-core里的`com.cxytiandi.encrypt.core`包下，感兴趣的同学可以自己去看下。



## 常见问题

### 1. Spring Cloud Zuul中如何使用？

使用方式和Spring Boot中一样，没区别。

### 2. 如果需要所有请求都做加解密处理怎么办？

默认不配置RequestDecyptUriList和ResponseEncryptUriList的情况下，就会对所有请求进行处理（拦截器指定范围内的请求）

### 3. Swagger测试接口的时候怎么处理？

可以开启调试模式，就不对请求做加解密处理，通过配置debug=true

### 4. RequestDecyptUriList和ResponseEncryptUriList能否支持/user/*模式匹配？

过滤器本身就有这个功能了，所以框架中是完全匹配相等才可以，可以通过过滤器的 registration.addUrlPatterns("/user/*","/order/*");来指定需要处理的接口地址。

### 5. 默认开启全部加解密功能，如果想要忽略某些接口怎么办？

配置方式可以使用下面的方式进行忽略：

```properties
spring.encrypt.requestDecyptUriIgnoreList[0]=/save
spring.encrypt.responseEncryptUriIgnoreList[0]=/encryptEntity
spring.encrypt.responseEncryptUriIgnoreList[1]=/save
```



注解可以使用@DecryptIgnore和@EncryptIgnore进行忽略

## 自定义加密算法

内置了AES加密算法对数据进行加解密操作，同时用户可以自定义算法来代替内置的算法。

自定义算法需要实现EncryptAlgorithm接口：

```java
/**
 * 自定义RSA算法
 * 
 * @author yinjihuan
 * 
 * @date 2019-01-12
 * 
 * @about http://cxytiandi.com/about
 *
 */
public class RsaEncryptAlgorithm implements EncryptAlgorithm {

	public String encrypt(String content, String encryptKey) throws Exception {
		return RSAUtils.encryptByPublicKey(content);
	}

	public String decrypt(String encryptStr, String decryptKey) throws Exception {
		return RSAUtils.decryptByPrivateKey(encryptStr);
	}

}
```



注册Filter的时候指定算法：

```java
EncryptionConfig config = new EncryptionConfig();
registration.setFilter(new EncryptionFilter(config, new RsaEncryptAlgorithm()));
```

## 详细配置使用

### 手动注册过滤器使用

```
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<EncryptionFilter> filterRegistration() {
    	EncryptionConfig config = new EncryptionConfig();
    	config.setKey("abcdef0123456789");
    	config.setRequestDecyptUriList(Arrays.asList("/save", "/decryptEntityXml"));
    	config.setResponseEncryptUriList(Arrays.asList("/encryptStr", "/encryptEntity", "/save", "/encryptEntityXml", "/decryptEntityXml"));
        FilterRegistrationBean<EncryptionFilter> registration = new FilterRegistrationBean<EncryptionFilter>();
        registration.setFilter(new EncryptionFilter(config));
        registration.addUrlPatterns("/*");
        registration.setName("EncryptionFilter");
        registration.setOrder(1);
        return registration;
    }

}
```



### Spring Boot Starter方式使用

启动类加@EnableEncrypt注解，开启加解密自动配置，省略了手动注册Filter的步骤

```
@EnableEncrypt
@SpringBootApplication
public class App {

	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}
	
}
```



配置文件中配置加密的信息，也就是EncryptionConfig

```
spring.encrypt.key=abcdef0123456789
spring.encrypt.requestDecyptUriList[0]=/save
spring.encrypt.requestDecyptUriList[1]=/decryptEntityXml

spring.encrypt.responseEncryptUriList[0]=/encryptStr
spring.encrypt.responseEncryptUriList[1]=/encryptEntity
spring.encrypt.responseEncryptUriList[2]=/save
spring.encrypt.responseEncryptUriList[3]=/encryptEntityXml
spring.encrypt.responseEncryptUriList[4]=/decryptEntityXml
```



如果感觉配置比较繁琐，你的加解密接口很多，需要大量的配置，还可以采用另一种方式来标识加解密，就是注解的方式。

响应的数据需要加密，就在接口的方法上加@Encrypt注解

```
@Encrypt
@GetMapping("/encryptEntity")
public UserDto encryptEntity() {
	UserDto dto = new UserDto();
	dto.setId(1);
	dto.setName("加密实体对象");
	return dto;
}
```



接收的数据需要解密，就在接口的方法上加@Decrypt注解

```
@Decrypt
@PostMapping("/save")
public UserDto save(@RequestBody UserDto dto) {
	System.err.println(dto.getId() + "\t" + dto.getName());
	return dto;
}
```



同时需要加解密那么两个注解都加上即可

```
@Encrypt
@Decrypt
@PostMapping("/save")
public UserDto save(@RequestBody UserDto dto) {
	System.err.println(dto.getId() + "\t" + dto.getName());
	return dto;
}
```



### Spring MVC中使用

Spring MVC中可以直接在web.xml中注册Filter，不方便传递的是配置的参数，我们可以配置一个自定的过滤器，然后在这个过滤器中配置EncryptionFilter

```
public class ApiEncryptionFilter implements Filter {

	EncryptionFilter filter = null;
	
	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		EncryptionConfig config = new EncryptionConfig();
		config.setKey("abcdef0123456789");
		config.setRequestDecyptUriList(Arrays.asList("/save"));
		config.setResponseEncryptUriList(Arrays.asList("/encryptEntity"));
		filter = new EncryptionFilter(config);
	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		filter.doFilter(request, response, chain);
	}

	@Override
	public void destroy() {
		
	}

}
```



web.xml

```
<filter>
        <description>自定义加解密过滤器</description>  
        <filter-name>ApiEncryptionFilter</filter-name>
        <filter-class>com.cxytiandi.mvc.filter.ApiEncryptionFilter</filter-class>
</filter>

<filter-mapping>
        <filter-name>ApiEncryptionFilter</filter-name>
        <url-pattern>/*</url-pattern>
 </filter-mapping>
```



如果需要使用注解的话需要在spring的xml中配置ApiEncryptDataInit

```
<bean id="apiEncryptDataInit" class="com.cxytiandi.encrypt.springboot.init.ApiEncryptDataInit"></bean>
```



### 注意事项

要么使用手动注册Filter的方式开启加解密功能，手动构造EncryptionConfig传入EncryptionFilter中，要么使用@EnableEncrypt开启加解密功能。

@EnableEncrypt+配置文件可以在Spring Boot,Spring Cloud Zuul中使用

@EnableEncrypt+@Encrypt+@Decrypt可以在Spring Boot，Spring MVC中使用

**相同URI问题**

当存在两个相同的URI时，比如GET请求的/user和POST的请求/user。如果只想对其中某一个进行处理，我们的逻辑的是按照URI进行匹配，这样就会影响到另一个，原因是URI是一样的。

如果是使用@Encrypt+@Decrypt的方式，框架会自动处理，会为每一个URI加上前缀来区分不同的请求方式。同时提供了扩展的属性值，在@Encrypt+@Decrypt中都有value属性，可以手动配置uri。因为某些框架不是用的Spring MVC的注解，比如CXF，框架无法做到适配所有的注解，这个时候可以用uri属性来配置。

配置格式为：请求类型+访问的URI

```
get:/user

post:/user
```



包括在配置文件中也可以采用前缀的方式来区分相同的URI。



## 注意

spring-boot-starter-encrypt是最开始的1.0版本，基于Spring MVC机制实现的，像Zuul中就使用不了，代码留着可以给大家参考下。

示列：https://github.com/yinjihuan/spring-boot-starter-encrypt-example

原理讲解：http://cxytiandi.com/blog/detail/20235

## 文章
- [1.1.1版本发布](https://mp.weixin.qq.com/s/3HoHnzsdPIvNgjkLr9GSIw)
- [1.1.2版本发布](https://mp.weixin.qq.com/s/JJFxbkb9HtVMECByvMJ4Gg)
- [1.2版本发布](https://mp.weixin.qq.com/s/TbTr44Hc9gkJB40L9eHHYQ)


# 作者
- 尹吉欢 1304489315@qq.com
- 博客 http://cxytiandi.com/blogs/yinjihuan
