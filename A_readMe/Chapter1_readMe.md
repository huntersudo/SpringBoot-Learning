# 引入web模块，完成一个简单的RESTful API

## 结构

Spring Boot的基础结构共三个文件（具体路径根据用户生成项目时填写的Group所有差异）：  

- src/main/java下的程序入口：Chapter1Application  
- src/main/resources下的配置文件：application.properties  
- src/test/下的测试入口：Chapter1ApplicationTests  

生成的Chapter1Application和Chapter1ApplicationTests类都可以直接运行来启动当前创建的项目，由于目前该项目未配合任何数据访问或Web模块，程序会在加载完Spring之后结束运行。

## 引入Web模块  

当前的pom.xml内容如下，仅引入了两个模块：

- spring-boot-starter：核心模块，包括自动配置支持、日志和YAML
- spring-boot-starter-test：测试模块，包括JUnit、Hamcrest、Mockito
引入Web模块，需添加spring-boot-starter-web模块：  

## 编写HelloWorld服务  

创建package命名为com.didispace.web（根据实际情况修改）  
创建HelloController类，内容如下  
```java

@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }

}

```
启动主程序，打开浏览器访问http://localhost:8080/hello，可以看到页面输出Hello World  

## 编写单元测试用例

打开的src/test/下的测试入口Chapter1ApplicationTests类。下面编写一个简单的单元测试来模拟http请求，具体如下：  

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = MockServletContext.class)
@WebAppConfiguration
public class Chapter1ApplicationTests {

	private MockMvc mvc;

	@Before
	public void setUp() throws Exception {
		mvc = MockMvcBuilders.standaloneSetup(new HelloController()).build();
	}

	@Test
	public void getHello() throws Exception {
		mvc.perform(MockMvcRequestBuilders.get("/hello").accept(MediaType.APPLICATION_JSON))
				.andExpect(status().isOk())
				.andExpect(content().string(equalTo("Hello World")));
	}

}

```

使用MockServletContext来构建一个空的WebApplicationContext，这样我们创建的HelloController就可以在@Before函数中创建并传递到MockMvcBuilders.standaloneSetup（）函数中  

注意引入下面内容，让status、content、equalTo函数可用  

```java
import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
````
