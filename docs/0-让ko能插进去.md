# 通用篇：怎么做出一个能插进手机内核的 ko

**这是四台机器共用的前置知识。不先解决「ko 能不能插进去」，后面改什么都是空谈。**

---

## 一、先搞清楚三个拦路虎

内核拒绝一个 ko，无非这三个原因：

| 拦路虎 | 症状 | 怎么过 |
|---|---|---|
| vermagic 不一致 | `insmod` 报 `Invalid module format` | 见第三步 |
| 符号 CRC 对不上 | 同上 | 见第三步 |
| 模块签名强制 | 报 `Required key not available` | 见第二步 |

---

## 二、先确认你这台能不能插 ko

终端里：

```
zcat /proc/config.gz | grep -E "MODULE_SIG_FORCE|MODULES=|MODVERSIONS|TRIM_UNUSED"
```

> 没有 `/proc/config.gz` 就从机器里提 `kernel_config`，或者看厂商放出来的内核源码里的 defconfig。

**要看的几行**：

| 配置 | 期望值 | 不对会怎样 |
|---|---|---|
| `CONFIG_MODULES=y` | 必须是 y | 根本不支持加载模块 |
| `CONFIG_MODULE_SIG_FORCE` | **没有** 或 `is not set` | 开了就必须签名，基本没戏 |
| `CONFIG_MODVERSIONS` | 通常是 y | 影响下面的绕过手法 |
| `CONFIG_TRIM_UNUSED_KSYMS` | 常见是 y | 用不到的导出符号被裁掉了 |

**`MODULE_SIG_FORCE` 没开** 就可以继续。

---

## 三、关键技巧：`__versions` 空节

这是整件事里最有用的一个点。

### 内核怎么判断 vermagic

```c
static inline int same_magic(const char *amagic, const char *bmagic, bool has_crcs)
{
    if (has_crcs) {
        amagic += strcspn(amagic, " ");   // ← 跳过版本串，只比后半截
        bmagic += strcspn(bmagic, " ");
    }
    return strcmp(amagic, bmagic) == 0;
}
```

**`has_crcs` 为真时，内核会跳过版本号那一段**，只比较后面的 `SMP preempt mod_unload modversions aarch64`。

### 怎么利用

让你的 ko **带一个 `__versions` 节，但长度为 0**：

- 有这个节 → `has_crcs` 为真 → vermagic 只比后半截 → **版本号对不上也能过**
- 长度为 0 → 没有任何 CRC 条目 → **CRC 检查也跳过了**

一箭双雕。

### 怎么验证

```
llvm-objdump -h 你的.ko | grep __versions
```

应该看到长度是 `00000000`：

```
34 __versions    00000000  0000000000000000  DATA
```

---

## 四、`sizeof(struct module)` 必须和设备一致

这个坑很隐蔽：**编译出来的 ko 没问题，插进去也成功了，但行为是错的或者直接崩**。

原因是内核加载模块时会按 `struct module` 的布局去读写字段。你编译环境算出来的大小和设备上的不一样，字段就全错位了。

### 怎么查设备上是多少

设备报告里通常能拿到。或者找一个设备上现成的 vendor ko，看它的 `.gnu.linkonce.this_module` 节大小。

### 常见差异来源

```
CONFIG_DEBUG_INFO_BTF_MODULES
```

开了它，`struct module` 会多 16 字节，再按缓存行对齐，**1024 → 1088**。

K70e 实测设备上是 **1088**，而默认编译出来是 1024。差 64 字节，`exit` 函数指针的位置就从 984 跑到 968，内核会读写到越界的地方。

### 怎么对齐

在内核源码的 `include/generated/autoconf.h` 里手动加一行：

```c
#define CONFIG_DEBUG_INFO_BTF_MODULES 1
```

> ⚠️ **加完不要再跑 `make syncconfig`**，会把这行冲掉。

---

## 五、外部符号必须是设备上真有的

你的 ko 用到的每个内核函数，设备上都得有，否则 `insmod` 报 `Unknown symbol`。

### 怎么看你的 ko 依赖哪些

```
llvm-nm -u 你的.ko
```

### 控制依赖数量的技巧

**少用花哨的内核 API。** 比如：

| 想用 | 问题 | 换成 |
|---|---|---|
| `strscpy` | 会牵出 `__fortify_panic` / `strnlen` | 手写一个有边界的复制循环 |
| `atomic_inc_return` | 会牵出 `alt_cb_patch_nops` | 普通计数器（日志这种场景够用） |
| `drm_kms_helper_hotplug_event` | 会并发重建链表，有崩溃风险 | 不用，让用户自己切一下档位 |

### 弱符号的坑

想调用一个「可能存在也可能被裁掉」的导出函数，第一反应是声明成弱符号：

```c
extern struct mtk_panel_ext *find_panel_ext(struct drm_panel *panel) __weak;
```

