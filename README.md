# 安卓屏幕刷新率超频 · 侧载 ko 完全教程

把内核模块（.ko）插进手机内核，在运行时改掉屏幕的刷新率档位。**不改分区、不刷 DTBO、重启就回原厂。**

仓库里有**完整可编译的源码**和**逐个值的解释**，照着能自己改出来。

---

## ⚠️ 第一步：先看你的屏是哪种

**这一步决定后面所有做法，搞错了白忙一整天。**

```sh
# 联发科
cat /proc/modules | grep panel

# 高通
ls /sys/firmware/devicetree/base/soc/qcom,mdss_mdp@*/ | grep panel
```

**看面板代号的最后三个字母：**

| 结尾 | 模式 | 屏幕怎么工作 | 怎么超频 | 难度 |
|---|---|---|---|---|
| **`_cmd`** | 命令模式 | DDIC 自带显存会自刷新，主机想发就发一帧 | 改 `mode->clock` 就行 | 简单 |
| **`_vdo`** | 视频模式 | 主机必须**连续不断**推像素流 | **必须抬 DSI 链路速率** | 难 |

```
帧率（命令模式）= 主机发帧的频率
帧率（视频模式）= 链路速率 ÷ (htotal × vtotal)
```

> 视频模式下**只改时序不会让屏幕变快**，只会改掉系统界面上显示的那个数字。
> 这是最容易白忙一整天的误区。

---

## 目录

| 文件 | 内容 |
|---|---|
| [`docs/0-让ko能插进去.md`](docs/0-让ko能插进去.md) | **先看这个**。vermagic、CRC、`struct module` 大小、符号依赖 |
| [`src/display_oc_vdo.c`](src/display_oc_vdo.c) | **视频模式完整源码**，600 行，全中文注释 |
| [`docs/1-视频模式-每个值的含义.md`](docs/1-视频模式-每个值的含义.md) | 源码里**每个数**代表什么、怎么查、怎么算 |
| [`docs/2-命令模式-怎么改.md`](docs/2-命令模式-怎么改.md) | 命令模式的改法 + **怎么反汇编别人的 ko 来学** |
| [`docs/3-怎么适配新机型.md`](docs/3-怎么适配新机型.md) | 手上没数据时，怎么一步步把参数摸出来 |

---

## 五分钟看懂视频模式的三个关键事实

### 1. 改 `mode->clock` 不会让屏幕变快

`mtk_dsi_calc_vdo_timing()` 从头到尾**没有拿像素时钟去写任何寄存器**。那个字段只被 `drm_mode_vrefresh()` 用来算给系统看的数字。

### 2. 链路速率只在上电那一刻能改

```c
static int mtk_dsi_poweron(struct mtk_dsi *dsi)
{
    if (++dsi->clk_refcnt != 1)
        return 0;                      // 不是第一次上电直接返回
    ret = mtk_dsi_set_data_rate(dsi);  // 全文唯一调用点
}
```

切档位走的是 DFPS 路径，**永远不碰 PLL**。改完必须**息屏再亮屏**，或者重启显示服务。

### 3. 驱动读链路速率有优先级

```c
if (dsi->d_rate != 0)                       // 1. 平时是 0
else if (DYN_MIPI_CHANGE && dyn_fps.data_rate)  // 2. ★ 默认走这条
else if (params->data_rate)                 // 3.
else if (params->pll_clk)                   // 4.
```

`MTK_DRM_OPT_DYN_MIPI_CHANGE` **默认值是 1**。只改 3 和 4 会被 2 整条压死。

---

## ⚠️ 新增档位会让序号重排（分情况）

这是 **MTK 面板驱动**的机制问题，**高通是另一套驱动，不一定有这回事**。

### 为什么会重排

MTK 面板驱动的回调吃的是**序号**：

```c
int (*ext_param_set)(struct drm_panel *, struct drm_connector *, unsigned int mode);
int (*mode_switch)(struct drm_panel *, struct drm_connector *,
                   unsigned int cur_mode, unsigned int dst_mode, ...);
```

你一新增档位，`drm_mode_sort` 按刷新率重排 `connector->modes`，**这些序号指向的就不是原来那一档了**。

### 命令模式：可以加档位，但不能信任序号

**解法是：序号只用来定位，判断依据用 `mode->clock`。**

```c
/* 按序号走到 connector->modes 的第 N 项 */
mode = 走链表(connector->modes, idx);
/* ★ 拿它的 clock 去和自己的克隆表比对，而不是拿序号去比 */
if (mode->clock == my_clone[k].clock)
    ext->params = &my_clone_params[k];
```

这样排序怎么变都不影响。现成的命令模式 ko 就是这么写的，它确实新增了档位并且能用。

### 视频模式：这招不够用，建议别加

