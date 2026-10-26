# nvtopa

[English](README.markdown) | [한국어](README.ko.markdown) | 中文 | [日本語](README.ja.markdown)

这是 [syllo/nvtop](https://github.com/syllo/nvtop) 的一个分支,重点是**温度监测**以及**无论终端多窄都不会被截断的设备信息头**。

![nvtopa 主界面](screenshot/nvtopa_main.png)

---

## 目录

- [本分支新增的功能](#本分支新增的功能)
- [键盘快捷键](#键盘快捷键)
- [截图](#截图)
- [测试环境](#测试环境)
- [构建与安装](#构建与安装)
- [文档](#文档)
- [许可证](#许可证)

## 本分支新增的功能

### 1. 宽度自适应的设备信息头

上游 nvtop 以大约 120 列的固定宽度排布设备信息头,因此在普通终端上,功耗、风扇和频率字段会直接超出右边缘。本分支在计算布局之前先把信息头适配到控制台宽度:沿着一条固定的**逐级放弃(give-up)阶梯**依次舍弃风扇填充、频率标签、ECC 和 NVLink 字段,最后是最小/最大读数,直到整行放得下为止。核心温度永远不会被完全舍弃。

```
Device 1 [NVIDIA GeForce RTX 3090] PCIe GEN 4@16x   NVL3 4x 0.000 KiB/s
1455MHz 9501MHz  TEMP  63° 46/66   JCT  74°C   VRAM  74°C   FAN  59%  POW 215 / 220 W
```

通过 pty 扫描验证:**40 到 200 列**之间的任何宽度都不会被截断。

### 2. 会话最小/最大温度,以及重置键

`TEMP`、`JCT` 和 `VRAM` 每个都显示当前读数,以及自程序启动(或上次重置)以来观测到的最小值和最大值:

- `TEMP 63° 46/66` — 当前 63 °C,最小 46 °C,最大 66 °C
- 按 **F7** 重置历史记录;最小/最大值收拢为当前读数,然后重新开始追踪。

最小/最大值在终端改变尺寸后仍然保留,因为它们保存在设备状态里,而不是随终端窗口重建。

### 3. PCIe 旁边的 NVLink 吞吐量

NVLink 传输速率从第二行信息头的末尾(窄终端上最先被截断的位置)移到了**第一行、紧跟 PCIe 吞吐量之后**:

```
Device 1 [NVIDIA GeForce RTX 3090] PCIe GEN 4@16x   NVL3 4x 0.000 KiB/s
```

PCIe 字段的窗口宽度会调整到它实际绘制的形状 — 详细形式
`PCIe GEN 4@16x RX: ... TX: ...`(46 列)、缩写形式
`GEN 4@16x RX 9.2G/TX 10.1G`(30 列),或者只有 `PCIe GEN 4@16x`(14 列)—
因此 NVLink 字段紧贴其后显示,而不是隔着一串空白。当空间彻底不够时(大约
60 列以下),舍弃 NVLink 并把空间还给 PCIe。没有桥接器的显卡和之前一样保留一个空白槽位。

### 4. 任何宽度下都保持原样的功耗格式

上游在窄终端上会把 `POW x / y W`(当前/最大)退化为单纯的 `POW x W`;本分支在所有宽度下都保持原来的"当前/最大"格式,只有当确实没有可舍弃的东西时才放弃整个字段。

### 5. 每个显卡最多 10 条曲线

每个 GPU 的图表一次最多接受 10 项指标(上游为 4 项)。默认在**每张设备图表中画出全部内容**,每条曲线有自己独立的颜色:

- GPU 利用率
- GPU 显存利用率
- GPU 核心温度
- 显存热点(junction)温度
- 显存(VRAM)温度

显卡不报告的传感器会自动跳过对应的曲线 — RTX 3050 没有显存温度传感器,因此不会绘制 `vram(c)` 曲线。

### 6. F8 — 循环切换图表内容

每张设备图表显示哪些指标取决于模式;**F8** 在各模式之间循环,并在切换时清空曲线缓冲区,使各模式的数据不会混在一起:

| 模式 | 绘制的曲线 |
| --- | --- |
| 全部指标(默认) | 利用率、显存利用率以及所有已报告的温度 |
| 原版 nvtop | 仅 GPU 利用率和显存利用率 |
| 仅温度 | 核心、热点、显存温度 |

### 7. 从 GDDR 寄存器读取显存热点与 VRAM 温度

这是本分支的原始功能,移植自
[ThomasBaruzier/gddr6-core-junction-vram-temps](https://github.com/ThomasBaruzier/gddr6-core-junction-vram-temps):
`JCT`(显存热点)和 `VRAM`(显存器件)读取自 GDDR6/GDDR6X/GDDR7 显存控制器的寄存器,任何受支持的 API 都不报告它们。仅适用于 Ampere(RTX 30)及更新的显卡;更老的显卡一律不读。通过 udev 规则把显卡的第一个 BAR 交给某个用户组后,普通用户也能读取 — 执行此配置的脚本见[机器配置仓库](https://github.com/xzrka/nvtopa-setup)。

## 键盘快捷键

| 按键 | 功能 |
| --- | --- |
| `F2` | 交互式设置窗口(Chart 页面配置曲线及其颜色) |
| `F6` | 对进程列表排序 |
| `F7` | **重置温度历史(最小/最大)** |
| `F8` | **循环切换图表内容**(全部指标 → 原版 nvtop → 仅温度) |
| `F9` | 终止进程 |
| `F10` | 退出 |
| `F12` | 保存配置 |
| `Ctrl+L` | 强制完整重绘 |

## 截图

三块 NVIDIA 显卡上的主界面 — 带最小/最大温度的宽度自适应信息头、PCIe 旁边的 NVLink,以及每张显卡一条五曲线图表:

![nvtopa 主界面](screenshot/nvtopa_main.png)

设置窗口(`F2`)— 在 Chart 页面可以单独开关各条曲线并切换它们的颜色:

![nvtopa 设置窗口](screenshot/nvtopa_setup.png)

## 测试环境

本分支在以下环境中开发与验证:

| 项目 | 内容 |
| --- | --- |
| GPU | 1 × NVIDIA GeForce RTX 3050(GDDR6,无显存温度传感器),2 × NVIDIA GeForce RTX 3090(GDDR6X,以 NVL3 4x 通过 NVLink 互连) |
| 操作系统 | Pop!_OS(Ubuntu 系),Linux x86-64 |
| 驱动 | NVIDIA 专有驱动(NVML + BAR0 寄存器访问) |
| 终端 | GNOME Terminal;通过 pty 测试工具进行无头验证 |

验证方式:

- **`tests/temp_colors.py`** — 在 pty 中重放程序,提取每个 `TEMP` / `JCT` / `VRAM` 数字上生效的 SGR 颜色,从而自动检查温度字段的颜色。运行方式:`python3 tests/temp_colors.py 80 24 -- ./build/src/nvtop`
- **`tests/pty_capture.py`** — 在任意尺寸的 pty 中运行程序并输出屏幕内容;用于从 40 到 200 列扫描信息头,验证每个宽度下的放弃阶梯。
- **`ctest`** — 上游测试套件,外加 `nvidia_vram_temps`:用假 sysfs 构造传感器寄存器并检查解码结果。
- **`tests/interfaceTests.cpp`** — PCIe 带宽表的单元测试。
- 在 **40、50、60、70、80、90、100、120、160、200 列**手动 pty 扫描,确认任何宽度下都不截断,且功耗字段始终保持 `POW x / y W` 格式;F8 的三种图表模式与 F7 重置在 pty 和 GNOME Terminal 实机中都做过验证。

## 构建与安装

标准 nvtop 构建:

```bash
git clone https://github.com/xzrka/nvtopa.git
cd nvtopa
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

在开发本分支的机器上,[机器配置仓库](https://github.com/xzrka/nvtopa-setup)中的 `install-nvtopa.sh` 会构建本分支并安装为 `nvtopa`(与发行版的 `nvtop` 并存),`unprivileged-temps.sh` 会设置允许普通用户读取显存温度寄存器的 udev 规则。

## 文档

- [`docs/UPSTREAM_README.markdown`](docs/UPSTREAM_README.markdown) — 完整的上游文档:厂商支持、各发行版安装方法、故障排除。
- 上游项目:<https://github.com/syllo/nvtop>
- 机器配置(udev 规则、安装脚本、假 sysfs):<https://github.com/xzrka/nvtopa-setup>

## 许可证

与上游 nvtop 相同:BSD-3-Clause。
