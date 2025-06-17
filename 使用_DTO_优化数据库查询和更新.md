# 使用 DTO 优化数据库查询和更新

这份笔记记录了如何通过使用 DTO (Data Transfer Object) 来优化数据库查询和更新的案例。

**背景**

在 `MallInfoServiceImpl` 类中，需要计算每个商场 (Mall) 的商品数量 (ProductCount)，并将结果更新到 `MallInfo` 表中。  初始实现存在性能问题，通过引入 DTO 和优化查询方式，显著提高了效率。

**原始实现 (已注释)**

```java
//        List<MallProduct> mallInfos = mallProductService.lambdaQuery()
//                .select(MallProduct::getMallId)
//                .groupBy(MallProduct::getMallId)
//                .list()
//                .stream()
//                .map(
//                        mallProduct -> {
//                            int productCount = mallProductService.lambdaQuery()
//                                    .eq(MallProduct::getMallId, mallProduct.getMallId())
//                                    .count();
//
//                            MallInfo mallInfo = new MallInfo();
//                            mallInfo.setId(mallProduct.getMallId());
//                            mallInfo.setProductCount(productCount);
//                            return mallInfo;
//                        }
//                )
//                .collect(Collectors.toList());
//
//        if (!mallInfos.isEmpty()){
//            this.updateBatchById(mallInfos);
//        }
```

这段代码存在以下问题：

1.  **多次数据库查询：**  首先查询所有不同的 `MallId`，然后针对每个 `MallId` 再次查询商品数量。  这导致大量的数据库交互。
2.  **潜在的性能问题：** 当商场数量很大时，对每个商场进行单独的 `count` 查询效率很低。
3.  **直接使用实体类：** 直接使用 `MallProduct` 实体类可能包含不必要的字段，增加数据传输量。

**优化后的实现 (当前代码)**

```java
public class MallInfoServiceImpl extends BaseMallInfoServiceImpl implements IMallInfoService {

    @Autowired IMallProductService mallProductService;

    @Autowired
    MallProductMapper mallProductMapper;

    @Override
    public void calcMallProductQuantity() {

        MPJLambdaWrapper<MallProduct> queryWrapper = new MPJLambdaWrapper<MallProduct>()
                .selectAs(MallProduct::getMallId, MallInfoCountDTO::getMallId)
                .selectFunc(() -> "COUNT(*)", "", "productCount")  // 修改这里，直接映射到 productCount
                .groupBy(MallProduct::getMallId);

                List<MallInfoCountDTO> mallInfoCountDTOList = mallProductMapper
                        .selectJoinList(MallInfoCountDTO.class,queryWrapper);

        //更新商品数量
        List<MallInfo> mallInfos = Optional.ofNullable(mallInfoCountDTOList)
                .orElse(Collections.emptyList())
                .stream()
                .filter(x -> x.getMallId() != null)  // 过滤掉 mallId 为 null 的情况
                .map( x -> {
                    MallInfo mallInfo = new MallInfo();
                    mallInfo.setId(x.getMallId());
                    mallInfo.setProductCount(x.getProductCount());
                    return mallInfo;
                        }
                )
                .collect(Collectors.toList());

        if (mallInfos.isEmpty()){
            return;
        }

        // 改用循环单个更新，而不是批量更新
        for (MallInfo mallInfo : mallInfos) {
            this.updateById(mallInfo);
        }
    }
}
```

这段代码做了以下优化：

1.  **使用 DTO (`MallInfoCountDTO`)：** 创建了一个专门用于数据传输的 DTO，只包含 `MallId` 和 `productCount` 两个字段。
2.  **`MPJLambdaWrapper` 一次性查询：** 使用 `MPJLambdaWrapper` (Multi-Projection Join Lambda Wrapper) 进行一次查询，利用数据库的 `COUNT(*)` 函数直接统计每个 `MallId` 的商品数量，并将结果映射到 `MallInfoCountDTO`。
3.  **Stream API 处理结果：** 使用 Java 8 Stream API 将 `MallInfoCountDTO` 转换为 `MallInfo` 对象。
4.  **循环单个更新：**  使用 `updateById()` 循环单个更新，可能是考虑到批量更新的性能问题或者数据量较小。
5.  **空值过滤：** 增加了 `filter(x -> x.getMallId() != null)` 过滤掉 `mallId` 为 null 的情况，避免空指针异常。

**`MallInfoCountDTO` 的定义 (推测)**

虽然代码中没有直接给出 `MallInfoCountDTO` 的定义，但可以推测其大致结构如下：

```java
public class MallInfoCountDTO {
    private Long mallId;
    private Integer productCount;

    // Getters and setters
    public Long getMallId() {
        return mallId;
    }

    public void setMallId(Long mallId) {
        this.mallId = mallId;
    }

    public Integer getProductCount() {
        return productCount;
    }

    public void setProductCount(Integer productCount) {
        this.productCount = productCount;
    }
}
```

**优势分析**

1.  **减少数据库交互：**  从多次查询减少到一次查询，显著提高了性能。
2.  **数据传输优化：** DTO 只包含必要的字段，减少了网络传输量。
3.  **代码可读性和维护性：**  DTO 明确定义了数据传输的结构，使代码更易于理解和维护。
4.  **解耦：** DTO 将数据访问层和业务逻辑层分离，提高了系统的灵活性和可扩展性。

**关键代码分析**

*   `.selectFunc(() -> "COUNT(*)", "", "productCount")`:  使用数据库的聚合函数 `COUNT(*)`，并在数据库层面完成统计，比在 Java 代码中处理更高效。  将结果映射到 DTO 的 `productCount` 字段。
*   `Optional.ofNullable(mallInfoCountDTOList).orElse(Collections.emptyList())`: 避免空指针异常，是一种良好的编程习惯。

**结论**

这个案例充分说明了 DTO 的价值。通过使用 DTO 和优化的查询方式，代码在性能、可读性、可维护性和架构方面都得到了提升。  它表明 DTO 不仅是一种理论上的最佳实践，更是解决实际问题的有效工具。  特别是在需要高效处理大量数据时，DTO 的优势更加明显。

创建于 2025-06-11 16:55:04。
