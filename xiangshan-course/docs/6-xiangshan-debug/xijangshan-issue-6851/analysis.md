# Issue #6851：向量访存 trigger 命中字节与异常地址选择分析

> **基线**：昆明湖 V2，`kunminghu-v2` `e7bab53e66dfb3c4a1d11cf9519b0396f8576cae`，`KunminghuV2Config`、`NUM_CORES=1`、默认 `ISSUE=E.b`。修复验证使用**同一提交仅应用上游 #6882 补丁**。登记见 [VERSIONS.md](../VERSIONS.md)。

## 1. 概要

向量访存的地址 trigger 命中字节与 flow 首个 active byte 含义不同：前者是导致断点的访问位置，后者只是这个 16 字节访存片段中第一个参与访问的位置。旧版 `VMergeBuffer` 用普通访存 `mask` 选取后者，导致 `mtval` 地址错误；`vstart` 通过独立的命中偏移路径计算，因此仍正确。

例如屏蔽元素 0～3、在元素 8 命中 trigger 时，flow 的首个 active byte 是 `buf+4`，但断点地址应为 `buf+8`。修复使 trigger 异常使用 `vecTriggerMask` 计算地址偏移，普通异常继续使用原来的访存 `mask`。

已完成修复前重跑和修复后验证：修复前 A、G 通过，B～F 错误，`FAIL / BAD TRAP`、退出码 1；修复后 A～G 全通过，`PASS / GOOD TRAP`、退出码 0。测试输入、配置、基线和子模块均相同。

