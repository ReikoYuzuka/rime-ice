# 自用修改说明

本文件记录我（ReikoYuzuka）对本仓库的个人改动，与上游雾凇拼音的原始设计无关。
上游文件保持原样、不做修改，所有个人内容放在独立目录里，方便随时同步上游更新。

---

## 一、目录约定

```
D:\Rime\
├── cn_dicts/          # 上游词库（字表、基础、扩展、腾讯词向量等）—— 不要动
├── en_dicts/          # 上游英文词库 —— 不要动
├── lua/               # 上游 Lua 扩展 —— 不要动
├── mydicts/           # 【自用】个人词库，放这里
│   └── dongfang.dict.yaml
├── rime_ice.dict.yaml # 方案词库入口，只在 import_tables 里加自用词库
└── rime_ice.custom.yaml  # 【自用】方案补丁（如 pin_cand_filter），可选
```

**核心原则：自用词库一律放 `mydicts/`，不要混进 `cn_dicts/`。**
这样 `git pull` 同步上游时，官方目录永远是干净的，不会冲突。

---

## 二、添加新词库的完整流程

### 第 1 步：转换源文件

细胞词库（`.scel`）用 **深蓝词库转换** 的 CLI 版：

```powershell
ImeWlConverterCmd.exe <源文件.scel> -i scel -o rime -O mydicts\<名字>.dict.yaml -t pinyin
```

**`-t pinyin` 必须显式加上。** 不指定编码类型时，工具可能把拼音按字符拆开输出
（例如 `八云紫` 变成 `ba yun zi` 是正确的，但英文词可能被拆成 `o p e n c o d e` 这种
逐字母形式，会严重破坏输入法——详见第五节「历史踩坑」）。

其他常见源格式：

| 源格式 | `-i` 参数 |
|---|---|
| 搜狗细胞词库 `.scel` | `scel` |
| 搜狗文本词库 | `sogou` |
| 搜狗备份 `.bin` | `sogou`（部分版本需先解密） |
| QQ 拼音 | `qqpy` |
| 百度拼音 | `bdpy` |
| 微软拼音 | `win10` |

转换后是**纯数据**，没有 YAML 头部，需要补上（见第 2 步）。

### 第 2 步：补 YAML 头部

工具输出的文件只有词条行，必须手工加头部，否则部署会失败：

```yaml
# Rime dictionary
# encoding: utf-8
#
# <这里写词库来源和说明>

---
name: <词库名，必须和文件名一致>
version: "1.0"
sort: by_weight
...

<词条数据从这里开始>
```

**`name:` 必须与文件名相同。** 例如 `mydicts/dongfang.dict.yaml` 里必须写 `name: dongfang`。

### 第 3 步：设置权重

见第三节。

### 第 4 步：挂载到方案

编辑 `rime_ice.dict.yaml`，在 `import_tables:` 列表**末尾**加一行：

```yaml
import_tables:
  - cn_dicts/8105     # 字表
  - cn_dicts/base     # 基础词库
  - cn_dicts/ext      # 扩展词库
  - cn_dicts/tencent  # 腾讯词向量
  - cn_dicts/others   # 一些杂项
  - mydicts/dongfang  # 东方Project 细胞词库（自用）
```

**路径是相对于用户目录（`D:\Rime`）的**，不是相对于 `rime_ice.dict.yaml` 的位置。
所以 `mydicts/dongfang` 会解析为 `D:\Rime\mydicts\dongfang.dict.yaml`。

雾凇的约定是「有重复词条时，最上面的权重生效」，所以自用词库放**最后**。
但这只影响同名同码的极端情况，实际排序几乎完全由权重决定。

### 第 5 步：部署并重启

```powershell
# 部署（重新编译词库）
& "D:\Program Files\Rime\weasel-0.17.4\WeaselDeployer.exe" /deploy

# 重启服务（让已打开的应用程序重新连接）
& "D:\Program Files\Rime\weasel-0.17.4\WeaselServer.exe" /restart
```

**两个都要做。** 只部署不重启，已经加载输入法的程序（比如浏览器）可能仍在用旧词库。

---

## 三、权重设置标准

### 权重的意义

权重（`sort: by_weight` 时的第三列）决定**候选排序**。数字大的排前面。

输入 `byz` 时，谁的权重大谁就在第一页：

```
八云紫    ba yun zi    2000000   ← 权重高，排前面
毕业证    bi ye zheng     62460   ← 权重低，被压下去
```

### 上游权重地形图（实测数据）

