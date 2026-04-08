# 实现计划：使用 mem 和 size 参数过滤内存 dump 范围

## 目标
使用 `mem` 和 `size` 两个模块参数来限制 LiME dump 的内存范围，仅 dump 指定起始地址和大小范围内的内存。

## 当前状态分析

### 已有声明的参数
- [main.c:111-112](file:///Users/bytedance/work/LiME/src/main.c#L111-112) 已声明了 `mem` 和 `size` 参数：
  ```c
  module_param(mem, unsigned long, S_IRUGO);
  module_param(size, unsigned long, S_IRUGO);
  ```

### 缺失部分
1. **变量定义缺失**：虽然声明了 module_param，但没有定义对应的全局变量 `mem` 和 `size`
2. **参数验证缺失**：没有对参数的有效性进行检查
3. **过滤逻辑缺失**：在遍历内存范围时没有应用过滤

### 核心逻辑位置
- **内存遍历循环**：[main.c:207-229](file:///Users/bytedance/work/LiME/src/main.c#L207-229)
- **范围写入函数**：[main.c:311-384](file:///Users/bytedance/work/LiME/src/main.c#L311-384) `write_range()`
- **LiME header 写入**：[main.c:279-289](file:///Users/bytedance/work/LiME/src/main.c#L279-289) `write_lime_header()`

## 实现方案

### 1. 定义全局变量
在 [main.c](file:///Users/bytedance/work/LiME/src/main.c) 中添加全局变量定义（建议在第 102 行附近）：
```c
unsigned long mem = 0;  // 起始物理地址
unsigned long size = 0; // 要 dump 的大小（字节），0 表示 dump 全部
```

### 2. 参数验证和日志
在 `lime_init_module()` 函数中添加参数验证（建议在第 149 行后）：
- 检查 `mem` 和 `size` 参数的有效性
- 添加调试日志输出过滤范围
- 验证范围是否合法（mem + size 不溢出）

### 3. 实现范围过滤辅助函数
添加辅助函数判断内存范围是否需要 dump：
```c
static int should_dump_range(struct resource *res)
```

### 4. 实现范围裁剪逻辑
当内存范围部分重叠时，需要裁剪范围：
- 完全在过滤范围外：跳过
- 完全在过滤范围内：完整 dump
- 部分重叠：裁剪起始和结束地址

### 5. 修改内存遍历逻辑
在 [main.c:207-229](file:///Users/bytedance/work/LiME/src/main.c#L207-229) 的循环中：
- 应用范围过滤
- 对需要 dump 的范围进行裁剪
- 更新 header 和 padding 逻辑

## 详细实现步骤

### 步骤 1：添加全局变量定义
**位置**：[main.c:102](file:///Users/bytedance/work/LiME/src/main.c#L102) 附近
**内容**：
```c
unsigned long mem = 0;
unsigned long size = 0;
```

### 步骤 2：添加参数验证和日志
**位置**：[main.c:149](file:///Users/bytedance/work/LiME/src/main.c#L149) 之后
**内容**：
- 添加参数日志输出
- 验证参数合法性（如果 size > 0，则 mem 和 size 必须合法）

### 步骤 3：添加范围过滤辅助函数
**位置**：[main.c:86](file:///Users/bytedance/work/LiME/src/main.c#L86) 之后
**内容**：
```c
static int range_overlap(struct resource *res, unsigned long start, unsigned long end)
```

### 步骤 4：修改 init() 函数中的内存遍历循环
**位置**：[main.c:207-229](file:///Users/bytedance/work/LiME/src/main.c#L207-229)
**修改内容**：
- 在处理每个 RAM 资源前，检查是否在过滤范围内
- 如果不在范围内，跳过该资源
- 如果部分重叠，裁剪资源范围

### 步骤 5：修改 write_range() 函数
**位置**：[main.c:311-384](file:///Users/bytedance/work/LiME/src/main.c#L311-384)
**修改内容**：
- 支持传入裁剪后的范围参数
- 或者在调用前创建临时 resource 结构体

## 边界情况处理

1. **size = 0**：dump 所有内存（默认行为）
2. **mem = 0 且 size > 0**：从地址 0 开始 dump 指定大小
3. **范围完全不在系统内存中**：输出警告，不 dump 任何内容
4. **范围部分重叠**：只 dump 重叠部分
5. **范围跨越多个内存区域**：正确处理所有重叠区域

## 测试场景

1. 默认行为（mem=0, size=0）：dump 全部内存
2. 指定单个内存区域的一部分
3. 指定范围跨越多个内存区域
4. 指定范围完全不在系统内存中
5. 指定范围与系统内存无重叠

## 实现优先级

1. **高优先级**：基本过滤功能（完全包含/排除）
2. **中优先级**：范围裁剪功能（部分重叠）
3. **低优先级**：详细错误提示和边界检查

## 注意事项

1. 保持与现有代码风格一致
2. 不影响默认行为（当 mem=0, size=0 时）
3. 正确处理 resource_size_t 类型（32/64 位兼容）
4. 在 LIME_MODE_LIME 和 LIME_MODE_PADDED 模式下都要正确工作
5. 更新 header 中的地址范围以反映实际 dump 的范围
