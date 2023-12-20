# 1.概述
- 大数量下处理集合效率高
- 代码可读性强
- 消灭嵌套地狱
# 2.Lambda 表达式
## 核心原则
可推导可省略
## 基本格式
(参数列表)->{代码}
### 例一
创建线程并启动时可以使用匿名内部类的写法
```java
new Thread(new Runnable(){
	@Override
	public void run(){
		System.out.println("nihao");
	}
}).start();
```
可以使用 Lambda 的格式对其进行修改
```java
new Thread(()->
	System.out.println("nihao");
).start();
```
### 例二
现有一方法定义，使用 Lambda 表达式调用该类
```java
public static int calculateNum(IntBinaryOperator operator){  
    int a = 10;  
    int b = 20;  
    return operator.applyAsInt(a,b);  
}
```
Lambda 调用改函数写法
```java
calculateNum((left,right)->left+right);
```
### 例三
```java
public class LambdaDemo01 {  
    public static void main(String[] args) {  
        printNum(new IntPredicate() {  
            @Override  
            public boolean test(int value) {  
                return value%2==0;  
            }  
        });  
    }  
  
    public static void printNum(IntPredicate predicate){  
        int[] arr = {1,2,3,4,5,6,7,8,9,10};  
        for (int i : arr){  
            if(predicate.test(i)){  
                System.out.println(i);  
            }  
        }  
    }  
}
```
Lambda 改写
```java
public static void main(String[] args) {  
	printNum(value -> value%2==0);  
}  
```
### 例四
```java
public static void main(String[] args) {
    Integer resutl = typeConver(new Function<String, Integer>() {  
        @Override  
        public Integer apply(String s) {  
            return Integer.valueOf(s);  
        }  
    });  
    System.out.println(resutl);  
}  
public static <R> R typeConver(Function<String,R> function){  
    String str = "1234";  
    R result = function.apply(str);  
    return result;  
}
```
Lambda 写法
```java
Integer resutl = typeConver((String s)-> {  
    return Integer.valueOf(s);  
});  
System.out.println(resutl);
```
### 例五
```java
public static void main(String[] args) {  
  
    foreachArr(new IntConsumer() {  
        @Override  
        public void accept(int value) {  
            System.out.println(value);  
        }  
    });  
}  
  
public static void foreachArr(IntConsumer consumer){  
    int[] arr = {1,2,3,4,5,4,6,7,8,9,10};  
    for(int i : arr){  
        consumer.accept(i);  
    }  
}
```
Lambda 写法
```java
foreachArr((int value)-> {  
    System.out.println(value);  
});
```
## 省略规则
- 参数类型可以省略
- 方法体只有一句代码时大括号 return 和唯一一句代码的分号可省略
- 方法只有一个参数时，小括号可以省略
- 以上规则可省略
# 3.Stream 流
- Java 8 中的 Stream 使用的是函数式编程，他可以被用来对集合或数组进行链状流式的操作。可以更方便的让我们对集合或数组操作
## 案例数据准备
- 导入依赖
```xml
<dependency>  
    <groupId>org.projectlombok</groupId>  
    <artifactId>lombok</artifactId>  
    <version>1.18.24</version>  
</dependency>
```
```java
@Data  
@AllArgsConstructor  
@NoArgsConstructor  
@EqualsAndHashCode  //用于后期去重使用  
public class Author {  
    private Long id;  
    private String name;  
    private Integer age;  
    private String intro;  
    private List<Book> books;  
}
```
```java
@Data  
@AllArgsConstructor  
@NoArgsConstructor  
@EqualsAndHashCode  
public class Book {  
    private Long id;  
    private String name;  
  
    //分类->”哲学，小说“  
    private String category;  
      
    //评分  
    private Integer score;  
      
    //简介  
    private String intro;  
}
```
```java
private static List<Author> getAuthors(){  
    //数据初始化  
    Author author = new Author(1L,"蒙多",33,"一个从菜刀中明悟哲理的祖安人",null);  
    Author author2 = new Author(2L,"亚拉索",15,"狂风也追逐不上他的思考速度",null);  
    Author author3 = new Author(3L,"易",14,"是这个世界在限制他的思维",null);  
    Author author4 = new Author(3L,"易",14,"是这个世界在限制他的思维",null);  
    //书籍列表  
    List<Book>books1 = new ArrayList<>();  
    List<Book> books2 = new ArrayList<>();  
    List<Book> books3 = new ArrayList<>();  
    books1.add(new Book(1L,"刀的两侧是光明与黑暗","哲学,爱情",88,"用一把刀划分了爱恨"));  
    books1.add(new Book(2L,"一个人不能死在同一把刀下","个人成长,爱情",99,"讲述如何从失败中明悟真理"));  
  
    books2.add(new Book(3L,"那风吹不到的地方","哲学",85,"带你用思维去领略世界的尽头"));  
    books2.add(new Book(3L,"那风吹不到的地方","哲学",85,"带你用思维去领略世界的尽头"));  
    books2.add(new Book(4L,"吹或不吹","爱情,个人传记",56,"一个哲学家的恋爱观注定很难把他所在的时代理解"));  
  
    books3.add(new Book(5L,"你的剑就是我的剑","爱情",56,"无法想象一个武者能对他的伴侣这么的宽容"));  
    books3.add(new Book(6L,"风与剑","个人传记",100,"两个哲学家灵魂和肉体的碰撞会激起怎么样的火花呢"));  
    books3.add(new Book(6L,"风与剑","个人传记",100,"两个哲学家灵魂和肉体的碰撞会激起怎么样的火花呢"));  
  
    author.setBooks(books1);  
    author2.setBooks (books2);  
    author3.setBooks(books3);  
    author4.setBooks (books3);  
  
    List<Author> authorList = new ArrayList<>(Arrays.asList(author,author2,author3,author4));  
    return authorList;  
}
```
## 快速入门
- 需求
我们可以调用 getAuthors 方法获取到作家的集合，现在需要打印年龄小于 18 的作家的名字，并且要注意去重。
- 实现
```java
List<Author> authors = getAuthors();  
authors.stream()//把集合转换成流  
        .distinct()//去重  
        .filter(author->author.getAge()<18)//筛选年龄小于18的  
        .forEach(author->System.out.println(author.getName()));//遍历
```
## 常用操作
### 创建流
- 单列集合：`集合对象. Stream ()`
```java
List<Author> authors = getAuthors();  
Stream<Author> stream = authors.stream();
```
- 数组：`Arrays.stream(数组)或使用Stream.of来创建`
```java
Integer[] arr = {2,3,21,3,21,3,2,1};  
Stream<Integer> stream1 = Arrays.stream(arr);  
Stream<Integer> stream2 = Stream.of(arr);
```
- 双列集合：转换成单列集合再创建
```java
HashMap<String, Integer> map = new HashMap<>();  
map.put("蜡笔小新",19);  
map.put("黑子",17);  
map.put("日向下滑",16);  
  
Set<Map.Entry<String, Integer>> entrySet = map.entrySet();  
Stream<Map.Entry<String, Integer>> stream = entrySet.stream();
```
### 中间操作
- Filter
可以对流中的元素进行条件过滤，符合条件的才能继续留在流中。
```java
//列如：打印所有姓名长度大于1的作家姓名  
List<Author> authors = getAuthors();  
authors.stream()  
        .filter(author -> author.getName().length()>1)  
        .forEach(author -> System.out.println(author));
```
- Map
可以把流中的元素进行计算或转换
```java
//打印所有作家的姓名  
List<Author> authors = getAuthors();  
authors.stream()  
        .map(author -> author.getName())  
        .forEach(s -> System.out.println(s));
```
- Distinct
可以去除流中重复的元素
```java
//打印所有作家名称，并且要求其中不能有重复元素  
List<Author> authors = getAuthors();  
authors.stream()  
        .distinct()  
        .map(author -> author.getName())  
        .forEach(System.out::println);
```
注意：distinct 方法是依赖 Object 的 equals 方法来判断是否是相同对象的，所以需要注意重写 equals 方法。
- Sorted
可以对流中的元素进行排序
```java
//对流中元素按照年龄进行降序排序，并且要求不能有重复元素  
List<Author> authors = getAuthors();  
authors.stream()  
        .distinct()  
        .sorted((o1, o2) -> o2.getAge() - o1.getAge())  
        .forEach(author -> System.out.println(author.getAge()));
```
注意：如果调用空参的 sorted()方法，需要流中的元素实现了 Comparable
- Limit
可以设置流的最大长度，超出的部分将被抛弃
```java
//对流中的元素按照年龄进行降序排序，并且不能有重复元素，然后打印其中年龄最大的两个作家的姓名  
List<Author> authors = getAuthors();  
authors.stream()  
        .distinct()  
        .sorted((o1, o2) -> o2.getAge()-o1.getAge())  
        .limit(2)  
        .forEach(author -> System.out.println(author.getName()));
```
- Skip
跳过流中的前 n 个元素，返回剩下的元素
```java
//打印除了年龄最大的作家外的其他作家，要求不能有重复元素，并按照年龄降序排序  
List<Author> authors = getAuthors();  
authors.stream()  
        .distinct()  
        .sorted(((o1, o2) -> o2.getAge()-o1.getAge()))  
        .skip(1)  
        .forEach(author -> System.out.println(author.getName()));
```
- FlatMap
map 只能把一个对象转换成另一个对象来作为流的元素，而 flatMap 可以把一个对象转换为多个对象作为流的元素。
```java
//1.打印所有书籍的名字，要求对重复元素去重  
List<Author> authors = getAuthors();  
authors.stream()  
        .flatMap(author -> author.getBooks().stream())  
        .distinct()  
        .forEach(book -> System.out.println(book.getName()));
```
```java
//2. 打印现有数据的所有分类，要求对分类进行去重，不能出现这种格式：哲学，爱情  
List<Author> authors = getAuthors();  
authors.stream()  
        .flatMap(author -> author.getBooks().stream())  
        .distinct()  
        .flatMap(book -> Arrays.stream(book.getCategory().split(",")))  
        .distinct()  
        .forEach(System.out::println);
```
### 终结操作
- Foreach
```java
//输出所有作家的名字  
List<Author> authors = getAuthors();  
authors.stream()  
        .map(Author::getName)
        .distinct()
        .forEach(System.out::println);
```
- Count
可以获取当前流中元素的个数
```java
//打印这些作家的所出书籍的数目，注意删除重复元素  
List<Author> authors = getAuthors();  
long count = authors.stream()  
        .flatMap(author -> author.getBooks().stream())  
        .distinct()  
        .count();  
System.out.println(count);
```
- Max&min 
可以用来获取流中的最值
```java
//分别获取这些作家的所出书籍的最高分和最低分并打印  
//Stream<Author> -> Stream<Book> -> Stream<Integer>  
List<Author> authors = getAuthors();  
Optional<Integer> max = authors.stream()  
        .flatMap(author -> author.getBooks().stream())  
        .distinct()  
        .map(Book::getScore)  
        .max((score1, score2) -> score1 - score2);  
  
Optional<Integer> min = authors.stream()  
        .flatMap(author -> author.getBooks().stream())  
        .distinct()  
        .map(Book::getScore)  
        .min((score1, score2) -> score1 - score2);  
System.out.println(max.get());  
System.out.println(min.get());
```
- Collect
把当前流转换成一个集合
```java
//获取一个存放所有作者名字的List集合  
List<Author> authors = getAuthors();  
List<String> nameList = authors.stream()  
        .map(Author::getName)  
        .collect(Collectors.toList());  
System.out.println(nameList);;
```
```java
//获取一个所有书名的Set集合  
List<Author> authors = getAuthors();  
Set<String> nameSet = authors.stream()  
        .flatMap(author -> author.getBooks().stream())  
        .map(Book::getName)  
        .collect(Collectors.toSet());  
System.out.println(nameSet);
```
```java
//获取一个Map集合，map的key为作者名，value为List<Book>  
List<Author> authors = getAuthors();  
Map<String, List<Book>> map = authors.stream()  
        .distinct()  
        .collect(Collectors.toMap(author -> author.getName(), author -> author.getBooks()));  
System.out.println(map);
```
- AnyMatch
可以用来判断是否有任意符合匹配条件的元素
```java
//判断是否有年龄在29以上的作家  
List<Author> authors = getAuthors();  
boolean flag = authors.stream()  
        .anyMatch(author -> author.getAge() > 29);  
System.out.println(flag);
```
- AllMatch
可以用来判断是否都符合条件
```java
//判断是否所有作家都为成年人  
List<Author> authors = getAuthors();  
boolean flag = authors.stream()  
        .allMatch(author -> author.getAge() >= 18);  
System.out.println(flag);
```
- NoneMatch
可以判断流中的元素是否都不符合匹配条件
```java
//判断作家是否都没超过100岁  
List<Author> authors = getAuthors();  
boolean flag = authors.stream()  
        .noneMatch(author -> author.getAge() > 100);  
System.out.println(flag);
```
- FindAny
获取流中任意一个元素，该方法没有办法保证获取的一定是流中的第一个元素
```java
//获取任意一个大于18的作家，如果存在就输出他的名字  
List<Author> authors = getAuthors();  
Optional<Author> optionalAuthor = authors.stream()  
        .filter(author -> author.getAge()>18)  
        .findAny();  
optionalAuthor.ifPresent(author -> System.out.println(author.getName()));
```
- FindFirst
获取流中第一个元素
```java
//获取一个年龄最小的作家，并输出他的姓名  
List<Author> authors = getAuthors();  
Optional<Author> firstAuthor = authors.stream()  
        .sorted((o1, o2) -> o1.getAge() - o2.getAge())  
        .findFirst();  
firstAuthor.ifPresent(author -> System.out.println(author.getName()));
```
- Reduce 归并
对流中的数据按照你指定的计算方式计算出一个结果。（缩紧操作）
reduce 两个参数的重载形式其内部的计算方式如下：
```java
T result =identity;
for(T element : this stream)
	result = accumulator.apply(result,element)
return result;
```
例：
```java
//1. 使用reduce求所有作者年龄的和  
List<Author> authors = getAuthors();  
Integer result = authors.stream()  
        .distinct()  
        .map(Author::getAge)  
        .reduce(0, (result1, element) -> result1 + element);  
System.out.println(result);
```
```java
//2. 使用reduce求所有作者中年龄的最大值  
List<Author> authors = getAuthors();  
Integer max = authors.stream()  
        .map(Author::getAge)  
        .reduce(Integer.MIN_VALUE, (result, element) -> result < element ? element : result);  
System.out.println(max);
```
```java
//3.使用reduce求所有作者中年龄的最小值  
List<Author> authors = getAuthors();  
Integer min = authors.stream()  
        .map(Author::getAge)  
        .reduce(Integer.MAX_VALUE, (result, element) -> result > element ? element : result);  
System.out.println(min);
```
reduce 一个参数的重载形式其内部计算方式：
```java
boolean foundAny = false;
T result = null;
for (T element : this stream) {
	if (!foundAny) {          
		foundAny = true;
		result = element;
	}
	else
		result = accumulator.apply(result, element);  }  
return foundAny ? Optional.of(result) : Optional.empty();
```
## 注意事项
- 惰性求值（如果没有终结操作，中间操作不会得到执行）
- 流是一次性的
- 不会影响原数据
# 4.Optional
我们在编写代码时很多情况需要进行非空判断，判断过多导致代码臃肿，JDK 8 中引入了 Optional，养成使用 Optional 的习惯可以写出更优雅的代码来避免空指针异常
## 创建对象
```java
Author author = getAuthors();  
Optional<Author> optionalAuthor = Optional.ofNullable(author);
```
- 如果确定一个对象不为空，则可以使用 Optional 的静态方法 of 来把数据进行封装
```java
Author author = getAuthors();  
Optional<Author> optionalAuthor = Optional.of(author);
```
- 可以使用 Optional 的静态方法 empty 来将 null 封装成 Optional 对象
```java
Optional.empty();
```
## 安全消费值
我们获取到一个 Optional 对象后可以使用 ifPresent 方法来消费其中的值，这个方法会判断其内部封装的数据是否为空。
```java
Optional<Author> authorOptional = getAuthors();  
authorOptional.ifPresent(System.out::println);
```
## 安全获取值
如果我们期望安全的获取值，不推荐使用 get 方法，而是使用 Optional 提供的方法
- OrElseGet
获取数据并设置数据为空时的默认值，如果数据不为空则获取这个数据。
```java
Optional<Author> authorOptional = getAuthors();  
Author author = authorOptional.orElseGet(() -> new Author());  
System.out.println(author.getName());
```
- OrElseThrow
获取数据，如果数据不为空则获取该数据，为空则根据你传入的参数来创建异常抛出。
```java
Optional<Author> authorOptional = getAuthors();  
try {  
    authorOptional.orElseThrow((Supplier<Throwable>) () -> new RuntimeException("数据为null"));  
} catch (Throwable e) {  
    throw new RuntimeException(e);  
}
```
## 过滤
我们可以使用 filter 方法对数据进行过滤。如果原本有数据，但不符合判断，也会变成一个无数据的 Optional 对象。
```java
Optional<Author> authorOptional = getAuthors();  
authorOptional.filter(author -> author.getAge()>18).ifPresent(System.out::println);
```
## 判断
```java
Optional<Author> authorOptional = getAuthors();  
if (authorOptional.isPresent()) {  
    System.out.println(authorOptional.get().getName());  
}
```
## 数据转换
```java
Optional<Author> authorOptional = getAuthors();  
authorOptional.map(author -> author.getBooks())  
        .ifPresent(System.out::println);
```
# 5. 函数式接口
- 只有一个抽象方法的接口我们称之为函数式接口。
- JDK 中函数式接口都加上了@FunctionallInterface 注解标识
## 常见函数式接口
- Consumer 消费接口
![[Pasted image 20231220231559.png]]
- Function 计算转换接口
根据其参数列表和返回值类型我们可以知道，能在方法中传入参数计算或转换并返回结果
![[Pasted image 20231220231755.png]]
- Predicate 判断接口
![[Pasted image 20231220231858.png]]
- Supplier 生产型接口
![[Pasted image 20231220231944.png]]
## 常用默认方法
- And
我们在使用 Predicate 接口的时可能需要进行判断条件的拼接，而 and 方法相当于使用&&来拼接两个判断条件。
```java
//打印作家中年龄大于17并且姓名长度大于1的作家  
List<Author> authors = getAuthors();  
authors.stream()  
        .filter(((Predicate<Author>) author -> author.getAge() > 17).and(author -> author.getName().length()>1))  
        .forEach(author -> System.out.println(author.getName()+"==="+author.getAge()));
```
- Or
```java
//打印作家中年龄大于17或姓名长度小于2的作家  
List<Author> authors = getAuthors();  
authors.stream()  
        .filter(new Predicate<Author>() {  
            @Override  
            public boolean test(Author author) {  
                return author.getAge()>17;  
            }  
        }.or(new Predicate<Author>() {  
            @Override  
            public boolean test(Author author) {  
                return author.getName().length()<2;  
            }  
        })).forEach(System.out::println);
```
- Negate
```java
//打印作家中年龄不大于17的作家  
List<Author> authors = getAuthors();  
authors.stream()  
        .filter(new Predicate<Author>() {  
            @Override  
            public boolean test(Author author) {  
                return author.getAge()>17;  
            }  
        }.negate())  
        .forEach(System.out::println);
```
# 6.  方法引用
- 我们在使用 Lambda 时，如果方法体只有一个方法调用的话（包含构造方法），我们可以说使用方法引用进一步简化代码
## 基本格式
	类名或对象名:: 方法名