```
7600万  ── 「的」全库最高
3142万  ── 「是」
2957万  ── 「我」
1926万  ── 「江泽民」（base 峰值）
─────────── 以上是顶级高频字词，共约 21 条
 171万  ── 「部门」
 112万  ── 「北京」
─────────── base 的 100K-1M 段：10,851 条（2%）
  10万  ──
─────────── base 的 10K-99K 段：65,939 条（11.9%）
   1万  ── base 平均值
─────────── base 的 1K-9K 段：163,372 条（29.4%，最大一层）
  1000  ──
─────────── base 的 100-999 段：121,568 条（21.9%）
   100  ── ext 全部 339,189 条（统一压低）
    10  ──
─────────── base 的 1-9 段：114,776 条（20.7%）
     1  ── 细胞词库转换后的默认值（垫底）
```

来源：`cn_dicts/8105.dict.yaml`（8,758 条）、`base.dict.yaml`（554,892 条）、
`ext.dict.yaml`（339,189 条）。`tencent.dict.yaml` 未启用。

### 自用词库的权重标准

**细胞词库（专项词表：东方Project、Linux、动漫角色等）** —— 按词长分级：

| 词长 | 权重 | 说明 |
|---|---|---|
| 2 字 | **1000000** | 100 万 |
| 3 字 | **2000000** | 200 万 |
| 4 字及以上 | **3000000** | 300 万 |

**为什么按词长分级**：长词更独特、更不该被短词压。你打 `byz` 时「八云紫」（3字）
和「毕业证」（3字）平级竞争，你的词权重 200 万 > 6.2 万，胜出。
而 2 字词（如「彼岸」）会撞上更多常用词，不该那么霸道。

**为什么是百万量级**：能压过 `base` 的 100K-1M 和 10K-99K 两层（约 76,790 条，
占 base 的 13.8%），但**低于**「的」「是」「我」「北京」「部门」这些真正的日常高频词。
保证打单字母时日常字仍然稳居第一。

**个人日常词库（手机端导出、日常用词）** —— 建议：

| 词长 | 权重 | 说明 |
|---|---|---|
| 2 字 | **60000** | 6 万 |
| 3 字 | **80000** | 8 万 |
| 4 字及以上 | **100000** | 10 万 |

比细胞词库低一档。理由：个人词频只是「一个人」的使用习惯，
不该压过语言本身的通用高频词，但在同码竞争中应该赢多输少。

### 批量设置权重的脚本

不要用 `[string]::Join` 处理大文件——会内存溢出并把文件写成 0 字节（踩过坑）。
用 .NET 流式方法：