视频模式下**时序本身依赖每档的参数和 DDIC 状态**。序号错一位，`mode_switch` 就会给 DDIC 发错档位的帧率命令：

| 症状 | 原因 |
|---|---|
| 选 120 跑 90、选 90 跑 60 | 加了 1 个档位，整体错一位 |
| 不管选什么都是最低档 | 加了多个档位 |
| **每档都正好减半**（130→65、60→30） | DDIC 停在低帧，主机按高帧推流，每两帧才刷一次 |

**所以视频模式推荐：档位数量一个不动，把原有的格子就地改，并保持名次。**

```
排序前(原厂)  [120, 90, 60, 30]   名次 0,1,2,3
排序后(改完)  [144,120, 90, 60]   名次 0,1,2,3   ← 一格没动
```

名次对得上，驱动怎么数都是对的。本仓库的 `src/display_oc_vdo.c` 走的就是这条路。

> ❌ **试过但错的做法**：在钩子里把序号翻译回去。结果更糟 —— 驱动拿到翻译后的序号，**转头又用它去取时序**，取到错的那一格。翻译序号这条路本身是错的。

## 怎么编译

```sh
# 1. 拿到对应的内核源码（GKI + 你机器的 kernel_config）
# 2. 编译
make -C <内核源码路径> M=$(pwd)/src ARCH=arm64 LLVM=1 \
     KBUILD_MODPOST_WARN=1 modules

# 3. 剥掉调试符号（350KB → 20KB）
llvm-strip --strip-debug src/display_oc_vdo.ko

# 4. 检查：__versions 必须存在且长度为 0
llvm-objdump -h src/display_oc_vdo.ko | grep __versions

# 5. 检查：不能有这两个符号（有的话说明还在新增档位）
llvm-nm -u src/display_oc_vdo.ko | grep -E "drm_mode_duplicate|drm_mode_probed_add"
```

细节见 [`docs/0-让ko能插进去.md`](docs/0-让ko能插进去.md)。

---

## 怎么用

```sh
# 加载
insmod display_oc_vdo.ko

# 生效
echo 1 > /sys/module/display_oc_vdo/parameters/install

# ★ 关键：让 DSI 走一次下电上电
#   息屏，等 3 秒，再亮屏

# 看日志
cat /sys/module/display_oc_vdo/parameters/log
cat /sys/module/display_oc_vdo/parameters/status

# 还原
echo 0 > /sys/module/display_oc_vdo/parameters/install
rmmod display_oc_vdo
```

### 日志长这样

```
[  12.340] === v1.0  目标 144/120/90/60  链路x144/120 ===
[  12.341] 参数#0 已改 dr 1100->1320 pll 550->660  PHY时序=自动
[  12.350] 名次0  120Hz -> 120Hz  clock=514751 vtotal=3341
[  12.353] 名次3   30Hz -> 144Hz  clock=514751 vtotal=2784
[  31.002] *** 驱动采用链路速率 = 1320 Mbps ***      ← ★判断生效的唯一硬指标
[  32.100] 实测帧率 144.02 Hz
```

---

## 爬坡方法

**判据：花屏 = 到顶了。**

每次**只改三个数**，结构一个字不动：

```c
#define TOP_HZ  130     // ← 想要的帧率
#define NUM     130     // ← 同上
#define DEN2    120     // ← 原厂最高档，不变
```

```
126 → 130 → 132 → 135 → 144 ...
```

花屏就退回上一档，那就是这块屏的上限。

> **别人说最高多少不算数。** 同型号不同批次的屏体质都不一样，**自己实测出来的才是你的上限。**

---

## 安全规矩

| 规矩 | 为什么 |
|---|---|
| **不要装开机自动加载** | 长按电源键重启就回原厂，这是唯一的保险 |
| **不要碰 `settings`** | 会留下卡死的刷新率，重启都不恢复 |
| 读内存用 `copy_from_kernel_nofault` | 偏移算错了不会崩内核 |
| 写之前校验 `data_rate == pll_clk × 2` | 对不上就是偏移错了，宁可不改 |
| 克隆结构体放模块 `.bss` | 不要 kmalloc 交给驱动去 free |
| 不碰 DRM 链表 | 很多手机开了 `CONFIG_DEBUG_LIST`，并发改动直接 BUG 重启 |
| 卸载时把改过的值全部还原 | 否则悬空指针，下次切档必崩 |

### 已经中招了怎么办

**刷新率卡死，重启也不恢复**：

```sh
settings delete system min_refresh_rate
settings delete system peak_refresh_rate
```

**黑屏/花屏回不来**：长按电源键重启。只要没装开机脚本，重启后就是原厂状态。

---

## 免责声明

折腾内核模块有变砖风险。**先确认自己会救砖再动手**，出了问题自己承担。

本仓库只讲原理和方法，不对任何后果负责。
