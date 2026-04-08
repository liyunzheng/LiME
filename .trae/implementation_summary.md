# 内存范围过滤功能实现总结

## 实现完成

已成功实现使用 `mem` 和 `size` 参数对 dump 的内存进行过滤的功能。

## 代码修改

### 1. 全局变量定义
**文件**: [src/main.c:88-89](file:///Users/bytedance/work/LiME/src/main.c#L88-89)
```c
unsigned long mem = 0;   // 起始物理地址
unsigned long size = 0;  // 要 dump 的大小（字节），0 表示 dump 全部
```

### 2. 参数验证
**文件**: [src/main.c:177-184](file:///Users/bytedance/work/LiME/src/main.c#L177-184)
- 添加参数日志输出
- 检查 mem + size 是否溢出
- 仅在 size > 0 时进行验证

### 3. 辅助函数
**文件**: [src/main.c:91-112](file:///Users/bytedance/work/LiME/src/main.c#L91-112)

#### `should_dump_range()`
判断内存资源是否需要 dump：
- size == 0: 返回 1（dump 所有）
- 资源完全在范围外: 返回 0（跳过）
- 资源与范围有重叠: 返回 1（需要 dump）

#### `clip_range()`
裁剪内存资源到指定范围：
- size == 0: 不裁剪
- start < mem: 裁剪 start 到 mem
- end >= mem + size: 裁剪 end 到 mem + size - 1

### 4. 内存遍历逻辑修改
**文件**: [src/main.c:242-282](file:///Users/bytedance/work/LiME/src/main.c#L242-282)

修改了 `init()` 函数中的内存遍历循环：
1. 检查资源是否在过滤范围内（`should_dump_range()`）
2. 如果不在范围内，跳过该资源及其子资源
3. 如果在范围内，创建临时 resource 结构体并裁剪（`clip_range()`）
4. 使用裁剪后的范围进行 dump
5. 更新 header 和 padding 逻辑以反映裁剪后的地址

## 功能特性

### 向后兼容
- 当 `size=0` 时，行为与修改前完全一致
- 默认参数值确保不影响现有用户

### 范围过滤
- **完全排除**: 内存区域完全在过滤范围外时跳过
- **完全包含**: 内存区域完全在过滤范围内时完整 dump
- **部分重叠**: 内存区域与过滤范围部分重叠时裁剪到重叠部分

### 格式支持
- **LIME 格式**: header 中的地址范围反映实际 dump 的范围
- **RAW 格式**: 仅输出过滤范围内的内存数据
- **PADDED 格式**: padding 基于 mem 参数计算

### 功能集成
- 与 digest 功能兼容：仅计算过滤后内存的摘要
- 与压缩功能兼容：仅压缩过滤后的内存数据
- 与 TCP/Disk 输出方法兼容

## 使用方法

### 基本用法
```bash
# Dump 所有内存（默认行为）
insmod lime.ko path=output.lime format=lime

# Dump 指定范围的内存
insmod lime.ko path=output.lime format=lime mem=0x10000000 size=0x10000000
```

### 参数说明
- `mem`: 起始物理地址（十六进制）
- `size`: 要 dump 的大小（十六进制），0 表示 dump 全部

### 示例场景

#### 1. Dump 前 256MB 内存
```bash
insmod lime.ko path=output.lime format=lime mem=0 size=0x10000000
```

#### 2. Dump 从 4GB 开始的 256MB
```bash
insmod lime.ko path=output.lime format=lime mem=0x100000000 size=0x10000000
```

#### 3. 结合摘要和压缩
```bash
insmod lime.ko path=output.lime format=lime mem=0x10000000 size=0x10000000 digest=sha256 compress=1
```

## 测试验证

### 编译测试
```bash
cd /Users/bytedance/work/LiME/src
make clean
make
```

### 功能测试
详细的测试场景请参考 [.trae/test_plan.md](file:///Users/bytedance/work/LiME/.trae/test_plan.md)

## 技术细节

### 地址类型处理
- 使用 `unsigned long` 类型存储 mem 和 size
- 与 `resource_size_t` 类型兼容
- 支持 32 位和 64 位系统

### 边界情况
- 正确处理 mem + size 溢出
- 正确处理跨越多个内存区域的过滤范围
- 正确处理过滤范围完全不在系统内存中的情况

### 性能影响
- 过滤逻辑在内存遍历阶段执行，不影响性能
- 跳过不需要的内存区域，可能提高性能

## 文件修改清单

- [src/main.c](file:///Users/bytedance/work/LiME/src/main.c): 主要实现文件
  - 添加全局变量定义
  - 添加参数验证
  - 添加辅助函数
  - 修改内存遍历逻辑

## 下一步建议

1. **编译验证**: 在实际内核环境中编译测试
2. **功能测试**: 在 QEMU 虚拟机中运行测试场景
3. **集成测试**: 添加到现有测试框架
4. **文档更新**: 更新 README 和用户文档
5. **性能测试**: 验证过滤功能对性能的影响

## 实现状态

✅ 全局变量定义  
✅ 参数验证  
✅ 辅助函数实现  
✅ 内存遍历逻辑修改  
✅ 向后兼容性保证  
✅ 多格式支持  
✅ 功能集成（digest, compress）  

实现已完成，等待编译和功能测试验证。