## 语法详解
- 引用类的静态方法
	格式：类名:: 方法名
- 引用对象的实例方法
	格式：对象名:: 方法名
- 引用类的实例方法
	格式：类名:: 方法名
- 构造器引用
	格式：类名::new
# 7. 高级用法
## 基本数据类型优化
- 我们用到很多 Stream 流都使用了泛型，所以涉及到的参数和返回值都是引用数据类型。
即使我们操作的都是整数、小数，但实际用的都是他们的包装类。JDK 5 中引用了自动装箱和自动拆箱让我们在使用对应包装类时就好像使用基本数据类型一样方便。但是自动拆装箱是需要消耗时间的，当数据量足够大时，我们需要对其进行优化。
Steam 中为我们提供了专门针对基本数据类型的方法：mapToInt，mapToLong，mapToDouble，flatMapToInt，flatMapToDouble 等
```java
List<Author> authors = getAuthors();  
authors.stream()  
        .map(Author::getAge)  
        .map(age -> age+10)  
        .filter(age -> age>18)  
        .map(age -> age+2)  
        .forEach(System.out::println);  

//优化后：
authors.stream()  
        .mapToInt(author->author.getAge())  
        .map(age -> age+10)  
        .filter(age -> age>18)  
        .map(age -> age+2)  
        .forEach(System.out::println);
```
## 并行流
- 当流中有大量元素时，我们可以使用并行流提高操作效率。其实并行流是将任务分配给多线程去完成。
parallel 方法可以将串行流转换为并行流
```java
Stream<Integer> stream = Stream.of(1,3,4,435,543,24,3,42,4,24,32,4,2,4,24);  
Integer sum = stream.parallel()  
        .peek(new Consumer<Integer>() {  
            @Override  
            public void accept(Integer num) {  
                System.out.println(num + Thread.currentThread().getName());  
            }  
        })  
        .filter(num -> num > 5)  
        .reduce((result, ele) -> result + ele)  
        .get();  
System.out.println(sum);
```
也可以直接使用 parallelStream 直接获取并行流对象
```java
List<Author> authors = getAuthors();  
Integer result = authors.parallelStream()  
        .distinct()  
        .map(Author::getAge)  
        .reduce(0, (result1, element) -> result1 + element);  
System.out.println(result);
```