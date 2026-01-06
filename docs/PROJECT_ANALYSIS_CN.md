# Ink/Stitch 项目深度分析报告

## 目录

1. [项目概述](#1-项目概述)
2. [项目结构分析](#2-项目结构分析)
3. [核心算法与功能](#3-核心算法与功能)
4. [输入输出与参数配置](#4-输入输出与参数配置)
5. [C++/Go 重写工作量评估](#5-cgo-重写工作量评估)
6. [非开源软件集成可行性分析](#6-非开源软件集成可行性分析)
7. [刺绣知识学习计划](#7-刺绣知识学习计划)

---

## 1. 项目概述

### 1.1 项目定位
Ink/Stitch 是一个基于 Inkscape 的开源机器刺绣设计平台，目标是为业余爱好者和专业数字化设计师提供完整的刺绣设计工具链。

### 1.2 技术栈
- **主语言**: Python 3.x
- **UI框架**: wxPython (原生对话框) + Flask (Web界面)
- **图形库**: Inkscape/inkex (SVG处理)
- **几何计算**: Shapely, NetworkX
- **刺绣格式**: pystitch (基于pyembroidery)
- **构建系统**: Makefile + Python脚本

### 1.3 许可证
**GPLv3** - 这是一个强制开源许可证，任何基于此代码的修改或衍生作品都必须以相同许可证开源。

---

## 2. 项目结构分析

### 2.1 目录结构
```
inkstitch/
├── inkstitch.py              # 主入口点
├── lib/                      # 核心库
│   ├── extensions/           # 82个Inkscape扩展实现
│   ├── elements/             # 刺绣元素类型定义
│   ├── stitches/             # 核心刺绣算法
│   ├── stitch_plan/          # 针迹计划数据结构
│   ├── gui/                  # GUI组件
│   ├── svg/                  # SVG处理工具
│   ├── threads/              # 线色管理
│   ├── lettering/            # 字体系统
│   ├── tartan/               # 格子图案系统
│   ├── inx/                  # INX文件生成
│   └── utils/                # 工具函数
├── bin/                      # 构建脚本
├── tests/                    # 测试套件
├── tiles/                    # 77+图案瓦片
├── fonts/                    # 嵌入字体
├── palettes/                 # 线色调色板
├── symbols/                  # 符号库
└── translations/             # 国际化文件
```

### 2.2 核心模块关系图
```
                    ┌──────────────────┐
                    │   inkstitch.py   │
                    │   (入口点)        │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  lib/extensions/ │
                    │  (扩展调度)       │
                    └────────┬─────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼───────┐   ┌───────▼───────┐   ┌───────▼───────┐
│ lib/elements/ │   │ lib/stitches/ │   │   lib/gui/    │
│ (元素定义)     │◄──│ (针迹算法)     │   │   (用户界面)   │
└───────┬───────┘   └───────┬───────┘   └───────────────┘
        │                   │
        └────────┬──────────┘
                 │
        ┌────────▼─────────┐
        │ lib/stitch_plan/ │
        │ (针迹计划生成)    │
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │    pystitch      │
        │ (文件格式输出)    │
        └──────────────────┘
```

### 2.3 代码统计
| 模块 | 文件数量 | 核心功能 | 复杂度 |
|------|---------|---------|--------|
| extensions/ | 82 | 用户命令入口 | 中等 |
| elements/ | 15 | 刺绣元素定义 | 高 |
| stitches/ | 15 | 核心算法 | 非常高 |
| stitch_plan/ | 10 | 数据结构 | 中等 |
| gui/ | 25 | 用户界面 | 中等 |
| svg/ | 10 | SVG处理 | 中等 |

---

## 3. 核心算法与功能

### 3.1 填充算法 (Fill Algorithms)

#### 3.1.1 自动填充 (Auto Fill)
**文件**: `lib/stitches/auto_fill.py`
**复杂度**: 非常高

核心步骤：
1. **光栅生成**: 在指定角度生成平行线阵列
2. **图形分割**: 使用Shapely计算多边形与光栅的交点
3. **路径优化**: 使用NetworkX图算法优化针迹顺序
4. **拉力补偿**: 调整形状边缘以补偿线材拉力
5. **随机化**: 添加针迹长度变化避免可见图案

关键参数：
- `fill_angle` - 填充角度 (度)
- `row_spacing` - 行间距 (mm)
- `max_stitch_length` - 最大针迹长度 (mm)
- `pull_compensation` - 拉力补偿值 (mm)

#### 3.1.2 轮廓填充 (Contour Fill)
**文件**: `lib/stitches/contour_fill.py`
沿形状轮廓由外向内收缩填充

#### 3.1.3 蜿蜒填充 (Meander Fill)
**文件**: `lib/stitches/meander_fill.py`
随机蜿蜒路径填充

#### 3.1.4 圆形填充 (Circular Fill)
**文件**: `lib/stitches/circular_fill.py`
从中心向外的螺旋/同心圆填充

#### 3.1.5 引导填充 (Guided Fill)
**文件**: `lib/stitches/guided_fill.py`
沿用户定义的引导线方向填充

### 3.2 缎面柱 (Satin Column)

**文件**: `lib/elements/satin_column.py`, `lib/stitches/auto_satin.py`
**复杂度**: 高

缎面柱系统使用"轨道-横档"(Rail-Rung)模型：
- **轨道 (Rails)**: 两条平行路径定义缎面柱边缘
- **横档 (Rungs)**: 垂直于轨道的短线定义针迹方向

关键算法：
1. 路径采样与对齐
2. 宽度补偿计算
3. 底衬针迹生成
4. Z字形/E型/S型针迹变体

### 3.3 轮廓线 (Stroke/Running Stitch)

**文件**: `lib/elements/stroke.py`
简单的沿路径运行针迹，支持多种变体：
- 运行针迹 (Running Stitch)
- 豆针迹 (Bean Stitch)
- 手工针迹外观 (Manual Stitch)

### 3.4 十字绣 (Cross Stitch)

**文件**: `lib/stitches/cross_stitch.py`
基于网格的十字绣图案生成

### 3.5 针迹计划系统 (Stitch Plan)

**文件**: `lib/stitch_plan/`

数据结构层次：
```
StitchPlan
    └── ColorBlock[]
            └── Stitch[]
                    ├── x, y 坐标
                    ├── color 颜色
                    └── flags (JUMP, TRIM, STOP, COLOR_CHANGE)
```

转换流程：
```
SVG Elements → Embroidery Elements → Stitch Groups → Stitch Plan → Output File
```

---

## 4. 输入输出与参数配置

### 4.1 输入格式

#### 4.1.1 主要输入
- **SVG文件**: Inkscape原生格式，包含Ink/Stitch专用命名空间属性

#### 4.1.2 刺绣文件导入
通过pystitch支持40+种刺绣格式导入：
- PES, DST, JEF, VIP, EXP, XXX, VP3, HUS
- 完整列表见pystitch/pyembroidery文档

### 4.2 输出格式

#### 4.2.1 刺绣格式
| 格式 | 描述 | 特点 |
|------|------|------|
| PES | Brother | 最常用家用格式 |
| DST | Tajima | 工业标准 |
| JEF | Janome | Janome机器 |
| EXP | Melco | 商业刺绣 |
| VP3 | Pfaff/Viking | 高端家用 |

#### 4.2.2 辅助输出
- **PNG**: 真实感预览图 / 简单预览图
- **CSV**: 针迹数据导出 (调试用)
- **PDF**: 打印输出 (颜色参考)
- **ZIP**: 打包多格式输出

### 4.3 参数配置系统

#### 4.3.1 文档级元数据
存储在SVG `<metadata>` 元素中：

```xml
<inkstitch:metadata>
{
    "min_stitch_len_mm": 0.1,
    "collapse_len_mm": 3,
    "thread-palette": "Default",
    "rotate_on_export": 0
}
</inkstitch:metadata>
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `min_stitch_len_mm` | float | 0.1 | 最小针迹长度 |
| `collapse_len_mm` | float | 3.0 | 跳跃折叠距离 |
| `thread-palette` | string | "Default" | 线色调色板 |
| `rotate_on_export` | int | 0 | 导出旋转角度 |

#### 4.3.2 元素级参数

**填充参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `fill_method` | enum | 填充方法 (auto_fill, contour, meander等) |
| `angle` | float | 填充角度 |
| `row_spacing_mm` | float | 行间距 |
| `max_stitch_length_mm` | float | 最大针迹长度 |
| `staggers` | int | 交错数量 |
| `pull_compensation_mm` | float | 拉力补偿 |
| `underpath` | bool | 启用底层路径 |

**缎面参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `satin_method` | enum | 方法 (satin/e_stitch/zigzag) |
| `pull_compensation_mm` | float | 宽度补偿 |
| `center_walk_underlay` | bool | 中心走线底衬 |
| `zigzag_underlay` | bool | Z字形底衬 |
| `contour_underlay` | bool | 轮廓底衬 |

**轮廓参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `stroke_method` | enum | 方法 (running/bean/manual) |
| `running_stitch_length_mm` | float | 针迹长度 |
| `bean_stitch_repeats` | int | 豆针重复次数 |

#### 4.3.3 用户全局设置
存储位置: `~/.config/inkstitch/settings.json` (Linux)

```json
{
    "cache_size": 100,
    "simulator_speed": 16,
    "simulator_line_width": 0.1,
    "default_font": "..."
}
```

#### 4.3.4 开发调试配置
**文件**: `DEBUG.toml`

```toml
[LIBRARY]
prefer_pip_inkex = true

[LOGGING]
log_config_file = "LOGGING.toml"

[DEBUG]
debug_type = "vscode"  # vscode/pycharm/pydev
debug_enable = true
disable_from_inkscape = true

[PROFILE]
profiler_type = "pyinstrument"  # cprofile/profile/pyinstrument/monkeytype
profile_enable = true
```

### 4.4 命令系统

SVG中的命令标记符号：
| 命令 | 功能 |
|------|------|
| `starting_point` | 自定义起点 |
| `ending_point` | 自定义终点 |
| `stop` | 机器暂停 |
| `trim` | 剪线 |
| `ignore_object` | 忽略对象 |
| `ignore_layer` | 忽略图层 |
| `origin` | 设计中心点 |

---

## 5. C++/Go 重写工作量评估

### 5.1 评估前提
基于资深全栈工程师视角，假设：
- 熟悉刺绣领域知识
- 精通目标语言
- 有几何算法经验
- 全职投入

### 5.2 模块复杂度评估

| 模块 | Python代码量 | 算法复杂度 | C++重写难度 | Go重写难度 |
|------|-------------|-----------|-------------|-----------|
| 核心算法 (stitches/) | ~5000行 | 非常高 | 高 | 高 |
| 元素系统 (elements/) | ~3000行 | 高 | 中 | 中 |
| 针迹计划 (stitch_plan/) | ~2000行 | 中 | 低 | 低 |
| SVG处理 (svg/) | ~2000行 | 中 | 高 | 中 |
| 格式IO | 依赖pystitch | 中 | 中 | 中 |

### 5.3 关键技术挑战

#### 5.3.1 几何计算库
**Python**: Shapely (基于GEOS)
**C++**: 可直接使用GEOS或CGAL
**Go**: 需要找替代库(go-geom)或封装GEOS

工作量: C++ 较低, Go 中等

#### 5.3.2 图算法库
**Python**: NetworkX
**C++**: Boost.Graph 或自实现
**Go**: gonum/graph

工作量: 两者相近，中等难度

#### 5.3.3 SVG解析
**Python**: lxml + inkex
**C++**: 需要自选XML库 + 实现SVG解析逻辑
**Go**: encoding/xml + 自实现

工作量: C++/Go 都需要大量自定义实现

#### 5.3.4 刺绣格式支持
**Python**: pystitch/pyembroidery (40+格式)
**C++**: libembroidery (C库)
**Go**: 需封装libembroidery或移植

工作量: C++ 中等, Go 高

### 5.4 工作量估算

#### 5.4.1 C++ 完整重写
| 阶段 | 工时估算 | 说明 |
|------|---------|------|
| 架构设计 | 2周 | 类设计、接口定义 |
| 基础设施 | 4周 | 构建系统、依赖集成、测试框架 |
| SVG解析 | 4周 | XML解析、路径解析、变换处理 |
| 核心算法 | 12周 | 填充算法、缎面算法、路径优化 |
| 元素系统 | 4周 | 元素类型、参数系统 |
| 格式输出 | 4周 | 集成libembroidery |
| UI层 | 8周 | Qt或其他GUI框架 |
| 测试&调试 | 6周 | 单元测试、集成测试、Bug修复 |
| **总计** | **44周 (~11个月)** | 单人全职 |

#### 5.4.2 Go 完整重写
| 阶段 | 工时估算 | 说明 |
|------|---------|------|
| 架构设计 | 2周 | 包结构、接口设计 |
| 基础设施 | 3周 | 模块系统、依赖管理 |
| SVG解析 | 5周 | XML解析、路径解析 |
| 几何库封装 | 4周 | CGO封装GEOS或使用go-geom |
| 核心算法 | 14周 | 算法移植、调试 |
| 元素系统 | 4周 | 结构体、接口实现 |
| 格式输出 | 6周 | CGO封装或移植格式代码 |
| UI层 | 10周 | fyne/Gio或Web前端 |
| 测试&调试 | 6周 | |
| **总计** | **54周 (~14个月)** | 单人全职 |

### 5.5 推荐策略

#### 5.5.1 最小可行方案 - 仅核心算法库
如果只需要刺绣算法能力（不含UI/Inkscape集成）：

**C++ 核心库**: 16-20周
- GEOS几何计算
- Boost.Graph路径优化
- libembroidery格式支持
- 暴露C API供其他语言调用

**Go 核心库**: 20-24周
- CGO封装GEOS
- gonum/graph路径优化
- CGO封装libembroidery

#### 5.5.2 渐进式方案
1. 先实现核心填充算法 (8周)
2. 添加缎面算法 (4周)
3. 添加格式输出 (4周)
4. 按需扩展其他功能

### 5.6 风险因素
1. **刺绣领域知识**: 不熟悉可能增加50%工时
2. **边缘情况**: 几何算法的边缘情况调试可能很耗时
3. **格式兼容性**: 刺绣格式文档不完整，需要逆向工程
4. **性能优化**: 大型设计可能需要额外优化

---

## 6. 非开源软件集成可行性分析

### 6.1 许可证限制

#### 6.1.1 GPLv3核心条款
Ink/Stitch使用**GPLv3**许可证，这是一个"强copyleft"许可证：

> **第5条 (传播修改版)**: 如果您传播基于本程序的作品，您必须以本许可证向任何接收副本的人授权整个作品。

#### 6.1.2 关键限制
1. **修改必须开源**: 任何基于Ink/Stitch代码的修改，在分发时必须以GPLv3开源
2. **链接传染**: 与GPLv3代码链接的代码也受GPLv3约束
3. **网络使用豁免**: GPLv3(非AGPL)不要求网络服务开源

### 6.2 合规集成方案

#### 6.2.1 方案A: 完全独立重写 ✅ 可行
**描述**: 基于对Ink/Stitch的学习，用C++/Go独立实现算法

**合规性**: 完全合规，只要：
- 不复制/翻译GPLv3代码
- 独立实现算法（算法本身不可版权）
- 不使用Ink/Stitch的任何原始代码

**风险**: 需要证明代码是独立实现的"净室开发"

**工作量**: 参见第5节评估

#### 6.2.2 方案B: 进程隔离调用 ⚠️ 有争议
**描述**: 闭源软件通过独立进程调用Ink/Stitch

**技术实现**:
```
闭源应用 ──(IPC/管道)──> Ink/Stitch进程 ──> 结果
```

**合规性分析**:
- FSF观点: 通过管道调用可能不构成"聚合作品"
- 但如果两者"密切集成"，可能被视为整体
- **灰色地带，存在法律风险**

#### 6.2.3 方案C: 使用替代开源库 ✅ 可行
**描述**: 使用更宽松许可证的刺绣库

**可用资源**:
- **libembroidery** (Embroidermodder项目) - Zlib许可证 ✅
- **pyembroidery** - MIT许可证 ✅

这些库提供：
- 40+刺绣格式读写
- 基本针迹操作
- 不包含高级填充算法

**工作量**: 需要自行实现填充/缎面算法

#### 6.2.4 方案D: 网络服务模式 ⚠️ 有条件可行
**描述**: Ink/Stitch作为独立网络服务运行

**合规性**:
- GPLv3不要求网络服务开源(AGPL才要求)
- 但用户必须能获取服务端源码(如果请求)
- **服务端代码必须保持GPLv3开源**

**适用场景**:
- 内部工具使用
- 提供公开的刺绣转换API服务

### 6.3 综合建议

#### 6.3.1 如果想完全闭源商业化
**推荐**: 方案A - 完全独立重写

**策略**:
1. 学习Ink/Stitch的算法原理（算法本身不受版权保护）
2. 参考学术论文和专利文献
3. 使用MIT/Zlib许可的库（libembroidery, pyembroidery）
4. 独立实现核心算法
5. 保留设计文档作为独立开发证据

#### 6.3.2 如果可以接受部分开源
**推荐**: 混合方案

1. 核心算法库独立实现(闭源)
2. 使用libembroidery处理格式(Zlib许可)
3. 格式转换层可以开源

#### 6.3.3 法律建议
在商业化之前，建议：
1. 咨询知识产权律师
2. 进行代码审计确保无GPLv3代码混入
3. 保留独立开发的证据链

### 6.4 技术可行性总结

| 方案 | 合规性 | 工作量 | 风险 | 推荐指数 |
|------|--------|--------|------|----------|
| 完全独立重写 | ✅ 高 | 高 | 低 | ⭐⭐⭐⭐⭐ |
| 进程隔离 | ⚠️ 争议 | 低 | 高 | ⭐⭐ |
| 替代库+自实现 | ✅ 高 | 中高 | 低 | ⭐⭐⭐⭐ |
| 网络服务 | ⚠️ 有条件 | 中 | 中 | ⭐⭐⭐ |

---

## 7. 刺绣知识学习计划

### 7.1 学习目标
- 理解机器刺绣的基本原理
- 掌握刺绣数字化设计概念
- 能够评估和优化刺绣设计
- 理解软件算法背后的刺绣逻辑

### 7.2 阶段一：基础知识 (2-3周)

#### 7.2.1 刺绣机器原理
**学习内容**:
- 机器刺绣vs手工刺绣的区别
- 刺绣机的基本结构（针杆、底线、面线、送布系统）
- 常见机器类型（家用、商用、工业）

**推荐资源**:
- YouTube: "How Embroidery Machines Work"
- Brother/Janome官方教程视频

#### 7.2.2 基本针法类型
**学习内容**:
| 针法 | 英文 | 用途 | 特点 |
|------|------|------|------|
| 跑针 | Running Stitch | 轮廓、细节 | 简单线条 |
| 缎面针 | Satin Stitch | 字母、细边 | 光滑、有光泽 |
| 填充针 | Fill Stitch | 大面积区域 | 密实、纹理 |
| 豆针 | Bean Stitch | 加粗轮廓 | 三重往返 |
| 十字绣 | Cross Stitch | 传统风格 | X形图案 |

#### 7.2.3 刺绣术语
**核心术语表**:
- **SPM/针速** - Stitches Per Minute
- **密度** - 针迹间距
- **底衬** - Underlay, 填充前的基础针迹
- **拉力补偿** - Pull Compensation, 补偿线张力收缩
- **跳针** - Jump Stitch, 移动但不刺绣
- **剪线** - Trim, 切断线头
- **换色** - Color Change
- **锁针** - Lock Stitch, 固定线头
- **起针点/收针点** - Entry/Exit Point

### 7.3 阶段二：设计原理 (3-4周)

#### 7.3.1 数字化基础
**学习内容**:
- 什么是数字化 (Digitizing)
- 矢量图形vs位图在刺绣中的应用
- 设计到针迹的转换过程

**实践练习**:
1. 使用Ink/Stitch将简单形状转为刺绣
2. 观察不同参数对针迹的影响

#### 7.3.2 填充策略
**学习内容**:
- 填充方向选择
- 分段填充技术
- 渐变填充效果
- 纹理填充

**核心概念**:
```
填充角度影响:
- 视觉效果：不同角度产生不同光泽
- 针迹长度：斜向填充针迹更短
- 稳定性：某些角度更容易拉扯织物
```

#### 7.3.3 缎面柱设计
**学习内容**:
- 轨道和横档的概念
- 宽度变化处理
- 转角处理技术
- 底衬策略

**设计原则**:
- 缎面宽度一般不超过12mm
- 使用底衬增加立体感
- 避免过长的平行针迹

#### 7.3.4 针迹顺序优化
**学习内容**:
- 最小化跳针
- 颜色分组
- 针迹顺序对成品的影响

### 7.4 阶段三：进阶技术 (4-6周)

#### 7.4.1 材料与针法匹配
| 面料类型 | 推荐针法 | 底衬 | 注意事项 |
|----------|---------|------|---------|
| 棉布 | 标准 | 中等 | 通用性强 |
| 针织 | 疏密适中 | 厚重 | 防止拉伸 |
| 皮革 | 稀疏 | 无/薄 | 避免过多穿孔 |
| 毛巾 | 密实 | 厚重 | 覆盖毛圈 |
| 丝绸 | 精细 | 轻薄 | 防止起皱 |

#### 7.4.2 常见问题诊断
**问题与解决方案**:
| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 起皱 | 密度过高、底衬不足 | 降低密度、增加底衬 |
| 断线 | 针迹过长、张力过大 | 缩短针迹、调整张力 |
| 边缘不平 | 拉力补偿不足 | 增加补偿值 |
| 填充稀疏 | 行间距过大 | 减小间距 |
| 线头外露 | 锁针不足 | 增加锁针 |

#### 7.4.3 高级填充技术
- 渐变填充
- 雕刻效果
- 3D泡沫刺绣
- 贴布绣

### 7.5 阶段四：软件算法理解 (4-6周)

#### 7.5.1 Ink/Stitch源码学习路径
```
推荐阅读顺序:
1. lib/stitch_plan/stitch.py     # 理解基本数据结构
2. lib/elements/stroke.py        # 最简单的元素类型
3. lib/elements/fill_stitch.py   # 填充元素
4. lib/stitches/auto_fill.py     # 核心填充算法
5. lib/elements/satin_column.py  # 缎面柱
6. lib/stitches/auto_satin.py    # 缎面算法
```

#### 7.5.2 关键算法深入
**自动填充算法核心流程**:
```python
1. 创建光栅线 (create_grating_lines)
   - 在指定角度生成平行线

2. 与多边形求交 (intersect_with_polygon)
   - 使用Shapely计算交点

3. 构建图 (build_graph)
   - 节点：交点
   - 边：相邻点连接

4. 找最优路径 (find_path)
   - 使用NetworkX的欧拉路径算法

5. 应用拉力补偿
   - 向外扩展边缘点
```

#### 7.5.3 几何算法学习
**推荐学习**:
- 计算几何基础（凸包、多边形操作）
- 图论基础（最短路径、欧拉路径）
- Shapely库使用
- NetworkX库使用

### 7.6 学习资源汇总

#### 7.6.1 在线课程
| 资源 | 类型 | 语言 | 推荐指数 |
|------|------|------|----------|
| Udemy "Machine Embroidery Digitizing" | 视频课程 | EN | ⭐⭐⭐⭐ |
| YouTube "Embroidery Talk" 频道 | 免费视频 | EN | ⭐⭐⭐⭐⭐ |
| 哔哩哔哩 刺绣教程 | 免费视频 | CN | ⭐⭐⭐ |

#### 7.6.2 书籍推荐
- 《Machine Embroidery: Tips, Techniques, and Troubleshooting》
- 《The Complete Photo Guide to Machine Embroidery》
- 《Digitizing Made Easy》 by John Deer

#### 7.6.3 软件资源
| 软件 | 价格 | 用途 |
|------|------|------|
| Ink/Stitch | 免费 | 学习、实践 |
| Embird | 付费 | 专业参考 |
| Wilcom | 专业 | 工业标准 |

#### 7.6.4 社区资源
- Ink/Stitch官网: https://inkstitch.org
- Inkscape论坛刺绣板块
- Reddit r/MachineEmbroidery

### 7.7 实践项目建议

#### 项目1: 简单Logo (第2周)
- 创建简单几何形状
- 应用填充和轮廓
- 导出并查看结果

#### 项目2: 字母设计 (第4周)
- 设计缎面柱字母
- 理解宽度补偿
- 优化针迹顺序

#### 项目3: 复杂填充图案 (第6周)
- 多色设计
- 不同填充方向组合
- 底衬策略

#### 项目4: 算法修改实验 (第8周)
- 修改填充算法参数
- 观察输出变化
- 理解参数关系

### 7.8 学习时间线总结

```
Week 1-3:   基础知识 - 机器原理、针法类型、术语
Week 4-7:   设计原理 - 数字化、填充、缎面柱
Week 8-13:  进阶技术 - 材料匹配、问题诊断、高级技术
Week 14-19: 算法理解 - 源码学习、几何算法
Week 20+:   持续实践 - 项目练习、社区参与
```

---

## 附录

### A. 常用命令参考
```bash
# 构建
make inx                    # 生成INX文件
make dist                   # 完整构建
make locales               # 生成翻译文件

# 测试
make test                  # 运行所有测试
pytest tests/test_xxx.py  # 运行单个测试

# 代码质量
make style                 # PEP8检查
make mypy                  # 类型检查
```

### B. 关键文件索引
| 文件 | 功能 |
|------|------|
| `inkstitch.py` | 主入口 |
| `lib/stitches/auto_fill.py` | 核心填充算法 |
| `lib/elements/satin_column.py` | 缎面柱元素 |
| `lib/stitch_plan/stitch_plan.py` | 针迹计划 |
| `DEBUG_template.toml` | 调试配置模板 |

### C. 格式兼容性表
| 格式 | 读取 | 写入 | 颜色支持 |
|------|------|------|---------|
| PES | ✅ | ✅ | 完整 |
| DST | ✅ | ✅ | 有限 |
| JEF | ✅ | ✅ | 完整 |
| EXP | ✅ | ✅ | 无 |
| VP3 | ✅ | ✅ | 完整 |

---

*报告生成时间: 2026-01-06*
*基于 Ink/Stitch 代码分析*
