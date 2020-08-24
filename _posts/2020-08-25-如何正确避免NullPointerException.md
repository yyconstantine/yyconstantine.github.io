---

layout:   post

title:   如何正确避免NullPointerException

subtitle:  NullPointerException | 基本功修炼

date:    2020-08-25

author:   yyconstantine

header-img: img/post-bg-universe.jpg

catalog: true

tags:

  - 苦练基本功

---

### 如何正确避免NullPointerException

---

我们常说的代码健壮性，很多时候就是对各种异常情况的处理，比如数据的格式、数据的类型、非法输入、空指针等。前几种异常我们可以简单地通过JSR-303（Valid、Validation）来进行处理，而空指针是我们最常见、也最容易忽略的问题。作者也在实际开发中写出过很多NPE，痛定思痛决定水一篇博客出来记录一下。。

首先，空指针的场景我认为大致可以分为两种

- 对象的空指针
- 集合的空指针

---

#### 一个基本认知

首先，对于我们后台开发的同学来说，一个最基本的认知就是，任何来源的数据我们都不能认为是可信的。即，我们获取到一个对象，马上想到它可能是一个null而非一个new出来的对象；我们获取到一个集合，马上想到它可能是一个null或者一个emptyList()。

---

#### 对象的空指针

对象的空指针很好理解，我们在开发中会有很多的model，我们在Spring项目中会存在各种各样的bean，我们无法保证我们获取到的属性一定不为空，所以最基本的场景，我们会这样写：

```java
if (obj == null) 
    return;
```

一般来说，作为一个前置的判断条件，我们这样处理就足够了，但是仍然有一些特殊场景，可能我们需要用到一个非空的属性进行某些操作（允许其属性为空），则我们可以这样写：

```java
Object nonNullObj = Optional.ofNullable(obj).orElse(new Object());
```

这样我们可以正常地使用包装过的obj而不必担心NPE问题

即使如此，也还是存在NPE的情况，假设我们定义一个model如下：

```java
@Data
public class Model {
    private Integer number;
}
```

在我们基于上述场景使用obj的时候，进行这样的操作：

```java
if (nonNullObj.getNumber().equals(100)) {
    // do something
}
```

由于我们的number属性没有初始化，则会抛出NPE。个人觉得有两种避免方式：

- 使用equals进行比较，尽量将已知的量放在括号外进行比较：

  ```java
  if (100.equals(nonNullObj.getNumber())) {
      // do something
  }
  ```

- 或者使用Optional的特性（看起来有些啰嗦）：

  ```java
  if (Optional.ofNullable(obj)
      .filter(Objects::nonNull)
      .filter(o -> o.getNumber() != null && o.getNumber().equals(100))
      .isPresent()) {
      // do something
  }
  ```

  

---

#### 集合的空指针

当你避开了对象的空指针，恭喜你过了一关，但是集合的空指针更是让人防不胜防～（踩坑无数）

如最开始所说，我们仍然对一个传递过来的集合进行非空判断：

```java
if (CollectionUtils.isEmpty(list)) 
    return;
if (MapUtils.isEmpty(map)) 
    return;
```

一个值得注意的点是，除了进行非空判断，我们应当尽量自己定义一个集合去接受数据，而不是直接操作获取的数据集合：

```java
ArrayList<Integer> myList = dataSource.list();
```

这样做的原因是：我们接收到的集合类，可能其并不是一个完整的List/Map。

为什么这么说呢？我们翻看`ArrayList`的源码知道，其继承了一个`AbstractList`，这个抽象List已经实现了一部分方法，但是如add方法并没有实现，而是抛出了一个`UnsupportedOperationException`，Map也同理。

另外，如果我们只是简单地循环一个接收的集合，可能并不需要再单独定义一个变量接收，则我们也可以这样写：

```java
Optional.ofNullable(list).orElse(new ArrayList())
    .forEach(System.out::println);
```

此外，我们考虑到，可能集合中存在部分元素为空，则我们对上述代码进行增强：

```java
Optional.ofNullable(list).orElse(new ArrayList())
    .stream()
    .filter(Objects::nonNull)
    .forEach(System.out::println);
```

另外，我们可能不单单是进行一些简单的循环操作，在stream操作中我们经常会使用map进行模型转换，这里同样存在一个坑：

```java
Map<Integer, String> map = Maps.newHashMap();
List<Integer> list = Lists.newArrayList(1, 2, 3);
List<String> newList = list.stream()
    .filter(Objects::nonNull)
    .map(item -> map.get(item))
    .collect(Collectors.toList());
```

我们可以看到，map只进行了初始化，实际没有赋值，所以我们每次循环的map.get(item)都取到null值，但是collect会将这些null值作为集合的元素全部填充到集合内，这时我们再使用这个集合就会出现问题，尽管我们进行了常规的判空操作：

```java
if (CollectionUtils.isEmpty(newList)) {
    return;
}

System.out.print(newList.get(0));
```

在我们使用get(0)时，发生NPE，原因是我们的newList集合元素为：[null, null, null]

所以，我们对进行了模型转换操作的后，需要再次进行非空过滤：

```java
List<String> newList = list.stream()
    .filter(Objects::nonNull)
    .map(item -> map.get(item))
    .filter(Objects::nonNull)
    .collect(Collectors.toList());
```



