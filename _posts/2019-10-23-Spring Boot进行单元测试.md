---
layout:     post
title:      Spring Boot单元测试
subtitle:   Spring Boot进行单元测试
date:       2019-10-23
author:     yyconstantine
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 单元测试
---

### Spring Boot进行单元测试

---

#### 准备工作

- 引入依赖

  ```xml
  <dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-test</artifactId>
  	<scope>test</scope>
  </dependency>
  ```

- 搭建一个Spring Boot项目环境

- 对类或方法创建单元测试类，通过```Ctrl+Shift+T```（win下）或```⇧⌘T ```（Mac下）

---

#### Service && Dao 单元测试

创建后的单元测试方法大概长这样👇，我们可以在这里验证dao的结果正确性/service执行逻辑的合理性，这里不再过多描述

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class AdminDaoTest {
    
    @Autowired
    private AdminDao dao;
    
    @Test
    public void users() {
        List<User> list = dao.userList();
        Assert.assertNotNull(list);
        System.out.println(list);
    }
    
}
```

---

#### Controller单元测试

在对service或dao测试的时候，由于其本身就注入到了Spring容器中，所以我们可以通过```@Autowired```的方式调用，但controller本身是以HTTP请求的方式调用，所以我们无法直接通过注入的方式实现其调用。在这里，我们使用MockMvc模拟对接口的请求。

- 一个正常的controller，其中包含GET/POST/PUT/DELETE请求

  ```java
  @RestController
  @Slf4j
  public class AdminController {
      
      private AdminService adminService;
  
      private RedisUtil redisUtil;
  
  	private PropertyConfig propertyConfig;
  
  	@Autowired
  	public void setAdminService(AdminService adminService) {
  		this.adminService = adminService;
  	}
  
  	@Autowired
  	public void setRedisUtil(RedisUtil redisUtil) {
  		this.redisUtil = redisUtil;
  	}
  
  	@Autowired
  	public void setPropertyConfig(PropertyConfig propertyConfig) {
  		this.propertyConfig = propertyConfig;
  	}
      
      @PutMapping("/pwd")
  	public Result editPwd(@NotNull(message = "用户id不能为空") String globalUserId,
  						  @NotNull(message = "原密码不能为空") String oldPwd,
  						  @NotNull(message = "新密码不能为空") String newPwd) {
  		return adminService.editPwd(globalUserId, MD5Util.getMD5(oldPwd), MD5Util.getMD5(newPwd)) ?
  				Result.success(SuccessEnum.GLOBAL_EDIT_SUCCESS) :
  				Result.error(ErrorEnum.GLOBAL_EDIT_ERROR);
  	}
      
      @DeleteMapping("/quit/{globalUserId}")
  	public Result quit(@NotNull(message = "用户id不能为空") @PathVariable String globalUserId) {
  		// 删除redis对应session
  		redisUtil.del(globalUserId);
  		return Result.success(SuccessEnum.GLOBAL_QUIT_SUCCESS);
  	}
      
      @GetMapping("/resource/{userId}")
  	public Result<List<TreeVO>> resourceList(@NotNull(message = "用户id不能为空") @PathVariable String userId) {
  	    return Result.success(SuccessEnum.GLOBAL_SEARCH_SUCCESS, adminService.resourceList(userId));
  	}
      
      @PostMapping("/user")
  	public Result addUser(@Validated UserDTO userDTO) {
  		return adminService.addUser(userDTO) ?
                  Result.success(SuccessEnum.GLOBAL_ADD_SUCCESS) :
                  Result.error(ErrorEnum.GLOBAL_ADD_ERROR);
  	}
      
  }
  ```

- controller对应的单元测试类：

  ```java
  @SpringBootTest
  @RunWith(SpringRunner.class)
  @Slf4j
  public class AdminControllerTest {
      @Autowired
      private WebApplicationContext context;
  
      private MockMvc mockMvc;
      private MockCookie cookie;
  
      @Before
      public void setupMockMvc() {
          // 初始化MockMvc,用于模拟发送请求
          mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
          // 前置cookie使请求通过拦截器
          cookie = new MockCookie("globalUserId", "12345");
      }
      
      @Test
      public void resourceListByAdmin() throws Exception {
          mockMvc.perform(MockMvcRequestBuilders.get("/resource/12345")
                                                .cookie(cookie))
                  .andExpect(MockMvcResultMatchers.status().isOk())
                  .andDo(MockMvcResultHandlers.print());
      }
          
      @Test
      public void editPwd() throws Exception {
          String json = "{\"globalUserId\":\"12345\",\"oldPwd\":\"12345\",\"newPwd\":\"678910\"}";
          mockMvc.perform(MockMvcRequestBuilders.put("/pwd")
                                                .cookie(cookie))
                  .andExpect(MockMvcResultMatchers.status().isOk())
                  .andDo(MockMvcResultHandlers.print());
      }
      
      @Test
      public void quit() throws Exception {
          mockMvc.perform(MockMvcRequestBuilders.delete("/quit/12345")
                  .cookie(cookie))
                  .andExpect(MockMvcResultMatchers.status().isOk())
                  .andDo(MockMvcResultHandlers.print());
      }
      
  }
  ```

- 由于POST方法存在多种提交方式，所以我们单拎出来：

  - **application/x-www-form-urlencoded，俗称form表单提交**

  - ```java
    @Test
    public void login() throws Exception {
        // 组成提交表单的结构
        UrlEncodedFormEntity form = new UrlEncodedFormEntity(Arrays.asList(
                new BasicNameValuePair("userName", "admin"),
                new BasicNameValuePair("password", "123456")
        ), "utf-8");
        // 发送请求
        mockMvc.perform(MockMvcRequestBuilders.post("/login")
                .cookie(cookie)
                .content(EntityUtils.toString(form))// 添加表单元素
                .contentType(MediaType.APPLICATION_FORM_URLENCODED))
                .andExpect(MockMvcResultMatchers.status().isOk())// 预期返回结果
                .andDo(MockMvcResultHandlers.print());// 输出返回结果
    }
    ```

  - **application/json，也就是我们用```@RequestBody```接收的参数**

  - ```java
    @Test
    public void login() throws Exception {
        // 采用组装实体的方式转化json
        UserInfo userInfo = new UserInfo();
        userInfo.setUserName("admin");
        userInfo.setPassword("123456");
        String json = JSONObject.toJSONString(userInfo);
        // 或者你可以直接拼接json串
        String json = "{\"userName\":\"admin\",\"password\":\"123456\"}";
        // 发送请求
        mockMvc.perform(MockMvcRequestBuilders.post("/login")
                .cookie(cookie)
                .content(json)
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())// 预期返回结果
                .andDo(MockMvcResultHandlers.print());// 输出返回结果
    }
    ```

  - **multipart/form-data，表单方式提交流数据**，参照Stack Overflow

  - ```java
    MockMultipartFile file = new MockMultipartFile("data", "dummy.csv",
            "text/plain", "Some dataset...".getBytes());
    MockMultipartHttpServletRequestBuilder builder =
            MockMvcRequestBuilders.fileUpload("/test1/datasets/set1");
    builder.with(new RequestPostProcessor() {
        @Override
        public MockHttpServletRequest postProcessRequest(MockHttpServletRequest request) {
            request.setMethod("PUT");
            return request;
        }
    });
    mvc.perform(builder
            .file(file))
            .andExpect(status().ok());
    ```

---

#### 一个单元测试的技巧

在做单元测试的时候，由于数据都是开发人员随意指定的，会造成“垃圾数据”，我们就可以在测试类或方法上添加```@Transactional```注解，测试完即数据回滚

```java
@Test
@Transactional
public void editPwd() throws Exception {
    String json = "{\"globalUserId\":\"12345\",\"oldPwd\":\"12345\",\"newPwd\":\"678910\"}";
    mockMvc.perform(MockMvcRequestBuilders.put("/pwd")
                                          .cookie(cookie))
            .andExpect(MockMvcResultMatchers.status().isOk())
            .andDo(MockMvcResultHandlers.print());
}
```

---

#### 给自己埋坑

- 断言的使用（Assert/AssertThat）
- jupiter的使用