# 测试套件评分标准汇总

本文档汇总仓库中 12 组测试项目的评分方式，尽量以 `judge/` 下脚本和各题目说明文档为准。  
对于仓库内未找到独立评分脚本或说明不完整的项目，本文会明确标注来源限制。

## 1. 总体约定

- 评测系统会捕获 `#### OS COMP TEST GROUP START xxxxx ####` 和 `#### OS COMP TEST GROUP END xxxxx ####` 之间的输出。
- 评测脚本从标准输入读取这段输出，并返回 JSON 列表。
- JSON 列表中的每一项对应一个测试点，至少包含：
  - `name`: 测试点名称
  - `score`: 本测试点得分
- 可选字段：
  - `all`: 该测试点满分
  - `pass`: 通过数量
  - `result`: 原始测量值
  - `baseline`: 参考基线值

## 2. 各测试项目评分标准

### 2.1 basic

- 评分脚本：`judge/judge_basic.py`
- 评分方式：按每个 syscall 测试点逐项判分。
- 计分规则示例：
  - `brk`：共 3 分
    - 输出 `Before alloc,heap pos: X`
    - 输出 `After alloc,heap pos: Y` 且 `Y = X + 64`
    - 输出 `Alloc again,heap pos: Z` 且 `Z = Y + 64`
  - `chdir`：共 3 分
    - 输出 `chdir ret: 0`
    - 输出中包含 `test_chdir`
- 脚本中每个测试点都定义了独立的满分值，按断言通过数量计分。
- 参考文件：
  - `[judge_basic.py](oskernel-testsuits-cooperation/judge/judge_basic.py)`
  - `[basic.md](oskernel-testsuits-cooperation/doc/basic/basic.md)`

### 2.2 busybox

- 评分脚本：`judge/judge_busybox.py`
- 评分方式：`busybox_cmd.txt` 中每一条命令算 1 分。
- 判分规则：
  - 脚本实际执行 `busybox <command>`。
  - 若命令执行返回值为 0，则输出 `testcase busybox <command> success`，该项得 1 分。
  - 若返回值非 0，则输出 `testcase busybox <command> fail`，该项得 0 分。
  - 特例：命令文本包含 `false` 时，返回值非 0 也仍视为通过。
- 参考文件：
  - `[judge_busybox.py](oskernel-testsuits-cooperation/judge/judge_busybox.py)`
  - `[busybox.md](oskernel-testsuits-cooperation/doc/busybox/busybox.md)`

### 2.3 lua

- 评分脚本：`judge/judge_lua.py`
- 评分方式：`lua` 测试脚本列表中的每个 Lua 文件算 1 分。
- 判分规则：
  - 若执行后输出 `testcase lua <script> success`，该项得 1 分。
  - 否则得 0 分。
- 测试脚本清单：
  - `date.lua`
  - `file_io.lua`
  - `max_min.lua`
  - `random.lua`
  - `remove.lua`
  - `round_num.lua`
  - `sin30.lua`
  - `sort.lua`
  - `strings.lua`
- 参考文件：
  - `[judge_lua.py](oskernel-testsuits-cooperation/judge/judge_lua.py)`
  - `[lua.md](oskernel-testsuits-cooperation/doc/lua/lua.md)`

### 2.4 libctest

- 评分脚本：`judge/judge_libctest.py`
- 评分方式：每个测例 1 分。
- 判分规则：
  - 测例输出包含 `Pass!`，该项得 1 分。
  - 否则得 0 分。
- 该题主要看 libc 测试样例能否正确执行并输出通过信息。
- 参考文件：
  - `[judge_libctest.py](oskernel-testsuits-cooperation/judge/judge_libctest.py)`
  - `[libctest.md](oskernel-testsuits-cooperation/doc/libctest.md)`

### 2.5 iozone

- 评分脚本：`judge/judge_iozone.py`
- 当前仓库状态：存在评分脚本，但未找到独立题目文档。
- 评分方式：按多个 I/O 场景的吞吐量结果进行性能评分。
- 脚本提取的场景包括：
  - `iozone write/read`
  - `iozone random-read`
  - `iozone read-backwards`
  - `iozone stride-read`
  - `iozone fwrite/fread`
  - `iozone pwrite/pread`
  - `iozone pwritev/preadv`
- 脚本做法：
  - 从输出中提取 `Children see throughput for ...` 和 `Max throughput per process`。
  - 将结果与内置 baseline 对比。
  - 按 `result / baseline` 计算相对分数；若分数达到或超过 1，则按 `2 - 1/score` 进行压缩映射。
- 说明：
  - 脚本名为 `judge_iozone.py`，但内部变量名使用了 `lmbench`，属于脚本实现细节，不影响其“按吞吐量相对基线计分”的逻辑判断。
- 参考文件：
  - `[judge_iozone.py](oskernel-testsuits-cooperation/judge/judge_iozone.py)`
  - `[README.md](oskernel-testsuits-cooperation/README.md)`

### 2.6 unixbench

- 当前仓库状态：有题目说明文档，但未找到对应 `judge_unixbench.py`。
- 题目描述和评分方式在 `doc/Unixbench.md` 中写为：
  - 主要考察综合性能。
  - 文档明确写到“脚本中无评分依据，实现正确的情况下性能越高越好。”
