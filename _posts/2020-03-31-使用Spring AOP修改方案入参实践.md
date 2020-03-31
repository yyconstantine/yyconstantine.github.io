\---

layout:   post

title:   使用Spring AOP修改方案入参实践

subtitle:  使用Spring AOP修改方案入参实践

date:    2020-03-31

author:   yyconstantine

header-img: img/post-bg-universe.jpg

catalog: true

tags:

  \- Spring

\---

### 使用Spring AOP修改方案入参实践

---

业务需求是这样的：运营后台的新增/修改操作需要记录操作人，项目的session处理是全局持有一个userId来保持会话信息。这样会造成一个问题：想获取处理人信息时，需要通过userId获取到用户信息，再返回用户信息中的name作为操作人名称。每次都这样取很啰嗦（不太geek），所以预设优化方案如下：

- 由于用户信息（尤其是姓名）是基本不会变的信息，所以userId作为key，userName作为value在Redis存储
- 每次读取先从redis读取，读取不到再从数据库读取

上面的方案解决了每次用时查库的问题（实际运营后台操作量也不大，可以容忍= =），但每次需要手动获取，比较麻烦，于是想到通过aop的方式填充：

- 首先定义一个基类：

  ```java
  @Data
  public class BaseOperatorModel {
      
      private String createBy;
      
      private String updateBy;    
      
  }
  ```

- 然后任意接口参数继承：

  ```java
  @Data
  @EqualsAndHashCode(callSuper = true)
  public class ProductDTO extends BaseOperatorModel {
      
      private String productName;
      
      private Integer productType;
      
  }
  ```

- 再定义一个切面类，这里我们选择切所有方法：

  ```java
  @Aspect
  public class OperatorReplaceAspect {
      @Pointcut("execution(public * me.sxl.*.controller.*.*(..))")
      public void paramReplace() {
      }
      
      // 只有使用ProceedingJoinPoint，才能通过proceed(Object[])方法改变入参的值
      @Around("paramReplace()")
      public Object paramReplace(ProceedingJoinPoint pjp) throws Throwable {
          // 首先获取request，目的是获取session信息
          ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
          HttpServletRequest request = Objects.requireNonNull(attributes).getRequest();
  
          // 定义一个jackson序列化方式
          ObjectMapper mapper = new ObjectMapper();
          // 获取方法参数
          Object[] args = pjp.getArgs();
          // 定义接收参数的容器
          Map<String, Object> map = null;
          // 定义接收参数类型的class
          Class<?> clazz = null;
          // 定义获取args位置信息的index
          int index = -1;
          // for循环是针对存在多参数的情况
          for (int i = 0; i < args.length; i++) {
              // 由于参数存在继承关系,所以遍历到我们需要的参数时,这里返回true
              if (args[i] instanceof BaseOperatorModel) {
                  // 赋值class信息
                  clazz = args[i].getClass();
                  // 将该参数转换为map类型，方便处理
                  map = mapper.convertValue(args[i], Map.class);
                  // 获取index信息
                  index = i;
                  // 这里封装了通过userId获取userName的方法
                  String userName =
                          CommonUtils.getUserName((String) request.getSession().getAttribute(Constants.GLOBAL_USER_ID_KEY));
                  // 遍历map中的所有参数,找出名字为createBy/updateBy的参数，并对其赋值
                  for (Map.Entry entry : map.entrySet()) {
                      if (entry.getKey().equals(Constants.CREATE_BY_KEY)) {
                          map.put(Constants.CREATE_BY_KEY, userName);
                      }
                      if (entry.getKey().equals(Constants.UPDATE_BY_KEY)) {
                          map.put(Constants.UPDATE_BY_KEY, userName);
                      }
                  }
              }
          }
          // **下面很重要,如果不进行赋值操作及调用proceed方法,则入参改变不会生效**
          if (index >= 0) {
              args[index] = mapper.convertValue(map, clazz);
          }
          return pjp.proceed(args);
      }
  }
  ```

- over