来源：[issue #6851](https://github.com/OpenXiangShan/XiangShan/issues/6851)。已有修复：[PR #6882](https://github.com/OpenXiangShan/XiangShan/pull/6882)，提交 `779e512d7af510fc9b506bedfda4cb7cdaabfe1e`。

## 2. 分析对象、版本与证据边界

| 对象 | 版本或材料 |
|---|---|
| 修复前源码 | `e7bab53e66dfb3c4a1d11cf9519b0396f8576cae` |
| 修复后源码 | 同一基线仅应用上游 `779e512d7af510fc9b506bedfda4cb7cdaabfe1e` 的修改 |
| difftest | `3729300ae233816d332d057f170472ebe35147b0`，两次未切换 |
| 配置 | `KunminghuV2Config`、单核、默认 `ISSUE=E.b` |
| 工具 | Mill 0.12.3、Verilator 5.052、Clang 19.1.1、OpenJDK 21 |
| 测试源码及链接脚本 | [vtrig.S](bug-replay/vtrig.S)、[link.ld](bug-replay/link.ld) |
| 输入及反汇编 | [vtrig.elf](bug-replay/vtrig.elf)、[vtrig.dump](bug-replay/vtrig.dump) |
| 输入 SHA-256 | `11c3b3fbdbbef7d74e850b2b93094058a942faeb39fff68dc67cf620aef23325` |
| 修复前结果 | [baseline-rerun-summary.txt](bug-replay/baseline-rerun-summary.txt)、[完整日志](bug-replay/baseline-rerun.log.gz) |
| 修复后结果 | [after-summary.txt](bug-replay/after-summary.txt)、[完整日志](bug-replay/after.log.gz) |

证据分为程序和反汇编、软件可见 CSR 日志、固定版本源码、生成 RTL 静态摘录。没有内部信号波形，不将源码推导当作逐周期观测。验证使用 `--no-diff`，由测试处理程序读取 CSR 自行判定，未执行 NEMU 或 Sail。

修复版的唯一受控源码差异是 `VMergeBuffer.scala`，与上游 `779e512d7af510fc9b506bedfda4cb7cdaabfe1e` 中该文件逐字节一致。日志显示 `e7bab53e66, dirty: 1`，其含义是基线加上游修复改动，不能写成完整 `779e512d7af510fc9b506bedfda4cb7cdaabfe1e` 代码树的测试。

归档中的个人文件系统路径已替换为仓库相对位置或通用占位符，日志行数、CSR 数值、源码身份、退出码和时间未改。构建日志属于历史记录，重跑方法见第 8 节。测试 ELF、源码和反汇编未修改。

## 3. 测试程序与触发条件

输入为 [vtrig.S](bug-replay/vtrig.S) 及其预编译 [vtrig.elf](bug-replay/vtrig.elf)。程序运行在 M 态，启用向量状态，设置 `mtvec`，置 `mstatus.MIE=1`，再配置 trigger 0：

- 加载触发器：`tdata1=0x6000000000000041`。
- 存储触发器：`tdata1=0x6000000000000042`。
- `tdata2=buf+OFF`，按地址相等匹配，`action=0`、`chain=0`、`size=0`。
- 每次触发后，异常处理程序记录 `mcause`、`mtval`、`vstart`，关闭触发器并清零 `vstart`，将 `mepc` 加 4 后返回。这里跳过的是本测试中的 4 字节访问指令，不是通用变长指令处理器。

报告打印前将记录的 `mtval` 减去 `buf`，因此日志中的 `buf+0x8` 表示相对测试缓冲区的偏移，不是绝对虚拟地址。CSR 结果由 guest 程序读取和检查；`--no-diff` 表示没有用 NEMU 独立比较。

各用例通过以下访问区分元素位置、flow 边界、掩码和标量路径：

| 用例 | 访问 | vl / 条件 | trigger 地址 | 期望 vstart |
|---|---|---|---|---:|
| A | `vle8.v` | vl=9 | buf+0 | 0 |
| B | `vle8.v` | vl=9 | buf+8 | 8 |
| C | `vle8.v` | vl=32 | buf+24 | 24 |
| D | `vle8.v v0.t` | vl=9，元素 0～3 inactive | buf+8 | 8 |
| E | `vle32.v` | vl=4 | buf+8 | 2 |
| F | `vse8.v` | vl=9 | buf+8 | 8 |
| G | 标量 `lb` | 访问 buf+8，事先清零 vstart | buf+8 | 0 |

测试来自 davidharrishmc 的 [复现仓库固定提交](https://github.com/davidharrishmc/xiangshan-repros/tree/6a70c0982e8b4ae4a3339ca0702a97ff013d96ee/xiangshan-vector-trigger-tval)。随包源码、链接脚本、ELF 和反汇编按原字节保留，未将作者参考输出作为独立实测证据。预编译 ELF 原始构建过程未独立复验，两次测试使用相同输入。

## 4. trigger 命中字节与 flow 首个 active byte

flow 是向量访存拆分后的一个 16 字节片段。普通 `mask` 描述本 flow 中哪些字节参与访问；`vecTriggerMask` 描述被选中的地址 trigger 命中了哪个字节。active 只表示“这次访问有效”，不表示“这个字节触发断点”。下表中的偏移均相对该 flow 起点，掩码为基于输入和源码推导的示意值，不是波形采样。

| 用例 | flow 起点 | active 字节偏移 | 首个 active 偏移 | trigger 命中偏移 | 旧 mtval | 应有 mtval |
|---|---|---|---:|---:|---|---|
| B | buf | 0～8 | 0 | 8 | buf+0 | buf+8 |
| C：第二个 flow | buf+16 | 0～15 | 0 | 8 | buf+16 | buf+24 |
| D | buf | 4～8 | 4 | 8 | buf+4 | buf+8 |

例如 B 的普通访问 mask 是 `0x01ff`（bit 0～8 有效），trigger mask 是 `0x0100`（只有 bit 8）。D 屏蔽了前四个元素，普通 mask 变成 `0x01f0`（bit 4～8 有效），trigger mask 仍是 `0x0100`。因此：

```text
旧选择：flow 起点 + first_active(mask)
B: buf + 0      D: buf + 4

trigger 修复后的选择：flow 起点 + first_active(vecTriggerMask)
B: buf + 8      D: buf + 8
```

`backend/fu/NewCSR/Debug.scala:236–273` 的固定版本代码进一步说明二者的语义：`lowMatch` 将 `tdata2` 低位对应的 one-hot 字节位置与访存 `mask` 相交，以确认该命中字节确实 active；触发后生成的 `triggerMask` 来自所选 trigger 的 `tdata2` 低位 one-hot 编码。它不是把全部 active 字节原样传下去。可在[固定版本源码](https://github.com/OpenXiangShan/XiangShan/blob/e7bab53e66dfb3c4a1d11cf9519b0396f8576cae/src/main/scala/xiangshan/backend/fu/NewCSR/Debug.scala#L236-L273)核对。

## 5. VMergeBuffer 的异常地址证据链

以下源码路径和行号均对应固定基线 `e7bab53e66dfb3c4a1d11cf9519b0396f8576cae`，可在 [XiangShan 远程源码](https://github.com/OpenXiangShan/XiangShan/tree/e7bab53e66dfb3c4a1d11cf9519b0396f8576cae/src/main/scala/xiangshan/mem)核对。以下描述是源码推导，与架构日志相互印证；没有运行时内部信号采样。

### 5.1 触发器的命中位置已被保留

`src/main/scala/xiangshan/mem/pipeline/LoadUnit.scala:1150–1155`：

```scala
s1_out.vecVaddrOffset := Mux(
  s1_trigger_debug_mode || s1_trigger_breakpoint,
  loadTrigger.io.toLoadStore.triggerVaddr - s1_in.vecBaseVaddr,
  s1_in.vaddr + genVFirstUnmask(s1_in.mask).asUInt - s1_in.vecBaseVaddr
)
s1_out.vecTriggerMask := Mux(s1_trigger_debug_mode || s1_trigger_breakpoint,
  loadTrigger.io.toLoadStore.triggerMask, 0.U)
```

触发器发生时，`vecVaddrOffset` 来自实际命中的 `triggerVaddr`，并且保存了 `vecTriggerMask`。`LoadUnit.scala:1431` 根据偏移和元素宽度形成 `vstart`：

```scala
s2_out.uop.vpu.vstart := Mux(s2_in.isLoadReplay || s2_in.isFastReplay,
  s2_in.uop.vpu.vstart, s2_in.vecVaddrOffset >> s2_in.uop.vpu.veew)
```

存储路径的对应代码是 `StoreUnit.scala:405–410` 和 `:502`。这解释了加载和存储用例中的 `vstart` 都能正确指向元素。

### 5.2 异常地址走另一条选择路径

`LoadUnit.scala:1864` 将 `s3_in.fullva` 送到 `io.vecldout.bits.vaddr`，`:1869–1870` 则分别送出 `vstart` 和 `vecTriggerMask`。存储侧在 `StoreUnit.scala:690–696` 也分别传递 `mask`、`vaddr`、`vstart` 和 `vecTriggerMask`。

两条路径进入共享的 `BaseVMergeBuffer`。其 `VMergeBuffer.scala:285–289` 使用：

```scala
val firstUnmask = genVFirstUnmask(selPort(0).mask).asUInt
val addrOffset  = Mux(entryIsUS, firstUnmask, 0.U)
val vaddr       = selVaddr + addrOffset
val vstart      = Mux(entryIsUS, selPort(0).vstart, selElemInfield)
```

这里 `mask` 是访存有效字节掩码。对 unit-stride 访问，计算的是 flow 地址加第一个有效字节偏移，未选择触发器的命中字节掩码。随后 `:296–299` 分别存入 `entry.vstart` 和 `entry.vaddr`；`:123` 又将 `source.vaddr` 传给异常反馈。因此，正确的 `vstart` 与错误的异常地址可以同时存在。

### 5.3 根因

根因是 **unit-stride 向量访存的异常地址生成没有区分普通访问掩码和触发器命中字节掩码**。触发器已经给出正确位置，但 merge buffer 重新依据访存 `mask` 选择地址，忽略了 `vecTriggerMask`。

例如 B 的 flow 有效字节从偏移 0 开始，而命中字节在偏移 8。旧代码取前者，得到 `buf+0`。D 中有效字节从偏移 4 开始，仍命中偏移 8，旧代码便输出 `buf+4`。这与本机日志吻合。

## 6. 地址选择修复

旧逻辑：

```scala
val firstUnmask = genVFirstUnmask(selPort(0).mask).asUInt
```

`mask` 表示 flow 中所有参与访存的有效字节，找到的只是首个 active byte。例如 D 屏蔽元素 0～3 后，首个 active byte 是 `buf+4`，但命中 trigger 的字节是 `buf+8`。用普通 `mask` 计算异常地址，会把两种位置混为一谈。

新逻辑：

```scala
val isTrigger   = TriggerAction.isExp(selPort(0).trigger) || TriggerAction.isDmode(selPort(0).trigger)
val excpMask    = Mux(isTrigger, selPort(0).vecTriggerMask, selPort(0).mask)
val firstUnmask = genVFirstUnmask(excpMask).asUInt
```

触发器异常或进入 debug mode 时，选择实际命中字节的 `vecTriggerMask`，再找其首个有效字节，得到应报告的偏移。其他异常继续使用原访存 `mask`。后面的 `addrOffset` 仍受 `entryIsUS` 控制，是 unit-stride 地址选择修复。

Load/Store 共用 `BaseVMergeBuffer` 的这段逻辑，因而同时修复向量加载和存储用例。`vstart` 原本使用实际触发偏移生成，补丁无需改动这条已经正确的路径。数据通路见本文第 5 节。

[rtl-mask-selection.txt](bug-replay/rtl-mask-selection.txt) 保存修复前后 Load/Store merge-buffer 的生成 RTL 摘录和完整文件哈希：旧 RTL 对 `mask` 做优先编码，新 RTL 先选择 `vecTriggerMask` 或普通 `mask`，再编码地址偏移。这是静态生成证据，不是内部信号波形。

上游修复的直接父提交为 `1d6139ed6`。使用同一基线只应用该提交的修改，可以避免两个上游版本之间其他改动干扰对照。完整构建日志见 [after-build.log.gz](bug-replay/after-build.log.gz)，构建命令为 `make emu CONFIG=KunminghuV2Config NUM_CORES=1 -j16`，构建退出码为 0。工具版本和子模块见第 2 节，运行结果及退出码见第 7 节。

## 7. 修复前后实测对照

下表地址均为日志中的 `mtval`，相对 `buf` 表示；全部用例在两次运行中均为 `mcause=3`，`vstart` 与期望一致。

| 用例 | 修复前实际 mtval | 修复后实际 mtval | 期望 mtval | vstart | 前 → 后 |
|---|---|---|---|---:|---|
| A | buf+0x0 | buf+0x0 | buf+0x0 | 0 | ok → ok |
| B | buf+0x0 | buf+0x8 | buf+0x8 | 8 | WRONG → ok |
| C | buf+0x10 | buf+0x18 | buf+0x18 | 24 | WRONG → ok |
| D | buf+0x4 | buf+0x8 | buf+0x8 | 8 | WRONG → ok |
| E | buf+0x0 | buf+0x8 | buf+0x8 | 2 | WRONG → ok |
| F | buf+0x0 | buf+0x8 | buf+0x8 | 8 | WRONG → ok |
| G | buf+0x8 | buf+0x8 | buf+0x8 | 0 | ok → ok |
| 结束 | FAIL / BAD TRAP | PASS / GOOD TRAP | PASS / GOOD TRAP | — | 退出码 1 → 0 |

归档日志分别为 [baseline-rerun.log.gz](bug-replay/baseline-rerun.log.gz) 和 [after.log.gz](bug-replay/after.log.gz)，摘要分别为 [baseline-rerun-summary.txt](bug-replay/baseline-rerun-summary.txt) 和 [after-summary.txt](bug-replay/after-summary.txt)。两份日志构成同一测试输入和配置的修复前后实测对照。

修复后仿真执行 `cycleCnt=70,792`、`instrCnt=9,756`，最终在测试结束 PC `0x80000302` 命中 GOOD TRAP。日志的主机用时为 `519,593ms`。修复前后的指令数和周期数还受测试结果打印长度、失败计数分支等影响，不能将其差值当作修复带来的性能收益。

## 8. 从远程仓库复现

### 8.1 获取交付材料及 XiangShan 源码

以下以任意工作目录为起点，从本课程仓库和 XiangShan 上游获取文件。先准备 Git、GNU Make、Python 3、Java、Mill、Verilator、C++ 编译器及 XiangShan 所需依赖；软件环境可参考 [第 1 章](../../1-xiangshan-development-environment/)。本例不是 nexus-am 程序，直接运行随包 ELF，无需设置 AM_HOME。

```bash
# 在自己选择的工作目录运行；以下布局为两个并列的仓库。
export WORKSPACE="$PWD"
git clone https://github.com/10leaf/XiangShanLab.git
export CASE_DIR="$WORKSPACE/XiangShanLab/xiangshan-course/docs/6-xiangshan-debug/xijangshan-issue-6851"

git clone --branch kunminghu-v2 https://github.com/OpenXiangShan/XiangShan.git
export NOOP_HOME="$WORKSPACE/XiangShan"
cd "$NOOP_HOME"
git switch --detach e7bab53e66dfb3c4a1d11cf9519b0396f8576cae
git submodule update --init --recursive
git -C difftest rev-parse HEAD
# 应输出 3729300ae233816d332d057f170472ebe35147b0
```

课程仓库 URL 使用本仓库的 origin；应选择包含本案例的远程分支或提交。若使用其他 fork，将第一个 clone URL 换成对应远程地址即可。后续路径由仓库布局推导，不依赖作者机器目录。已有仓库可从其根目录设置 `CASE_DIR` 和 `NOOP_HOME`，基线需与上表一致。

### 8.2 构建并运行修复前版本

在上一步的同一终端继续：

```bash
cd "$NOOP_HOME"
make emu CONFIG=KunminghuV2Config NUM_CORES=1 -j16

cd "$CASE_DIR"
# 预期退出码为 1；捕获退出码后继续后续步骤。
before_status=0
"$NOOP_HOME/build/emu" -i bug-replay/vtrig.elf --no-diff -C 2000000 \
    > bug-replay/replay-before.log 2>&1 || before_status=$?
printf 'before exit status: %s\n' "$before_status"
grep -aE '^[A-G] |^FAIL|HIT BAD TRAP' bug-replay/replay-before.log
```

应观察 A、G 为 `ok`，B～F 为 `WRONG`，最终 `FAIL / HIT BAD TRAP`。`required:` 前是实际值，括号内是期望值；必须查看具体 CSR 差异，不能仅凭任意 BAD TRAP 判断复现。

`-j16` 控制主机编译并行度，`NUM_CORES=1` 是处理器核数。`-C 2000000` 限制仿真周期数，不是运行时间；日志重定向期间终端可能暂时无输出，可在另一终端用 `tail -f bug-replay/replay-before.log` 观察。

### 8.3 从上游提交提取修复并重新运行

在已获取的 XiangShan 仓库内，从上游修复提交提取其实际改动，保留旧构建后应用到同一基线。子模块和测试输入保持不变：

```bash
cd "$NOOP_HOME"
fix_commit=779e512d7af510fc9b506bedfda4cb7cdaabfe1e
# 提取该提交相对于其父提交的修改，不切换整个代码树。
git diff "${fix_commit}^" "$fix_commit" -- \
    src/main/scala/xiangshan/mem/vector/VMergeBuffer.scala \
    > "$WORKSPACE/upstream-6882.diff"
git apply --check "$WORKSPACE/upstream-6882.diff"
mv build build.before-fix-6851
git apply "$WORKSPACE/upstream-6882.diff"
git diff --stat
# 唯一受控改动应为 src/main/scala/xiangshan/mem/vector/VMergeBuffer.scala
make emu CONFIG=KunminghuV2Config NUM_CORES=1 -j16

cd "$CASE_DIR"
after_status=0
"$NOOP_HOME/build/emu" -i bug-replay/vtrig.elf --no-diff -C 2000000 \
    > bug-replay/replay-after.log 2>&1 || after_status=$?
printf 'after exit status: %s\n' "$after_status"
grep -aE '^[A-G] |^PASS|^FAIL|HIT (GOOD|BAD) TRAP' bug-replay/replay-after.log
```

应观察 A～G 全部 `ok`，最终 `PASS / HIT GOOD TRAP`，退出码 0。不要重复应用修复；若 `build.before-fix-6851` 已存在，先选另一个备份名称。备份中的 `emu` 符号链接可能仍指向新 build，若要重跑旧版，使用 `build.before-fix-6851/verilator-compile/emu` 实体文件。

此处输出以 `replay-` 命名，不覆盖已有实测归档。按第 7 节逐项比较前后日志中 A～G 的 `mcause`、`mtval`、`vstart`；只有地址正确且全部用例通过，才确认本例修复有效。

### 8.4 不重新仿真，直接查看归档证据

```bash
cd "$CASE_DIR"
cat bug-replay/baseline-rerun-summary.txt
cat bug-replay/after-summary.txt
# 从完整日志提取测试结果
gzip -dc bug-replay/baseline-rerun.log.gz | grep -aE '^[A-G] |^FAIL|HIT BAD TRAP'
gzip -dc bug-replay/after.log.gz | grep -aE '^[A-G] |^PASS|HIT GOOD TRAP'
```

两份摘要保留完整日志的对应行号。查看实际值与括号内的期望值，并与第 7 节对照：修复前 B～F 地址错误，修复后 A～G 全部正确。构建配置由第 2 节的固定版本、工具信息和完整构建日志补充。

### 8.5 重新编译测试输入

```bash
cd "$CASE_DIR/bug-replay"
# 默认使用 riscv64-unknown-elf-gcc 和 riscv64-unknown-elf-objdump
make
# 或指定 Linux 交叉工具链，将新输入放在独立输出目录
make RVCC=riscv64-linux-gnu-gcc RVOBJDUMP=riscv64-linux-gnu-objdump \
    OUTPUT_DIR=rebuilt
```

重新编译的输入需另记哈希并重新运行，不保证与已验证 ELF 逐字节一致。Makefile 允许指定编译器、objdump 和输出目录。

## 9. 文件组织与交付证据

根目录为分析文档，`bug-replay/` 保存测试输入、反汇编和修复前后证据。

```text
xijangshan-issue-6851/
├── analysis.md
└── bug-replay/
    ├── vtrig.S / link.ld / Makefile                 测试源码及构建入口
    ├── vtrig.elf / vtrig.dump                       已验证输入与反汇编
    ├── baseline-rerun.log.gz / after.log.gz          修复前后完整日志
    ├── baseline-rerun-summary.txt / after-summary.txt
    ├── rtl-mask-selection.txt                      生成 RTL 的地址选择证据
    └── after-build.log.gz                          完整修复版构建记录
```

测试、方法、版本和配置见第 2、3、8 节；触发条件和根因见第 3～5 节；修复原理、已有 PR/commit、前后实测见第 6、7 节。材料按指定目录组织，不重复提交已有上游修复。

## 10. 验证范围与限制

本次验证覆盖 M 态、地址相等匹配、`action=0`、单个触发器、8/32 位 unit-stride 加载、8 位存储、跨 flow 的命中位置和带掩码访问，并有首元素及标量对照。没有采集内部波形，没有运行 NEMU/Sail，也没有覆盖 debug-mode action、其他匹配模式、S/U 态、分页或其他向量访存形态。原有标量和首元素用例通过，不等于完成所有非 trigger 异常的回归。

原 issue 引用 Sdtrig 对组合访问及 `mcontrol6 action=0` 故障地址的要求。本例没有归档规范快照，也不将作者 Sail 结果当作独立实测证据。源码行号和 flow 说明仅针对本页固定的昆明湖 V2 基线，不能外推至 V3。