**但这在 arm64 上有陷阱**：编译器会生成 `R_AARCH64_CALL26` 重定位。符号不存在时地址是 0，这条重定位超出跳转范围，**整个模块加载失败**。

```
llvm-readelf -r 你的.ko | grep 那个符号
```

看到 `R_AARCH64_CALL26` 就说明有这个风险。

**更安全的做法**：改用 kprobe 去钩那个函数。钩子挂不上最多是少一条路，不会拖垮加载。

---

## 六、剥掉调试符号

刚编出来的 ko 可能有几百 KB，其中绝大部分是调试信息。

```
llvm-strip --strip-debug 你的.ko
```

**K70e 实测：350KB → 20KB**，`.text` 本来就只有几 KB。

调试节是非 ALLOC 的，根本不会进内核内存，剥掉对功能零影响。

---

## 七、kprobe 符号名必须带模块前缀

一台手机上装了十几个面板模块：

```
panel_samsung_ana6705_cmd
panel_nt37801_cmd_fhd
panel_truly_nt35595_cmd
panel_alpha_jdi_nt36672e_vdo_120hz
panel_n11a_42_02_0a_dsc_vdo        ← 你要的
```

它们里面都有**同名的 static 函数**（`lcm_get_modes`、`mtk_panel_ext_param_set`、`mode_switch`）。

### 正确写法

```c
static struct kretprobe kr = {
    .kp.symbol_name = "panel_n11a_42_02_0a_dsc_vdo:lcm_get_modes",
    ...
};
```

**不带前缀会挂到别的面板模块上，钩子永远不触发。** 症状是「模块装上了但一点反应没有」—— 这个坑很容易卡住好几个小时。

### 一定要把注册结果打出来

```c
rc = register_kretprobe(&kr);
pr_info("hook lcm_get_modes = %d (0=成功)\n", rc);
```

**不打这个，出问题时完全不知道是钩子没挂上还是逻辑错了。**

---

## 八、安全写法（避免把日常机搞崩）

| 规矩 | 为什么 |
|---|---|
| 读内核内存一律用 `copy_from_kernel_nofault` | 偏移算错了只是返回错误，不会崩内核 |
| 写之前先做数值自检 | 比如 MTK 的 `data_rate == pll_clk × 2`，对不上就拒绝写 |
| 克隆的结构体放模块自己的 `.bss` | 不要 `kmalloc` 一块交给驱动去 `free`，会双重释放 |
| 不要往 DRM 链表里插/删节点 | 很多手机开了 `CONFIG_DEBUG_LIST`，并发改动直接触发 BUG 重启 |
| 不要留旧指针准备还原时回写 | 结构体会被重新分配，那是 use-after-free |
| 卸载时把改过的指针指回原厂 | 否则悬空指针，下次切档必崩 |
| **不要装开机自动加载** | 长按电源键重启就回原厂 —— 这是最后的保险 |
| **不要碰 `settings`** | 会留下卡死的刷新率，重启都不恢复 |

---

## 九、加载和卸载

### 加载

```
insmod 你的.ko
```

有参数的话：

```
insmod 你的.ko 参数1=值1 参数2=值2
```

### ⚠️ 换新版本必须先卸干净

**内核不允许同名模块装两次。**

```sh
rmmod 模块名
insmod 新的.ko
```

**没先 rmmod 的话，新 ko 根本没插进去，跑的还是旧的。** 这个坑会让你以为「改了没用」，其实是改的东西压根没加载。

**建议在 ko 里打上版本号**，加载时打到日志第一行：

```
[ 12.340] === v5.1 目标 132/120/90/60 ===
```

**看一眼就知道跑的是哪一版**，不用猜。

### 查看是否加载成功

```
grep '^模块名 ' /proc/modules
dmesg | tail -40
```

---

## 十、调试：让 ko 自己说话

折腾这种东西，**最浪费时间的就是「不知道卡在哪一步」**。

建议在 ko 里做一个环形日志，通过模块参数读出来：

```c
module_param_cb(log, &log_ops, &log_dummy, 0444);
```

然后：

```
cat /sys/module/你的模块/parameters/log
```

**要记的关键信息**：

- 版本号（认错版本白忙）
- 每个钩子的注册返回值（0 = 成功）
- 每个钩子触发了多少次（注册成功但次数是 0 = 符号对但代码没走到）
- 读到的原厂数值
- 写入的新数值
- 驱动最终真正采用的值 ← **这是判断生效与否的唯一硬指标**

### 写环形日志的一个坑

```c
int k = scnprintf(b, LOGW, "[%6llu.%03llu] ", ...);  // 实际只写了13个字符
vscnprintf(b + 15, LOGW - 15, fmt, a);               // 写到了结束符后面
```

消息落在 `\0` 后面，打印时读到第 13 位就停了，**日志全是空的只有时间戳**。

**必须按 `scnprintf` 的实际返回值接**，不要写死偏移。
