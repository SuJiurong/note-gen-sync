# EasyExcel 监听器原理详解

## 监听器核心原理

EasyExcel 的监听器基于**观察者设计模式**实现，主要通过 `AnalysisEventListener` 抽象类来处理 Excel 读取过程中的各种事件。当 Excel 解析器逐行读取数据时，会自动触发监听器的相应回调方法。

## 核心接口与方法

`AnalysisEventListener` 提供了几个关键方法：

- `invoke(T data, AnalysisContext context)`：每成功解析一行数据时触发- `doAfterAllAnalysed(AnalysisContext context)`：所有数据读取完成后的回调方法- `invokeHeadMap(Map, String> headMap, AnalysisContext context)`：读取表头时的回调方法（可选实现）## 完整工作流程
  1. **初始化阶段**   - 创建监听器实例并进行必要配置
     2. **读取阶段**   - EasyExcel 逐行读取 Excel 文件   - 将每行数据转换为指定的对象类型（如 `BrandImportDTO`）   - 调用监听器的 `invoke` 方法处理数据对象
     3. **完成阶段**   - 所有数据读取完毕后，自动调用 `doAfterAllAnalysed` 方法进行收尾工作## 代码实现分析（BatchReadListener）
