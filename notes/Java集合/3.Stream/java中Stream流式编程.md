# java中Stream的流式编程

## 1.Stream常用操作

```java
stream接口中定义了很多操作,大致可以分为两大类,一类是中间操作,另一类是终端操作

(1)中间操作
中间操作会返回另外一个流,多个中间操作可以链接起来形成一个查询
中间操作有惰性,如果流上没有一个终端操作,那么中间操作是不会做任何处理的

常用的中间操作:

1.map
map是将输入流中每一个元素映射为另一个元素形成输出流
// 初始化一个不可变字符串
List<String> words = ImmutableList.of("hello", "java8", "stream");
// 计算列表中每个单词的长度
List<Integer> list = words.stream()
        .map(String::length)
        .collect(Collectors.toList());
// output: 5 5 6
list.forEach(System.out::println);

2.flatMap
List<String[]> list1 = words.stream()
        .map(word -> word.split("-"))
        .collect(Collectors.toList());
        
// output: [Ljava.lang.String;@59f95c5d, 
//             [Ljava.lang.String;@5ccd43c2
list1.forEach(System.out::println);

你预期是List, 返回却是List<String[]>, 这是因为split方法返回的是String[]
这个时候你可以想到要将数组转成stream, 于是有了第二个版本
Stream<Stream<String>> arrStream = words.stream()
        .map(word -> word.split("-"))
        .map(Arrays::stream);
        
// output: java.util.stream.ReferencePipeline$Head@2c13da15, 
// java.util.stream.ReferencePipeline$Head@77556fd
arrStream.forEach(System.out::println);

3.filter
filter接收predicate对象,按条件过滤,符合条件的元素生成另一个流
// 过滤出单词长度大于5的单词，并打印出来
List<String> words = ImmutableList.of("hello", "java8", "hello", "stream");
words.stream()
        .filter(word -> word.length() > 5)
        .collect(Collectors.toList())
        .forEach(System.out::println);
// output: stream

(2)终端操作
终端操作将stream流转成具体的返回值,比如 List, Integer等,常见的终端操作还有: foreach,max,min,count等
// 找出最大的值
List<Integer> integers = Arrays.asList(6, 20, 19);
integers.stream()
        .max(Integer::compareTo)
        .ifPresent(System.out::println);
// output: 20
```

## 2.实战,使用stream重构代码

需求:过滤年龄大于20岁,并且分数大于95分的学生

```java
private List<Student> getStudents() {
    Student s1 = new Student("xiaoli", 18, 95);
    Student s2 = new Student("xiaoming", 21, 100);
    Student s3 = new Student("xiaohua", 19, 98);
    List<Student> studentList = Lists.newArrayList();
    studentList.add(s1);
    studentList.add(s2);
    studentList.add(s3);
    return studentList;
}
```

1.使用for循环

```java
public void refactorBefore() {
    List<Student> studentList = getStudents();
    // 使用临时list
    List<Student> resultList = Lists.newArrayList();
    for (Student s : studentList) {
        if (s.getAge() > 20 && s.getScore() > 95) {
            resultList.add(s);
        }
    }
    // output: Student{name=xiaoming, age=21, score=100}
    resultList.forEach(System.out::println);
}
```

2.使用lambda和stream重构

```java
public void refactorAfter() {
    List<Student> studentLists = getStudents();
        // output: Student{name=xiaoming, age=21, score=100}
        studentLists.stream()
                .filter(student->student.getAge() > 20 && student.getScore() > 95)
                .forEach(System.out::println);
}
```

## 3.Collection系列->创建流

java8中,扩展了 Collection 接口,添加了两个默认方法来创建流

```java
public interface Collection<E> extends Iterable<E> {
	...省略...
    default Stream<E> stream() {...}
    default Stream<E> parallelStream() {...}
}
```

`stream()` 是转化为顺序流，`paralletlStream()` 转换为并行流。

```java
public class ListStream {
    public static void main(String[] args) {
        String[] months = {"Jan", "Feb", "Mar", "Apr", "May",
                "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec", "Jun", "Jun", "Feb", "Jan"};
        // 集合创建
        List<String> list = Arrays.asList(months);
        list.stream().forEach(s -> System.out.format("%s ", s));
    }
}
```

输出结果

```java
Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec 
```

将 `List` 对象的 `stream()` 方法变换为 `parallelStream()` 方法后

输出结果

```java
Aug Jul Jun Feb Dec Oct Nov Apr Mar May Jan Sep
```

`stream()` 与 `parallelStream()` 的区别:

```java
stream() 是转化为顺序流，它使用主线程，是单线程的；parallelStream() 是转化为并行流，它是多个线程同时运行的。因此，stream() 是按顺序输出的，而parallelStream() 不是
```

如何将 `Map` 关系映射表转化为流呢?

```java
public class MapStream {
    public static void main(String[] args) {
        Map<String, Integer> users = new HashMap<>();
        users.put("小赵", 18);
        users.put("小钱", 29);
        users.put("小孙", 20);
        users.put("小李", 29);

        users.entrySet().stream().forEach(entry -> System.out.format("%s ", entry));
        System.out.println();
        users.keySet().stream().forEach(key -> System.out.format("%s ", key));
        System.out.println();
        users.values().stream().forEach(value -> System.out.format("%d ", value));
    }
}

```

输出结果

```java
小孙=20 小李=29 小钱=29 小赵=18 
小孙 小李 小钱 小赵 
20 29 29 18 
```

```java
从上面的例子就可得知，我们得到 Map 的 Entity 集合，Key 的集合和Value的集合，在将它们转化为流去处理。
```