我来详细讲解一下 EasyExcel 的监听器原理及其实现方式。
EasyExcel 监听器原理
EasyExcel 的监听器基于观察者设计模式实现，主要通过 AnalysisEventListener 接口来处理 Excel 读取过程中的各种事件。当 Excel 解析每一行数据时，会触发监听器的相应方法。
主要接口和方法
AnalysisEventListener<T> 是一个抽象类，包含几个关键方法：
invoke(T data, AnalysisContext context)：每读取一行数据时会调用此方法
doAfterAllAnalysed(AnalysisContext context)：所有数据读取完成后的回调方法
invokeHeadMap(Map<Integer, String> headMap, AnalysisContext context)：读取表头时的回调方法（可选实现）
工作流程
初始化阶段：创建监听器实例并配置
读取阶段：
EasyExcel 逐行读取 Excel 文件
每读取一行，将数据转换为指定的对象类型（如 BrandImportDTO）
调用监听器的 invoke 方法处理该数据对象
完成阶段：所有数据读取完毕后，调用 doAfterAllAnalysed 方法进行收尾工作
你代码中的监听器实现分析
我们来分析你代码中的 BatchReadListener 实现：
public class BatchReadListener<T> extends AnalysisEventListener<T> {
// 批处理大小
private final int batchSize;
// 批处理回调函数
private final Consumer<List<T>> batchConsumer;
// 临时存储数据
private List<T> dataList = new ArrayList<>();
// 计数器
private int total = 0;
private int batchCount = 0;

// 构造函数
public BatchReadListener(int batchSize, Consumer<List<T>> batchConsumer) {
    this.batchSize = batchSize;
    this.batchConsumer = batchConsumer;
}

@Override
public void invoke(T data, AnalysisContext context) {
    // 添加到临时列表
    dataList.add(data);
    total++;
  
    // 达到批处理大小，进行处理
    if (dataList.size() >= batchSize) {
        processBatch();
    }
}

@Override
public void doAfterAllAnalysed(AnalysisContext context) {
    // 处理最后一批数据
    if (!dataList.isEmpty()) {
        processBatch();
    }
    log.info("所有数据解析完成，共处理 {} 条数据，分 {} 批处理", total, batchCount);
}

// 处理批次数据
private void processBatch() {
    batchCount++;
    log.info("第 {} 批数据，处理 {} 条数据", batchCount, dataList.size());
    batchConsumer.accept(dataList);
    // 清空列表
    dataList = new ArrayList<>();
}
}
关键实现点解析
批量处理机制：
使用 dataList 临时存储数据
当数据量达到 batchSize 时触发处理
处理完成后清空列表，为下一批数据做准备
回调函数设计：
使用 Java 8 的 Consumer<List<T>> 函数式接口
允许外部代码灵活定义批处理逻辑
实现了业务逻辑与读取逻辑的解耦
完成处理机制：
doAfterAllAnalysed 方法确保最后一批不足 batchSize 的数据也能被处理
提供总体处理统计信息
在 Controller 中的使用方式
BatchReadListener<BrandImportDTO> listener = new BatchReadListener<>(
BATCH_SIZE,
brandBatch -> {
// 转换数据
List<Brand> brands = brandBatch.stream().map(x -> {
Brand brand = ImportConvertor.INSTANCE.mapToBrand(x);
brand.setCreator("lego");
return brand;
}).collect(Collectors.toList());
// 批量保存
brandService.saveBatch(brands);
log.info("成功保存{}条品牌数据", brands.size());
}
);

// 使用监听器读取文件
EasyExcel.read(file.getInputStream(), BrandImportDTO.class, listener)
.sheet()
.doRead();
初始化监听器：
设定批量大小（BATCH_SIZE = 500）
提供处理函数，定义每批数据的转换和保存逻辑
执行读取：
将监听器传入 EasyExcel.read() 方法
指定数据类型 BrandImportDTO.class
调用 doRead() 开始读取过程
相比一次性读取的优势
与 doReadSync() 方法相比：
内存效率：不会一次性加载全部数据到内存，特别适合处理大文件
实时处理：边读取边处理，避免大对象在内存中长时间停留
可靠性：即使数据量很大，也不会因内存溢出而失败
灵活性：可以根据不同业务场景定制处理逻辑
监听器工作原理图解
Excel文件 → EasyExcel读取引擎 → 逐行读取并转换为对象
↓
触发监听器invoke方法 → 添加到临时列表
↓
是否达到批处理大小? → 是 → 调用批处理函数 → 清空临时列表
↓ 否
继续读取下一行
↓
所有数据读取完毕 → 触发doAfterAllAnalysed → 处理剩余数据
这种设计充分利用了观察者模式的优势，实现了高效、可扩展的大文件数据处理。通过监听器机制，你可以灵活控制数据的读取和处理过程，实现内存友好的批量处理
