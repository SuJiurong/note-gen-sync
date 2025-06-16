# EasyExcel 监听器原理详解

## 监听器核心原理

EasyExcel 的监听器基于**观察者设计模式**实现，主要通过 `AnalysisEventListener` 抽象类来处理 Excel 读取过程中的各种事件。当 Excel 解析器逐行读取数据时，会自动触发监听器的相应回调方法。

## 核心接口与方法

`AnalysisEventListener` 提供了几个关键方法：

- `invoke(T data, AnalysisContext context)`：每成功解析一行数据时触发- `doAfterAllAnalysed(AnalysisContext context)`：所有数据读取完成后的回调方法- `invokeHeadMap(Map, String> headMap, AnalysisContext context)`：读取表头时的回调方法（可选实现）## 完整工作流程
  1. **初始化阶段**   - 创建监听器实例并进行必要配置
     2. **读取阶段**   - EasyExcel 逐行读取 Excel 文件   - 将每行数据转换为指定的对象类型（如 `BrandImportDTO`）   - 调用监听器的 `invoke` 方法处理数据对象
     3. **完成阶段**   - 所有数据读取完毕后，自动调用 `doAfterAllAnalysed` 方法进行收尾工作## 代码实现分析（BatchReadListener）

     ```java

     public class BatchReadListener> extends AnalysisEventListener> {
         // 配置参数
         private final int batchSize;          // 批处理大小
         private final Consumer>> batchConsumer;  // 批处理回调函数

         // 运行时状态
         private List> dataList = new ArrayList<>();  // 临时存储数据
         private int total = 0;    // 总记录数
         private int batchCount = 0;  // 已处理批次计数

         public BatchReadListener(int batchSize, Consumer>> batchConsumer) {
             this.batchSize = batchSize;
             this.batchConsumer = batchConsumer;
         }

         @Override
         public void invoke(T data, AnalysisContext context) {
             dataList.add(data);
             total++;

             // 达到批处理大小时触发处理
             if (dataList.size() >= batchSize) {
                 processBatch();
             }
         }

         @Override
         public void doAfterAllAnalysed(AnalysisContext context) {
             // 确保处理最后一批数据
             if (!dataList.isEmpty()) {
                 processBatch();
             }
             log.info("数据解析完成，共处理 {} 条数据，分 {} 批处理", total, batchCount);
         }

         private void processBatch() {
             batchCount++;
             log.info("正在处理第 {} 批数据，共 {} 条记录", batchCount, dataList.size());
             batchConsumer.accept(dataList);
             dataList = new ArrayList<>();  // 清空列表准备下一批
         }
     }
     ```
     ## 关键设计解析

     ### 1. 批量处理机制- 使用 `dataList` 临时存储数据- 当数据量达到 `batchSize` 时自动触发处理- 处理完成后立即清空列表，准备接收下一批数据

     ### 2. 回调函数设计- 采用 Java 8 的 `Consumer>>` 函数式接口- 实现业务逻辑与读取逻辑的解耦- 允许外部灵活定义批处理逻辑

     ### 3. 容错处理机制- `doAfterAllAnalysed` 确保处理最后一批不足 `batchSize` 的数据- 提供完整的处理统计信息

     ## 典型使用场景


     ```java

     // 初始化监听器
     BatchReadListener<BrandImportDTO> listener = new BatchReadListener<>(
         BATCH_SIZE,
         brandBatch -> {
             // 数据转换
             List<Brand> brands = brandBatch.stream()
                 .map(x -> {
                     Brand brand = ImportConvertor.INSTANCE.mapToBrand(x);
                     brand.setCreator("lego");
                     return brand;
                 })
                 .collect(Collectors.toList());

             // 批量保存
             brandService.saveBatch(brands);
             log.info("成功保存 {} 条品牌数据", brands.size());
         }
     );

     // 执行读取
     EasyExcel.read(file.getInputStream(), BrandImportDTO.class, listener)
         .sheet()
         .doRead();
     ```
     ## 技术优势对比

     相比 `doReadSync()` 同步读取方式：
     | 特性        | 监听器模式       | 同步读取模式      ||------------|----------------|----------------|| 内存效率     | 高（增量处理）    | 低（全量加载）    || 实时性       | 边读边处理       | 全部读完再处理    || 可靠性       | 避免内存溢出      | 大数据量易OOM    || 灵活性       | 可定制处理逻辑    | 固定处理方式      |

     ## 工作原理示意图

     ```
     Excel文件 → EasyExcel读取引擎 → 逐行读取并转换为对象
            ↓
     触发监听器invoke方法 → 数据存入临时列表
            ↓
     检查是否达到批处理阈值 → 是 → 执行批处理函数 → 清空临时列表
            ↓否
     继续读取下一行
            ↓
     文件读取结束 → 触发doAfterAllAnalysed → 处理剩余数据
     ```
     这种设计充分利用了观察者模式的优势，实现了高效、可扩展的大文件处理方案。通过监听器机制，开发者可以灵活控制数据处理流程，实现内存友好的批量操作。

     ```


     ```

     ```


     ```

     ```

     ```
