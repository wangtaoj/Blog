### Collector收集器泛型含义

```java
public interface Collector<T, A, R> {
    
    /**
     * 提供一个可变的容器，比如List, Map, Array等
     */
    Supplier<A> supplier();


    /**
     * 累加器消费函数
     * A是上面创建的容器类型
     * T是Stream流中的元素类型
     */
    BiConsumer<A, T> accumulator();

    /**
     * 组合器
     * 容器类的两个部分，一般都是并行流才会使用, 将两部分和成一个完整的容器
     */
    BinaryOperator<A> combiner();

   
    /**
     * 可对容器进行一次转换, 得到最终结果
     */
    Function<A, R> finisher();

    Set<Characteristics> characteristics();

}
```

### 代码片段

实体类

```java
public class Enti {
    private final Integer id;

    private final BigDecimal amt;

    public Enti(Integer id, BigDecimal amt) {
        this.id = id;
        this.amt = amt;
    }

    public Integer getId() {
        return id;
    }

    public BigDecimal getAmt() {
        return amt;
    }
}
```

```java
// 寻找下标
OptionalInt optionalInt = IntStream.range(0, list.size())
    .filter(i -> list.get(i).getId().equals(1))
    .findFirst();
if (optionalInt.isPresent()) {
    System.out.println(optionalInt.getAsInt());
}

// 分组并且获取每组的元素数量
Map<Integer, Long> collect = list.stream()
    .collect(Collectors.groupingBy(Enti::getId, Collectors.counting()));

// 分组收齐元素，并且对收集到的容器结果进行转换(注意不是容器中的元素)
Map<Integer, List<BigDecimal>> group = list.stream()
    .collect(
        Collectors.groupingBy(
            Enti::getId,
            Collectors.collectingAndThen(
                Collectors.toList(),
                resList -> resList.stream()
                    .map(Enti::getAmt).collect(Collectors.toList())
            )
        )
    );

// 分组收齐元素，并且对收集到的容器结果进行转换, 求和
Map<Integer, BigDecimal> group1 = list.stream()
    .collect(
        Collectors.groupingBy(
            Enti::getId,
            Collectors.collectingAndThen(
                Collectors.toList(),
                resList -> resList.stream().map(Enti::getAmt)
                    .reduce(BigDecimal.ZERO, BigDecimal::add))
    ));

// 自定义收齐器
Map<Integer, List<BigDecimal>> group3 = list.stream()
    .collect(Collectors.groupingBy(
        Enti::getId,
        Collector.of(
            ArrayList::new,
            (list1, item) -> list1.add(item.getAmt()),
            (left, rigth) -> {
                left.addAll(rigth);
                return left;
            },
            // 不对容器进行最终转换, 可以省略
            Function.identity()
        )));
```