```powershell
$f = "D:\Rime\mydicts\xxx.dict.yaml"
$lines = [System.IO.File]::ReadAllLines($f, [System.Text.Encoding]::UTF8)
$new = New-Object System.Collections.Generic.List[string]
foreach ($line in $lines) {
    if ($line -match "`t" -and $line -notmatch '^#') {
        $p = $line -split "`t"
        $len = $p[0].Length
        $w = if ($len -le 2) { 1000000 } elseif ($len -eq 3) { 2000000 } else { 3000000 }
        $new.Add("$($p[0])`t$($p[1])`t$w")
    } else {
        $new.Add($line)
    }
}
[System.IO.File]::WriteAllLines($f, $new, (New-Object System.Text.UTF8Encoding($false)))
```

**必须用 `UTF8Encoding($false)`**（无 BOM）。带 BOM 的 YAML 会导致解析失败。

---

## 四、当前已添加的词库

| 文件 | 来源 | 条目 | 权重 | 挂载行 |
|---|---|---|---|---|
| `mydicts/dongfang.dict.yaml` | 【东方project】.scel | 306 | 100万/200万/300万 | `rime_ice.dict.yaml` |

---

## 五、历史踩坑记录

### 坑 1：拼音被逐字母拆开（最严重）

**症状**：打单字母全是英文（`s` 出 `zen zig zip zoo Zach`），简拼失效，
打 `zd` 候选框直接消失，所有中文单字都打不出来。

**原因**：词库里有一批英文词的拼音被写成了**逐字母拆开**的形式：

```yaml
opencode	o p e n c o d e	48
python		p y t h o n	47
uv		u v		38
```

Rime 的超级简拼规则（`rime_ice.schema.yaml:361` 的 `abbrev/^([a-z]).+$/$1/`）
是**按音节取首字母**的。上面这种写法让每个字母都成了独立音节，
于是 `o`/`p`/`e`/`n`/`c`/`d` 这些单字母编码被灌入几百个垃圾词条，
把真正的中文单字挤出了候选窗口。

**修复**：拼音必须写成**整体一个音节**：

```yaml
opencode	opencode	48
python		python		47
uv		uv		38
```

**预防**：转换时加 `-t pinyin`；转换后检查拼音字段有没有多个单字母音节。

### 坑 2：单字条目会被静默丢弃

**症状**：词库里的单字（`是	shi`、`的	de`）完全不生效，日志里刷几百条警告。

**原因**：Rime 编译词库时按「词+拼音」判重。`cn_dicts/8105.dict.yaml` 已经包含
全部常用单字，你的词库里的单字会被判定为重复而**全部丢弃**。

**修复**：词库里**不要放单字条目**，单字排序用 `pin_cand_filter` 解决
（见第六节）。

### 坑 3：权重不是一个量级

**症状**：词条明明在词库里，却永远排不到候选前面。

**原因**：搜狗导出的「词频」是个人使用计数（平均只有 4.3，最高 1131），
而雾凇的权重是**词库间相对优先级**（千万级）。差四个数量级。

**修复**：按第三节的标准重新赋值。细胞词库本来就是「专项词」，
不靠个人词频排序，直接给固定权重。

### 坑 4：变体目录测试无效

**症状**：改了一堆配置做实验，看起来「怎么改都没用」。

**原因**：实验用的副本目录里，`build/` 做成了指向真实 build 的符号链接，
librime 仍然加载真实编译产物，所有变体被静默忽略。

**教训**：做配置实验必须在**真实用户目录**，或者把 `build/` 复制成真实目录
并在其中重新编译。

### 坑 5：PowerShell 写文件的两个陷阱

1. `Out-File -NoNewline` 会把字符串数组拼成**单行**、且丢光换行——
   写出来的 YAML 会被 librime 当成一行，只解析出 `name:` 段，词库实际是空的。
   **一律用 `[System.IO.File]::WriteAllLines` 或 `WriteAllText`。**

2. `[string]::Join` 处理大文件会**内存溢出**，而且报错后仍会执行 `WriteAllText`，
   把文件清成 0 字节。**大文件用 `[System.Collections.Generic.List[string]]` 逐行累加。**

### 坑 6：`WeaselDeployer.exe /?` 会挂住

该参数不被识别，程序会转为启动 GUI 并一直等待，不返回。

正确用法：

```
WeaselDeployer.exe /deploy    # 部署
WeaselDeployer.exe /sync      # 同步
WeaselDeployer.exe /dict      # 用户词典管理
WeaselServer.exe /restart     # 重启服务
```

### 坑 7：部署完不重启服务

`/deploy` 会重新编译词库并让服务重新加载，但**已经加载输入法的应用程序不会自动刷新**。
必须再执行 `WeaselServer.exe /restart`，或手动切换输入法（Ctrl+Space 切走再切回）。

---

## 六、pin_cand_filter（可选：候选置顶）

如果需要某些高频词**强制排第一**（比调权重更硬），可以新建 `rime_ice.custom.yaml`：

```yaml
patch:
  # 语法：编码<Tab>词1 词2 词3    分隔符是真实 TAB，不是空格
  pin_cand_filter:
    - "d	的 地 得"
    - "s	是 时 使 式"
    - "zdl	子代理"
```

**性质**：
- 效力**高于**中文词库、英文词库、`custom_phrase`
- 只能**提升已有候选项的顺序**，没有自创编码的能力
- 候选不在前 100 个时提升不到
- 含空格的编码会自动生成「末音节首字母」简码
  （`zi dai li` → 自动派生 `zidaili`/`zidl`），但**不会**生成 `zd` 这类中间态，
  需要显式写出

实现见 `lua/pin_cand_filter.lua`。

`*.custom.yaml` 已被 `.gitignore` 排除，不会进入版本控制。

---

## 七、同步上游

```bash
git remote add upstream https://github.com/iDvel/rime-ice.git   # 只需一次
git fetch upstream
git merge upstream/main
```

因为自用内容都在 `mydicts/`、`rime_ice.custom.yaml` 和
`rime_ice.dict.yaml` 的一行 import 里，冲突面极小。

**唯一可能冲突的地方**：`rime_ice.dict.yaml` 的 `import_tables` 列表。
如果上游也改了这一段，手动合并时保留双方即可。

---

## 八、参考资料

- 深蓝词库转换：https://github.com/studyzy/imewlconverter
- 雾凇拼音：https://github.com/iDvel/rime-ice
- Rime 官方：https://rime.im
- `WeaselDeployer` CLI 参数：https://github.com/rime/home/issues/79
