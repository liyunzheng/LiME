# 内存范围过滤功能测试计划

## 实现总结

已完成以下修改：

### 1. 添加全局变量定义
- 在 [main.c:88-89](file:///Users/bytedance/work/LiME/src/main.c#L88-89) 定义了 `mem` 和 `size` 全局变量
- 默认值为 0，表示 dump 所有内存

### 2. 参数验证
- 在 [main.c:177-184](file:///Users/bytedance/work/LiME/src/main.c#L177-184) 添加了参数验证
- 检查 mem + size 是否溢出
- 添加调试日志输出过滤范围

### 3. 辅助函数
- [main.c:91-100](file:///Users/bytedance/work/LiME/src/main.c#L91-100) `should_dump_range()`: 判断内存范围是否需要 dump
- [main.c:102-112](file:///Users/bytedance/work/LiME/src/main.c#L102-112) `clip_range()`: 裁剪内存范围到指定区间

### 4. 内存遍历逻辑修改
- [main.c:242-282](file:///Users/bytedance/work/LiME/src/main.c#L242-282) 修改了内存遍历循环
- 应用范围过滤，跳过不在指定范围内的内存区域
- 对部分重叠的区域进行裁剪

## 功能特性

### 默认行为
- 当 `size=0` 时，dump 所有内存（保持向后兼容）
- 当 `size>0` 时，仅 dump 指定范围的内存

### 范围过滤逻辑
1. **完全排除**: 如果内存区域完全在过滤范围外，跳过该区域
2. **完全包含**: 如果内存区域完全在过滤范围内，完整 dump
3. **部分重叠**: 如果内存区域与过滤范围部分重叠，裁剪到重叠部分

### 边界处理
- 正确处理跨越多个内存区域的过滤范围
- 在 LIME_MODE_LIME 模式下，header 中的地址范围反映实际 dump 的范围
- 在 LIME_MODE_PADDED 模式下，padding 计算基于裁剪后的地址

## 测试场景

### 场景 1: 默认行为（无过滤）
```bash
insmod lime.ko path=output.lime format=lime
```
**预期**: dump 所有内存，行为与修改前完全一致

### 场景 2: 从地址 0 开始 dump 指定大小
```bash
insmod lime.ko path=output.lime format=lime mem=0 size=0x10000000
```
**预期**: 仅 dump 前 256MB 内存

### 场景 3: 指定起始地址和大小
```bash
insmod lime.ko path=output.lime format=lime mem=0x100000000 size=0x10000000
```
**预期**: dump 从 4GB 开始的 256MB 内存

### 场景 4: 跨越多个内存区域
```bash
insmod lime.ko path=output.lime format=lime mem=0x10000000 size=0x20000000
```
**预期**: dump 从 256MB 开始的 512MB 内存，可能跨越多个 System RAM 区域

### 场景 5: 部分重叠
```bash
# 假设系统有一个内存区域 0x100000000-0x1FFFFFFFF
insmod lime.ko path=output.lime format=lime mem=0x180000000 size=0x10000000
```
**预期**: 仅 dump 该区域的后半部分（0x180000000-0x18FFFFFFF）

### 场景 6: 范围完全不在系统内存中
```bash
insmod lime.ko path=output.lime format=lime mem=0xFFFFFFFF00000000 size=0x10000000
```
**预期**: 不 dump 任何内容，输出文件可能为空或仅包含 header

### 场景 7: LIME 格式验证
```bash
insmod lime.ko path=output.lime format=lime mem=0x10000000 size=0x10000000
# 检查 header 中的地址范围
hexdump -C output.lime | head -n 1
```
**预期**: header 中的 s_addr 和 e_addr 应该是 0x10000000 和 0x1FFFFFFF

### 场景 8: RAW 格式
```bash
insmod lime.ko path=output.raw format=raw mem=0x10000000 size=0x10000000
```
**预期**: 输出文件大小应该正好是 256MB

### 场景 9: PADDED 格式
```bash
insmod lime.ko path=output.padded format=padded mem=0x10000000 size=0x10000000
```
**预期**: 输出文件大小应该正好是 256MB（padding 基于 mem 参数）

### 场景 10: 结合 digest 功能
```bash
insmod lime.ko path=output.lime format=lime mem=0x10000000 size=0x10000000 digest=sha256
```
**预期**: 生成 output.lime 和 output.lime.sha256，digest 仅计算过滤后的内存

### 场景 11: 结合压缩功能
```bash
insmod lime.ko path=output.lime format=lime mem=0x10000000 size=0x10000000 compress=1
```
**预期**: 输出文件大小应该小于未压缩的版本

## 验证方法

### 1. 编译测试
```bash
cd /Users/bytedance/work/LiME/src
make clean
make
```

### 2. 功能测试
在 QEMU 虚拟机中运行上述测试场景，验证：
- 输出文件大小符合预期
- LIME header 中的地址范围正确
- digest 和压缩功能正常工作

### 3. 边界测试
- 测试 mem=0, size=0（默认行为）
- 测试 mem=0, size=MAX（dump 所有内存）
- 测试 mem=MAX, size=1（超出系统内存范围）
- 测试 mem + size 溢出情况

## 注意事项

1. **地址对齐**: mem 参数应该页对齐（4KB），否则可能导致部分页面被 dump
2. **大小对齐**: size 参数应该页对齐，否则会被向上取整到下一个页面边界
3. **内存区域**: 系统可能有多个不连续的 System RAM 区域，过滤范围可能跨越多个区域
4. **性能**: 过滤功能不会显著影响性能，因为过滤发生在内存遍历阶段
5. **兼容性**: 当 size=0 时，行为与修改前完全一致，保持向后兼容

## 下一步

1. 编译测试验证代码正确性
2. 在 QEMU 环境中运行功能测试
3. 添加单元测试到测试框架
4. 更新文档说明新参数的使用方法