- 结合现有材料，可确认它属于性能型题目，但仓库中没有单独可执行的评分脚本供进一步拆解。
- 参考文件：
  - `[Unixbench.md](oskernel-testsuits-cooperation/doc/Unixbench.md)`

### 2.7 iperf

- 当前仓库状态：有题目说明文档，但未找到对应评分脚本。
- 文档中的评分依据仅给出方向性要求：
  - 能否成功编译并运行服务器/客户端。
  - 是否支持 TCP/UDP 带宽测试。
  - 是否能理解输出并完成性能测试。
  - 是否能根据参数进行优化和异常处理。
- 因未找到独立脚本，仓库中无法进一步还原为“每个测试点多少分”的精确规则。
- 参考文件：
  - `[iperf.md](oskernel-testsuits-cooperation/doc/iperf/iperf.md)`

### 2.8 libcbench

- 当前仓库状态：有题目说明文档，但未找到对应评分脚本。
- 文档中的评分依据基于性能指标本身：
  - `time`: 执行时间，越短越好
  - `virt`: 虚拟内存占用，越低越好
  - `res`: 常驻内存占用，越低越好
  - `dirty`: 脏页数量，越低越好
- 这说明它是性能观测型题目，但仓库里没有脚本级别的分数拆分规则。
- 参考文件：
  - `[libcbench.md](oskernel-testsuits-cooperation/doc/libcbench/libcbench.md)`

### 2.9 lmbench

- 评分脚本：`judge/lmbench.sh`
- 当前仓库状态：脚本存在，且已包含 24 个测试项。
- 评分方式：
  - 脚本对每个测试项抓取输出中的数值，格式为 `test_name:number`。
  - 若本机有 `lmbench_all`，则直接调用系统命令；否则执行当前目录下对应程序。
  - 对不同类型测试项，脚本取值方式不同：
    - 默认取输出中的第一个浮点数
    - `lat_fs` 取第三、第四列并相加
    - `lat_mmap`、`bw_file_rd`、`bw_mmap_rd` 取第一列
  - 脚本最后输出每项的原始数值，供后续评测系统按基线换算分数。
- 脚本列出的测试项共 24 个：
  - `1:syscall null latency`
  - `2:syscall read latency`
  - `3:syscall write latency`
  - `4:syscall stat latency`
  - `5:syscall fstat latency`
  - `6:syscall open latency`
  - `7:select latency`
  - `8:sig install latency`
  - `9:sig catch latency`
  - `10:sig prot latency`
  - `11:pipe latency`
  - `12:proc fork latency`
  - `13:proc exec latency`
  - `14:proc shell latency`
  - `15:fs write bandwidth`
  - `16:pagefault latency`
  - `17:mmap latency`
  - `18:pipe bandwidth`
  - `19:file system latency`
  - `20:512k file read io_only`
  - `21:512k file read open2close`
  - `22:512k mmap read mmaponly`
  - `23:512k mmap read open2close`
  - `24:context switch latency`
- 说明：
  - 该脚本输出的是 `test_name:number` 格式，实际评分通常由评测系统结合基线做相对换算。
- 参考文件：
  - `[lmbench.sh](oskernel-testsuits-cooperation/judge/lmbench.sh)`
  - `[lmbench.md](oskernel-testsuits-cooperation/lmbench/lmbench.md)`

### 2.10 netperf

- 当前仓库状态：README 中列出了 `netperf`，但仓库中未找到对应题目文档和评分脚本。
- 因此无法从本仓库恢复其精确评分标准。

### 2.11 cyclictest

- 当前仓库状态：有题目说明文档，说明中直接给出了测试脚本行为，但仓库中未找到独立 `judge_cyclictest.py`。
- 评分方式：
  - 运行 4 次 `cyclictest`：
    - `NO_STRESS_P1`
    - `NO_STRESS_P8`
    - `STRESS_P1`
    - `STRESS_P8`
  - 每次若输出 `cyclictest * end: success`，说明该次运行成功。
  - 如果没有输出 `kill hackbench: success`，则 `STRESS_P1` 和 `STRESS_P8` 的结果会被舍弃。
  - 文档还说明会根据 `cyclictest` 的延迟输出对实时性做进一步评分，但未给出精确公式。
- 参考文件：
  - `[cyclictest.md](oskernel-testsuits-cooperation/doc/cyclictest/cyclictest.md)`

### 2.12 ltp

- 当前仓库状态：有题目说明文档，但内容更偏介绍性质，未找到独立评分脚本。
- 题目描述中说明 LTP 以 PASS / FAIL / TCONF 等结果分类。
- 结合仓库中现有说明，可确认其基本评分方向为：
  - `PASS`：通过，计分
  - `FAIL`：失败，不计分
  - `TCONF`：不适用，通常不计入总分
  - `BROK`：中断，通常不计入总分
- 但仓库中没有给出该题在本项目中的具体分值拆分规则。
- 参考文件：
  - `[ltp.md](oskernel-testsuits-cooperation/doc/ltp/ltp.md)`

## 3. 小结

- 评分规则已经明确写入脚本的项目：
  - `basic`
  - `busybox`
  - `lua`
  - `libctest`
  - `iozone`
  - `lmbench`
- 说明文档中提到、但仓库内未找到独立评分脚本的项目：
  - `unixbench`
  - `iperf`
  - `libcbench`
  - `netperf`
  - `cyclictest`
  - `ltp`
