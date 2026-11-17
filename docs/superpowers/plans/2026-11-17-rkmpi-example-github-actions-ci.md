# Luckfox Pico RKMPI Example — GitHub Actions CI 实施计划

> **For agentic workers:** 作者同意本 PR 不必严格遵循 superpowers 的执行流程（根 `AGENTS.md`「优先级」）；按下面的 Task 顺序执行，复选框（`- [ ]`）用于跟踪。Task 1–10 已完成，代码入库后本 plan 已精简：已完成的 Task 只留落地指针，保留验证证据（§5）、偏离（§6）、作者执行与合并后的步骤（Task 11）以及可复用的命令（附录 A）。

**Goal:** 按 lvgl 仓 `dev`（`11497f3`）的做法为本仓新增 GitHub Actions：GHCR 镜像 + 两种 libc × 5 例程的交叉编译与兼容性检查、ShellCheck、文档检查（脚本入库）、根 `AGENTS.md` 体积检查，以及 `.github/AGENTS.md` 与相关说明。

**Architecture:** 推送 spec 与 plan、开草稿 PR（Task 1）；Dockerfile 取消注释（Task 2）；检查脚本入库（Task 3）；主 workflow 的 `build-image`、`build-example`（Task 4）；`shellcheck`、`doc-check`（Task 5）；体积检查 workflow（Task 6）；四份 `AGENTS.md`（Task 7）；本地 V1–V3（Task 8）；开发期 CI（Task 9）；回填 PR #1 plan、整理提交与 V5（Task 10）；作者执行的项（Task 11）。

**Tech Stack:** GitHub Actions（`ubuntu-24.04`）、GHCR、`actions/checkout@v7`、`docker/login-action@v4`、`actions/attest@v4`、`actions/upload-artifact@v7`、`gh attestation verify`、CMake、`gcc-arm-linux-gnueabihf`、`arm-rockchip830-linux-uclibcgnueabihf`、Python 3 + markdown-it-py、ShellCheck 0.9.0、actionlint。

**Spec:** [`docs/superpowers/specs/2026-11-17-rkmpi-example-github-actions-ci-design.md`](../specs/2026-11-17-rkmpi-example-github-actions-ci-design.md)

---

## 1. Global Constraints

- 分支 `cursor/github-actions-ci-d6e2`，起点 `origin/dev` `e130684`；PR 指向 fork `yuangezhizao/luckfox_pico_rkmpi_example` 的 `dev`，开发期为草稿，执行者不合并、不改为 ready（作者确认后再改）。
- 参考源一律取 lvgl 仓 `dev` `11497f3` 与 luckfox-pico 仓 `dev` `665c7f1`，用 `/tmp/lvgl`、`/tmp/lp` 的浅克隆或固定提交的 raw 链接。
- 不改 `CMakeLists.txt`、`build.sh`、`example/`、`lib/`、`include/`、`install/`；PR #1 的 spec 与 plan 只在单独的提交中修改（spec Q12、Q18）；`git add` 只写明确的路径；每个 Task 结束时 `git status --short` 只应出现本 Task 的文件。
- 本地验证构建不得改写工作区：仓库外的构建目录，`make install DESTDIR=…`（根 `AGENTS.md`）。
- 提交身份 `Cursor Agent <cursoragent@cursor.com>`，SSH 签名；创建或改写提交的 git 命令前加 `env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL`。本次 Run 的两个 co-author 钩子已改为 `远哥制造 <yuangezhizao@users.noreply.github.com>`；推送前 `git log --format=%B origin/dev..HEAD | grep -i co-authored-by` 只应出现该地址。
- rebase（含 fixup、reword）与 `--amend` 保留 author time，不用 `--reset-author`、`--ignore-date`、`--date`、`--committer-date-is-author-date`。
- 提交信息按根 `AGENTS.md`「提交约定」：`<type>(<scope>): <emoji> <描述>`，正文为「- 」开头的要点，PR 写完整链接或 `owner/repo#N`；末尾 `Refs: docs/superpowers/specs/2026-11-17-rkmpi-example-github-actions-ci-design.md §X`；spec 与 plan 放在 PR 的最后一个提交。
- 推送用 Cloud Agent 自带的凭据（`git push -u origin cursor/github-actions-ci-d6e2`）；创建、更新 PR 用 PR 管理工具，不用 `gh` 写操作；输出中的令牌先打码。
- 新增或修改的 `.sh` 跑 `shellcheck -x -P SCRIPTDIR`，要求 0 条；workflow 跑 actionlint。
- 提交、推送前运行文档检查：Task 3 之前用 `python3 /tmp/lvgl/tests/tools/docs/check-headings.py`，之后用入库的 `tests/tools/docs/check-headings.py`；不为 0 处时中止。

## 2. Review Focus

- **与 lvgl 仓的差异是否只有预期部分**：`build-image` 只改镜像名、signer workflow、run 摘要、注释与 dev 分支守卫（spec Q19），并发设置只对 `pull_request` 取消旧 run（spec Q21）；`check-headings.py` 只改 `QA_EXEMPT` 与文档路径（附录 A.4 的 diff）。
- **兼容性检查的上限与出处一致**：`env` 中的四个上限与 spec §2.6 的表相同，注释写明取自 Ubuntu 22.04 `jammy-updates` 的 armhf 包。
- **PR 上不构建、不覆盖正式 tag**：最终形态下 `pull_request` 只复用 dev 签名镜像；Q2 的临时提交不进最终历史。
- **门禁不会被跳过**：每行 5 个例程都过门禁，某个例程失败时 job 失败；门禁的反例本地验证过（spec §4 V3）。
- **根 `AGENTS.md` 的压缩不删信息**：逐句对照改前改后。

## 3. 提交

提交按主题拆分，以 `git log` 为准（spec §5）。开发期 spec 与 plan 最先推送，各轮 grilling 后的修订以 fixup、`amend!` 并入对应提交；V4 用的两个临时提交（PR 上构建临时镜像、改坏 spec 章号）叠在实现提交之上，Task 10 用 `git rebase -i` 删去，最终历史中没有它们，运行记录见 §5.5。

## 4. Tasks

### Task 1：spec 与 plan，推送并开草稿 PR ✅

- [x] spec 与 plan 最先推送，开草稿 PR [yuangezhizao/luckfox_pico_rkmpi_example#2](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/pull/2)；两轮 grilling（spec §8 Q1–Q15）的答案、作者追问的解释与实测（spec §2.6、§2.7）写入 spec，本 plan 据此补全；实现后作者复查引出的 Q16–Q18 见 §6。

### Task 2：两份 Dockerfile 取消注释 `python3-markdown-it` ✅

- [x] 两份 Dockerfile 各改 3 行（分组总述、文档检查组说明、包行），版本与 FROM 中的 python3 以 `dpkg-query` 实测为准（§5.2）；`docker build --check` 无告警。

### Task 3：文档检查脚本入库 ✅

- [x] `check-headings.py` 与 lvgl 仓只差出处与 `QA_EXEMPT`；`check-samples.sh` 的 `mk` 样例改取本仓 PR #1 的 spec 与 plan，没有对应原文的规则改为内联样例，共 115 个；对照见 §5.4、附录 A.4。

### Task 4：主 workflow 的 `build-image`、`build-example` ✅

- [x] `build-image` 与 lvgl 仓的差异只有镜像名、signer workflow、run 摘要与出处注释（附录 A.4），合并前按 Codex 审查又加了 dev 分支守卫与只对 `pull_request` 生效的自动取消（§6）；`build-example` 每行编译 `example/` 下的全部例程，兼容性检查的四个上限在 `env` 中（spec §3.4、Q15）；`run:` 只用环境变量，可在 CI 镜像中原样执行（附录 A.6）。

### Task 5：`shellcheck`、`doc-check` ✅

- [x] ShellCheck 排除上游 `build.sh`（spec Q9、Q13）；`doc-check` 在 `schedule` 中也运行（spec §3.2）。

### Task 6：体积检查 workflow ✅

- [x] 照搬 lvgl 仓，只改预算出处与超限提示的链接。

### Task 7：四份 `AGENTS.md` ✅

- [x] 新增 `.github/AGENTS.md`；根 `AGENTS.md` 的「测试与检查」按 lvgl 仓 PR #17 的原则改写，去向表加一行，删去与本文件其他位置重复的内容（spec §3.9、Q16），改后 8,182 字节；`.cursor/AGENTS.md`、`docs/superpowers/AGENTS.md` 改为指向 CI 与入库的脚本。

### Task 8：本地 V1–V3 ✅

- [x] 结果见 §5.1–§5.3，命令见附录 A.5、A.6。

### Task 9：开发期 CI（V4）✅

- [x] 本 fork 的 Actions 已启用，推送后两个 workflow 均为 `active`；两个临时提交的 run 见 §5.5。

### Task 10：回填 PR #1 plan、整理提交、V5、精简 plan ✅

- [x] PR #1 plan Task 8 中已核对的项勾选（与 Q18 对 PR #1 文档的修改合为单独的一个提交），加一段指向本 spec §2.5（spec Q12）；V6 在作者执行 tailnet SSH 后勾选（Task 11）。
- [x] `git rebase -i`：删去两个临时提交，并入 fixup，spec 与 plan 的提交移到末尾；逐个提交检出后 actionlint、ShellCheck、文档检查与样例均通过，根 `AGENTS.md` 不超过 8,192 字节。
- [x] `--force-with-lease` 推送；V5 见 §5.6；精简本 plan，更新 PR 正文。
- [x] 合并前准备（作者 11-19 确认，不做三路并行审查）：用临时提交在 runner 上复跑最终的 workflow，链接与引用检查，spec 状态行改为「已落地」（§5.7）；PR 改为 ready，不合并。

### Task 11：作者执行的项

- [x] tailnet SSH：作者从本地 Mac mini `ssh ubuntu@cursor-agent-for-luckfox-pico-rkmpi-example-d224f711` 登录成功，主机指纹与本 Agent 相同；`ssh root@…` 被拒，报 `Permission denied (publickey)`，sshd 生效配置为 `permitrootlogin no`（spec §2.5）。
- [x] 勾选 PR #1 plan Task 8 的 V6。
- [ ] 合并后（V6）：`push(dev)` 的 `build-image` 走可信首建并 attest；可选在 GHCR 包页面删除临时 tag `pr-2-703b5d91b21228250ee6dfc969cb4b2b`；下一个 PR 的 CI 走 dest-lock 复用，结果记入下一个 PR。

## 5. 验证证据

### 5.1 V1 静态检查（2026-11-18，附录 A.5）

- actionlint 1.7.12（调用 ShellCheck 0.9.0 检查各段 `run:`）对两个 workflow 0 条；ShellCheck 对 `.cursor/install.sh`、`.cursor/start.sh`、`tests/tools/docs/check-samples.sh` 0 条。
- `check-headings.py` 检查 4 个文件 0 处；`check-samples.sh` 样例 115 个，不符合 0 个（从其他目录运行结果相同）；根 `AGENTS.md` 8,188 字节（Q16 改写后为 8,182 字节，见 §6）。

### 5.2 V2 镜像（附录 A.6）

- Docker 29.1.3（fuse-overlayfs）；`docker build --check` 两份均无告警；镜像 956 MB（`.cursor/Dockerfile`）、1.08 GB（`Dockerfile.luckfox_pico`）。
- `dpkg-query`：FROM `ubuntu:24.04` 中没有 python3，安装后 `python3-markdown-it` 3.0.0-2、python3 3.12.3；FROM `luckfox_pico:1.0` 已有 python3 3.10.6，安装时带上 `python3-attr` 21.2.0，`python3-markdown-it` 1.1.0-2。两份镜像中文档检查 0 处、样例 115 个全部符合。
- 在 `.cursor/Dockerfile` 的镜像中以 `--user 0` 原样执行 `build-example` 的 4 段 `run:`（仓库为 `git archive HEAD` 的副本）：两行各 5 个例程编译成功、通过兼容性检查，`SHA256SUMS` 各 10 行（5 个程序、5 个模型），编译输出 `warning:` 0 条；uClibc 工具链为 `yuangezhizao/luckfox-pico` `665c7f1`。本仓工作区无改动。

### 5.3 V3 兼容性检查的反例（附录 A.6）

| 反例 | 结果 |
|---|---|
| glibc 行检查 uClibc 产物 | 退出 1：每个例程报 interpreter 为 `/lib/ld-uClibc.so.0`、`NEEDED` 缺 `libc.so.6` 且有 `libc.so.0` |
| uClibc 行检查 glibc 产物 | 退出 1，三项相反 |
| 调用 `sscanf`、`strtol` 的程序替换一个例程 | 退出 1：`要求 GLIBC_2.38，超过板上的 GLIBC_2.35；引用它的符号：__isoc23_sscanf@GLIBC_2.38 __isoc23_strtol@GLIBC_2.38`，汇总为 1/5 个例程未通过 |
| 调用 `std::chrono::current_zone()` 的程序替换一个例程 | 退出 1：`要求 GLIBCXX_3.4.31，超过板上的 GLIBCXX_3.4.30`，列出 `_ZNSt6chrono12current_zoneEv@GLIBCXX_3.4.31`；该程序在 qemu 中用 22.04 的 libstdc++ 运行同样报 version not found |
| 删去一个例程的可执行文件 | 退出 1：`产物不存在或为空` |
| 正常产物 | 退出 0，5 个例程通过 |

### 5.4 检查脚本与 lvgl 仓的对照（Task 3，附录 A.4）

- `check-headings.py` 与 lvgl 仓只差两处：文档字符串中的出处改为完整链接，`QA_EXEMPT` 改为空字典。
- `check-samples.sh` 改写的 16 个样例与原样例报出同一类问题：`q16a`–`q16d` 报 `#### 应为 2.2.x`，`q17a`–`q17f` 报 Task 号错误，`q19a`、`q19b` 报引用了不存在的 Task，`q24a`–`q24d` 报章标题的 Task 范围不符；新增的 `qa8` 报 QA 编号错误。
- lvgl 仓的 `q16a`–`q16d` 改写的 spec 在该仓的 `QA_EXEMPT` 中，复制为 `bad-q16*.md` 后不再豁免，每个还报 7 处 QA 编号错误；四级标题规则失效时这 4 个样例仍会失败，不能单独证明该规则。本仓的内联样例只含四级标题问题，没有这个缺口。对照实验：在仓库外的副本中把检查器的 `report(f, i, f"#### 应为 {label}", line)` 改为 `pass`，lvgl 仓的样例 114 个、不符合 0 个，本仓的样例 115 个、不符合 4 个（`bad-q16a.md`–`bad-q16d.md`）。已在 [yuangezhizao/luckfox_pico_rkmpi_example#2](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/pull/2) 的评论中告知，随 lvgl 仓的下一个 PR 处理。

### 5.5 V4 开发期 CI（2026-11-18，北京时间）

- 临时提交 `bf2490f`：[run 44323881172](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/actions/runs/44323881172) 22:24–22:26 全部 5 个 job 通过。`build-image` 推送临时 tag `pr-2-703b5d91b21228250ee6dfc969cb4b2b`（digest `sha256:8a52f1a6…`，不 attest），约 1.5 分钟；两行编译各约 40 秒，日志为 5 个例程编译成功、通过兼容性检查，uClibc 工具链 `665c7f1`；`📝 文档检查` 用 markdown-it-py 3.0.0，4 个文件 0 处、样例 115 个全部符合；ShellCheck 检查 3 个脚本。同一推送的 [run 44323881177](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/actions/runs/44323881177) 体积检查通过。
- 两个 artifact 各 11 个文件（5 个部署目录与 `SHA256SUMS`），`sha256sum -c` 全部通过。
- 改坏 spec 章号的临时提交 `188cbab`：[run 44324439390](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/actions/runs/44324439390) 中只有 `📝 文档检查` 失败，报 2 处章号不连续，退出码 1。

### 5.6 V5 最终形态 CI

- rebase 后的 `1707eba`：[run 44325673972](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/actions/runs/44325673972) 中 `build-image` 报 `pull_request：只复用 dev 签名镜像` 与 `ghcr.io/yuangezhizao/luckfox-rkmpi-example-ci:703b5d91b21228250ee6dfc969cb4b2b: not found` 后失败（临时 tag 不被复用路径使用），两行编译与 `📝 文档检查` 被跳过，`🐚 ShellCheck` 通过；[run 44325673774](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/actions/runs/44325673774) 体积检查通过。与 spec §2.3 的预期相同。

### 5.7 合并前检查（2026-11-19，北京时间）

- **复跑最终 workflow**：V4 之后 workflow 只改了兼容性检查的步骤名与注释（spec Q17），步骤名的表达式此前只经过 actionlint。在最终提交上叠加 V4 的临时提交：[run 44454229850](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/actions/runs/44454229850) 11:50–11:53 全部 5 个 job 通过，两行的步骤名分别为「✅ 兼容性检查（含版本上限）」与「✅ 兼容性检查（uClibc 不查版本上限）」，两个 artifact 各 11 个文件、`sha256sum -c` 通过；随后删去临时提交。
- **链接**：spec、plan、四份 `AGENTS.md`、Dockerfile、workflow、检查脚本，以及本 PR 与 PR #1 的正文和评论中共 69 个链接（附录 A.9）：65 个返回 200；Cursor 页脚的 Agent 链接需要登录（403）；Luckfox 论坛拒绝脚本访问（curl 与 WebFetch 均为 403），页面未能核实，只用于说明官方不再维护 Ubuntu 镜像，不影响结论；PR #1 正文中的 spec、plan 链接指向已删除的分支 `pi/cloudagent-env`（404）。作者要求两个 PR 正文中的 spec、plan 链接统一指向 `dev` 分支：PR #1 的文件已在 `dev` 上；本 PR 的两份文件合并后才在 `dev` 上，合并前这两个链接为 404。
- **引用**：spec 与 plan 中的 §、附录、Task 与 Q 引用都指向存在的位置；脚本列出的都是指向 lvgl 仓、luckfox-pico 仓文档的引用，或模板与说明中的 `§X`（附录 A.9）。

### 5.8 dev 分支守卫（spec Q19，附录 A.10）

用桩代替 `docker`、`gh`，原样运行 `build-image` 中的构建脚本（`GITHUB_REF`、事件名与 tag 是否存在按场景设置）：

| 场景 | 结果 |
|---|---|
| 非 `dev` 分支手动触发，勾选 `force_rebuild` | 退出 1，`❌ force_rebuild：正式镜像只在 dev 上构建并 attest`，没有调用构建 |
| 非 `dev` 分支手动触发，tag 不存在 | 退出 1，`❌ 首建：…`，没有调用构建 |
| 非 `dev` 分支手动触发，tag 已存在 | 退出 0，验证 provenance 后复用 |
| `dev` 上手动 `force_rebuild`、`schedule` | 退出 0，强制重建 |
| `dev` 上 `push`，tag 不存在 | 退出 0，首建 |
| `pull_request`，tag 不存在 | 退出 1，`❌ 无已存在镜像`，与改前相同 |

`pull_request` 路径与开发期临时提交所走的路径都在守卫之前，不受影响；actionlint 0 条。

## 6. 与计划的偏离

- PR 改为 ready 后 Codex 审查（P1）指出非 `dev` 分支手动触发会用 feature 分支签名覆盖正式 tag（spec Q19）：`build-image` 的强制重建与首建路径加 dev 分支守卫，`.github/AGENTS.md` 同步，并入对应的提交；验证见 §5.8，并在评论线程中回复。随后的 Codex 安全审查（P2）建议把 uClibc 工具链固定到 commit SHA，作者决定不固定（spec Q20），在线程中回复理由，不标记解决；代码不变。之后 Codex 代码审查（P2）指出推送正式镜像后、签名前可能被自动取消（spec Q21）：`cancel-in-progress` 改为只对 `pull_request` 生效，`.github/AGENTS.md` 补上缺少签名时的恢复方法，并入对应的提交；actionlint 0 条，并发设置的效果只能在 GitHub 上体现，按官方文档对表达式的支持判断，不另做运行验证。Codex 安全审查（P2）另建议把 `docker/login-action` 固定到 commit SHA，作者决定本 PR 不改、与两仓一起处理（spec Q22），在线程中回复以后的改法，不标记解决。
- 作者 11-19 要求对照 lvgl 仓 PR #17 的原则复查四份 `AGENTS.md`（spec Q16）：根文件「测试与检查」补上所有任务都用得到的 CI 常识、文档检查改为直接写命令，删去与去向表重复的指针，`.github/AGENTS.md` 删去与根文件重复的触发范围；修改并入对应的提交。
- 作者 11-19 问 CI 应使用哪个版本的交叉工具链（spec Q17）：在 `Dockerfile.luckfox_pico` 的镜像中实测 jammy 工具链（gcc 11.4、glibc 2.35）可编译全部 glibc 例程，附录 A.6 的步骤脚本把 `rkmpi-ci:active` 换成 `rkmpi-ci:alt` 即可复现。作者选择维持 24.04；兼容性检查的 `env` 注释写明上限只用于 glibc 行，步骤名按行显示是否查版本上限，`.github/AGENTS.md` 的引用同步，均并入对应的提交；actionlint 0 条，两行的检查步骤重跑通过。
- 作者 11-19 认为提交标题已在 git 中，文档不必逐条列出（spec Q18）：本 PR 与 PR #1 的 spec、plan 删去提交清单与「提交 N」指针，两个 PR 的正文同样精简；作者随后要求 PR #1 文档的修改与 Q12 的回填合为一个提交，两者都只改历史文档。
- 作者 11-19 要求把开发中用过、以后可参考的临时脚本写进 plan（Cloud Agent 重启后 `/tmp` 不保留）：附录补上 Cloud Agent 中临时安装 Docker、`tz.cpp` 反例、22.04 工具链实测（A.3、A.6），tailnet SSH 的本机核对（A.2），CI 结果核对（A.7）与逐个提交检查（A.8）；只用一次的脚本（如对比 PR 正文快照）不保存。
- Docker 安装时 `fuse-overlayfs` 的 `/etc/fuse.conf` 提问使 apt 停住，按 `.cursor/AGENTS.md`「核对镜像内容」用 `dpkg --force-confold --configure -a` 处理，不影响结果。

## 附录 A

命令都在本仓根目录运行。需要：lvgl 仓克隆到 `/tmp/lvgl`（`docs/superpowers/AGENTS.md`「检查」一条的命令）；uClibc 工具链稀疏检出到 `/tmp/luckfox-pico`（根 `AGENTS.md`「项目与构建」）。

### A.1 现状实测（spec §2.1）

```bash
# 两种 libc × 5 个例程：仓库外构建目录 + DESTDIR，工作区不变；打印门禁所需的 ABI 信息
git status --short > /tmp/st-before
for libc in glibc uclibc; do
  for ex in luckfox_pico_rtsp_opencv luckfox_pico_rtsp_opencv_capture luckfox_pico_rtsp_retinaface luckfox_pico_rtsp_retinaface_osd luckfox_pico_rtsp_yolov5; do
    b=/tmp/build-$libc-$ex; s=/tmp/stage-$libc-$ex; rm -rf "$b" "$s"
    if [ $libc = glibc ]; then e="GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-"; else e="LUCKFOX_SDK_PATH=/tmp/luckfox-pico"; fi
    env $e cmake -S . -B "$b" -DEXAMPLE_DIR=example/$ex -DEXAMPLE_NAME=$ex -DLIBC_TYPE=$libc > "$b.log" 2>&1 \
      && env $e make -C "$b" -j"$(nproc)" >> "$b.log" 2>&1 && make -C "$b" install DESTDIR="$s" >> "$b.log" 2>&1
    rc=$?; bin=$(find "$s" -name $ex -type f)
    echo "$libc $ex rc=$rc warn=$(grep -c 'warning:' "$b.log") $(readelf -d "$bin" | grep -oE 'libc\.so\.[0-9]') $(file -b "$bin" | grep -o 'interpreter [^,]*')"
  done
done
git status --short | diff /tmp/st-before - && echo worktree-unchanged
# glibc 产物所需的最高符号版本（本次 GLIBC_2.34）
arm-linux-gnueabihf-objdump -T "$(find /tmp/stage-glibc-luckfox_pico_rtsp_yolov5 -name luckfox_pico_rtsp_yolov5 -type f)" | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1
# ShellCheck：.cursor 两个脚本 0 条，build.sh 6 条
shellcheck -x -P SCRIPTDIR .cursor/install.sh .cursor/start.sh && echo cursor-0
shellcheck -x -P SCRIPTDIR build.sh | grep -oE 'SC[0-9]+ \([a-z]+\)' | sort | uniq -c
# 文档检查（lvgl 仓的脚本）与根 AGENTS.md 字节数
python3 /tmp/lvgl/tests/tools/docs/check-headings.py; wc -c AGENTS.md
# 各 action 的最新 release
for r in actions/checkout docker/login-action actions/attest actions/upload-artifact; do echo "$r $(gh api repos/$r/releases/latest -q .tag_name)"; done
```

### A.2 PR #1 的 V5、V6（spec §2.5）

```bash
whoami; id
stat -c '%U:%G %a %n' ~/.cursor ~/.cursor/skills ~/.cursor/skills/grilling ~/.cursor/skills/grilling/SKILL.md
sha256sum ~/.cursor/skills/grilling/SKILL.md
tailscale status | head -1; tailscale serve status; ss -ltn | grep ':22 '
dpkg-query -W openssh-server
# tailnet SSH：作者在另一台机器上 ssh ubuntu@<主机名> 后，在本机核对对方看到的 IP、主机指纹，以及 sshd 的生效配置与登录记录
tailscale ip -4; ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication) '
sudo last -n 3
```

saved environment 与 Build 用 Cursor Cloud 的 `environment-info`、`list-environment-builds`、`environment-build-logs` 查询（日志末尾的 `[INSTALL] Exit code:`）。`start.sh` 以 detached 方式运行，输出不进入平台的 start 日志（`batch-fetch-details` 的 `includeStartLogs` 只有 span 记录），以 tailscaled、Serve 与 sshd 的进程和端口判断。

### A.3 兼容性检查的依据（spec §2.6；以后调整版本上限或做 qemu 冒烟时复用）

```bash
# 板端上限：取 Ubuntu 22.04 jammy-updates 当前的 armhf 包，读各库提供的最高版本
R=$PWD; mkdir -p /tmp/abi && cd /tmp/abi && rm -rf root
P=http://ports.ubuntu.com/ubuntu-ports
curl -sSfL $P/dists/jammy-updates/main/binary-armhf/Packages.gz | zcat \
  | awk '/^Package: (libc6|libstdc\+\+6|libgcc-s1)$/{p=1} p&&/^Filename:/{print $2; p=0}' \
  | while read -r f; do curl -sSfLO "$P/$f" && dpkg-deb -x "$(basename "$f")" root; done
# top <文件> def|need：库提供的（Version definition）或文件要求的（Version needs）各前缀最高版本
top() { readelf -V "$1" | if [ "$2" = def ]; then awk '/Version definition/,/Version needs/'; else awk '/Version needs/,0'; fi \
  | grep -oE '(GLIBC|GLIBCXX|CXXABI|GCC)_[0-9.]+' | sort -uV | awk -F_ '{m[$1]=$0} END{for(k in m) printf "%s ", m[k]; print ""}'; }
for f in root/lib/arm-linux-gnueabihf/libc.so.6 root/usr/lib/arm-linux-gnueabihf/libstdc++.so.6 root/lib/arm-linux-gnueabihf/libgcc_s.so.1; do
  echo "$(basename "$f"): $(top "$(readlink -f "$f")" def)"
done
# 24.04 交叉工具链的 libstdc++ 能提供的最高版本
top "$(readlink -f /usr/arm-linux-gnueabihf/lib/libstdc++.so.6)" def
# 产物（附录 A.1 的 /tmp/stage-*）与入库 install/ 所需的最高版本
for b in $(find /tmp/stage-* -type f -name 'luckfox_pico_rtsp_*') $(git -C "$R" ls-files install | grep -v /model/ | sed "s#^#$R/#"); do
  echo "$b: $(top "$b" need)"
done
# 超过上限即不启动：sscanf、strtol 在 glibc 2.38+ 的头文件下被重定向到 __isoc23_*
printf '#include <cstdio>\n#include <cstdlib>\nint main(int c,char**v){int x=0;std::sscanf(v[0],"%%d",&x);return (int)std::strtol(v[c-1],nullptr,10)+x;}\n' > t.cpp
printf '#include <cstdio>\nint main(){std::puts("hello");}\n' > t2.cpp
# libstdc++ 的反例：std::chrono::current_zone() 要求 GLIBCXX_3.4.31（gcc 13 起）
printf '#include <chrono>\n#include <cstdio>\nint main(){auto z=std::chrono::current_zone();std::puts(z->name().data());}\n' > tz.cpp
arm-linux-gnueabihf-g++ -O2 t.cpp -o t && arm-linux-gnueabihf-g++ -O2 t2.cpp -o t2 && arm-linux-gnueabihf-g++ -std=c++20 -O2 tz.cpp -o tz
arm-linux-gnueabihf-nm -D t | grep isoc23
sudo apt-get install -y -qq qemu-user
qemu-arm -L root ./t 1; echo "rc=$?"    # version `GLIBC_2.38' not found，rc=1
qemu-arm -L root ./t2; echo "rc=$?"     # hello，rc=0
qemu-arm -L root ./tz; echo "rc=$?"     # version `GLIBCXX_3.4.31' not found，rc=1
cd "$R"
```

### A.4 与 lvgl 仓的差异（以后同步 lvgl 仓的修改时复用）

```bash
L=/tmp/lvgl   # lvgl 仓 dev 的浅克隆
diff $L/tests/tools/docs/check-headings.py tests/tools/docs/check-headings.py
diff $L/tests/tools/docs/check-samples.sh tests/tools/docs/check-samples.sh
diff $L/.github/workflows/agents-md-size.yml .github/workflows/agents-md-size.yml
# build-image：按名称对齐后比较，差异应只有镜像名、signer workflow、run 摘要、注释与 require_dev 守卫（spec Q19）；workflow 顶部的 concurrency 另有 Q21 的差异
sed -n '/^  build-image:/,/^  build-demo:/p' $L/.github/workflows/build-luckfox-lvgl-demo.yml | sed 's/lvgl-demo/X/g' > /tmp/bi-lvgl
sed -n '/^  build-image:/,/^  build-example:/p' .github/workflows/build-luckfox-rkmpi-example.yml | sed 's/rkmpi-example/X/g' > /tmp/bi-rkmpi
diff /tmp/bi-lvgl /tmp/bi-rkmpi
# 改写的样例报出的问题：把 check-samples.sh 生成的样例留下后逐个运行检查器
cat > /tmp/cmp-samples.sh <<'EOF'
src=$1; dir=$(cd "$(dirname "$src")" && pwd); keep=$(mktemp -d)
sed -e 's#^D=$(cd "$(dirname "$0")" \&\& pwd)#D='"$dir"'#' \
    -e 's#^n=0; miss=0$#cp bad-q16* bad-q17* bad-q19* bad-q24* bad-qa8* '"$keep"'/ 2>/dev/null; exit 0#' "$src" > /tmp/cs-run.sh
bash /tmp/cs-run.sh
for f in "$keep"/*.md; do printf '%-9s %s\n' "$(basename "$f" .md)" "$(python3 "$dir/check-headings.py" "$f" | grep -v '^检查' | sed "s#$keep/[^:]*:##" | head -2 | tr '\n' ' ' | cut -c1-160)"; done
EOF
bash /tmp/cmp-samples.sh $L/tests/tools/docs/check-samples.sh; bash /tmp/cmp-samples.sh tests/tools/docs/check-samples.sh
# 关掉四级标题编号检查的变体：样例应报出不符合；lvgl 仓的 q16a–q16d 因 QA 编号错误照样退出 1，报不出
for src in $L "$PWD"; do
  d=$(mktemp -d); cp -r "$src"/tests "$src"/docs "$d"/
  sed -i 's/^\( *\)report(f, i, f"#### 应为 {label}", line)$/\1pass/' "$d"/tests/tools/docs/check-headings.py
  bash "$d"/tests/tools/docs/check-samples.sh | tail -5; rm -rf "$d"
done
```

### A.5 V1 静态检查

```bash
V=$(gh api repos/rhysd/actionlint/releases/latest -q .tag_name)
curl -sSfL "https://github.com/rhysd/actionlint/releases/download/${V}/actionlint_${V#v}_linux_amd64.tar.gz" | tar xz -C /tmp actionlint
/tmp/actionlint && echo actionlint-0
mapfile -d '' files < <(git ls-files -z '*.sh' | grep -zvx 'build\.sh'); shellcheck -x -P SCRIPTDIR "${files[@]}" && echo shellcheck-0
python3 tests/tools/docs/check-headings.py; bash tests/tools/docs/check-samples.sh | tail -1; wc -c AGENTS.md
```

### A.6 V2、V3：在 CI 镜像中原样执行 workflow 的步骤（以后改 workflow 时复用）

```bash
# Cloud Agent 中临时安装 Docker（.cursor/AGENTS.md「核对镜像内容」，不写入 Dockerfile）：
# fuse-overlayfs 会提问是否覆盖 /etc/fuse.conf，用 --force-confold 保留现有文件；VM 内用 fuse-overlayfs 存储驱动与 legacy iptables
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -qq -o Dpkg::Options::=--force-confold docker.io docker-buildx fuse-overlayfs
sudo mkdir -p /etc/docker && echo '{"storage-driver": "fuse-overlayfs"}' | sudo tee /etc/docker/daemon.json
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy; sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
tmux -f /exec-daemon/tmux.portal.conf new-session -d -s dockerd 'sudo dockerd > /tmp/dockerd.log 2>&1'
until sudo docker info >/dev/null 2>&1; do sleep 2; done; sudo docker info | grep -E 'Server Version|Storage Driver'
# 两份镜像
for f in Dockerfile Dockerfile.luckfox_pico; do sudo docker build --check -f .cursor/$f .cursor; done
sudo docker build -f .cursor/Dockerfile -t rkmpi-ci:active .cursor
sudo docker build -f .cursor/Dockerfile.luckfox_pico -t rkmpi-ci:alt .cursor
for img in rkmpi-ci:active rkmpi-ci:alt; do
  sudo docker run --rm -v "$PWD":/repo:ro -w /repo "$img" bash -c \
    'dpkg-query -W python3 python3-markdown-it; python3 tests/tools/docs/check-headings.py; bash tests/tools/docs/check-samples.sh | tail -1'
done
# 把 build-example 每段 run: 连同 env 写成脚本（需要 python3-yaml）
cat > /tmp/extract-steps.py <<'EOF'
import os, sys, yaml
wf, job, out = sys.argv[1:]
j = yaml.safe_load(open(wf, encoding="utf-8"))["jobs"][job]
os.makedirs(out, exist_ok=True)
for i, s in enumerate(x for x in j["steps"] if "run" in x):
    env = {**j.get("env", {}), **s.get("env", {})}
    with open(f"{out}/{i:02d}.sh", "w", encoding="utf-8") as fh:
        fh.write(f"# {s['name']}\n")
        fh.writelines(f"export {k}='{v}'\n" for k, v in env.items() if "${{" not in str(v))
        fh.write(s["run"])
EOF
python3 /tmp/extract-steps.py .github/workflows/build-luckfox-rkmpi-example.yml build-example /tmp/ci-steps
# V2：仓库副本模拟 CI 检出，以 --user 0 在活动镜像中依次执行；uClibc 行挂载 /tmp/luckfox-pico
for l in glibc uclibc; do
  ws=/tmp/v2-ws-$l; sudo rm -rf $ws; mkdir -p $ws; git archive HEAD | tar -x -C $ws
  extra=""; [ $l = uclibc ] && extra="-v /tmp/luckfox-pico:/ws/luckfox-pico:ro"
  sudo docker run --rm --user 0 -v $ws:/ws $extra -v /tmp/ci-steps:/steps:ro -w /ws -e LIBC=$l -e GITHUB_WORKSPACE=/ws rkmpi-ci:active \
    bash -c 'for s in /steps/*.sh; do bash "$s" || exit 1; done' > /tmp/v2-$l.log 2>&1; echo "$l rc=$?"
done
# V3：把反例放进部署目录后只运行兼容性检查（02.sh）；t、tz 为附录 A.3 的小程序与调用 std::chrono::current_zone() 的程序（-std=c++20）
v3() {  # $1=名称 $2=检查用的 LIBC $3=产物来源目录 $4=可选：替换 opencv 例程的二进制
  local ws=/tmp/v3-$1; rm -rf "$ws"; mkdir -p "$ws/install/$2"; cp -r example "$ws/"; cp -r "$3"/. "$ws/install/$2/"
  [ -n "${4:-}" ] && cp "$4" "$ws/install/$2/luckfox_pico_rtsp_opencv_demo/luckfox_pico_rtsp_opencv"
  (cd "$ws" && LIBC=$2 bash /tmp/ci-steps/02.sh > out.log 2>&1); echo "== $1 rc=$?"; grep -E '::error|❌|✅' "$ws/out.log" | sort -u | head -6
}
v3 glibc-on-uclibc glibc /tmp/v2-ws-uclibc/install/uclibc; v3 uclibc-on-glibc uclibc /tmp/v2-ws-glibc/install/glibc
v3 glibc-2.38 glibc /tmp/v2-ws-glibc/install/glibc /tmp/abi/t; v3 glibcxx-3.4.31 glibc /tmp/v2-ws-glibc/install/glibc /tmp/abi/tz
# spec Q17 的备选：在基于 22.04 的镜像中编译 glibc 例程（gcc 11.4、glibc 2.35），并看两个反例在 jammy 工具链下的表现
ws=/tmp/alt-ws-glibc; sudo rm -rf $ws; mkdir -p $ws; git archive HEAD | tar -x -C $ws
sudo docker run --rm --user 0 -v $ws:/ws -v /tmp/ci-steps:/steps:ro -w /ws -e LIBC=glibc -e GITHUB_WORKSPACE=/ws rkmpi-ci:alt \
  bash -c 'arm-linux-gnueabihf-gcc --version | head -1; bash /steps/00.sh && bash /steps/02.sh' 2>&1 | grep -E 'gcc|✅|❌'
sudo docker run --rm -v /tmp/abi:/abi:ro rkmpi-ci:alt bash -c 'cd /tmp
  arm-linux-gnueabihf-g++ -O2 /abi/t.cpp -o t && readelf -V t | grep -oE "GLIBC_[0-9.]+" | sort -uV | tail -1   # GLIBC_2.34
  arm-linux-gnueabihf-g++ -std=c++20 -O2 /abi/tz.cpp -o tz 2>&1 | grep -m1 error'                               # current_zone 不存在，编译失败
```

### A.7 CI 结果核对（V4、V5；以后看 PR 上的 CI 时复用）

```bash
R=yuangezhizao/luckfox_pico_rkmpi_example
gh api repos/$R/actions/workflows -q '.workflows[] | "\(.name) \(.state)"'    # fork 中 workflow 是否启用
id=$(gh run list -R $R --workflow build-luckfox-rkmpi-example.yml --limit 1 --json databaseId -q '.[0].databaseId')
until gh run view $id -R $R --json status -q .status | grep -qx completed; do sleep 10; done
gh run view $id -R $R --json conclusion,headSha,jobs -q '.conclusion, .headSha[0:7], (.jobs[] | "\(.name)\t\(.conclusion)")'
gh run view $id -R $R --log-failed | grep -vF '[36;1m' | tail -20     # 失败步骤的输出，去掉回显的脚本行
gh run view $id -R $R --log | grep -E '镜像 digest|🚧|✅ 5 个|luckfox-pico SHA|markdown-it-py|样例'
rm -rf /tmp/art && mkdir /tmp/art && (cd /tmp/art && gh run download $id -R $R \
  && for d in */; do echo "$d $(find "$d" -type f | wc -l) files"; (cd "$d" && sha256sum -c --quiet SHA256SUMS && echo sha-ok); done)
```

### A.8 整理提交后的逐个提交检查（以后的 PR 可复用）

提交整理（fixup、`amend!`、autosquash、把 spec 与 plan 移到最后）沿用 PR #1 plan 附录 A.7，本 PR 的差异：推送用 Cloud Agent 的凭据（`git push --force-with-lease=<分支>:<远端旧提交> origin <分支>`）；钩子允许 noreply 尾注，不再用 `core.hooksPath=/dev/null`；`amend!` 提交可用 `GIT_EDITOR="cp <说明文件>" git commit --fixup=amend:<提交>` 生成。逐个提交检查加上 actionlint、ShellCheck 与样例：

```bash
for c in $(git rev-list --reverse origin/dev..HEAD); do
  d=$(mktemp -d); git archive $c | tar -x -C $d
  ( cd $d
    al=-; [ -d .github/workflows ] && { /tmp/actionlint .github/workflows/*.yml >/dev/null 2>&1 && al=0 || al=FAIL; }   # archive 中没有 .git，要显式传文件
    mapfile -d '' sh < <(find . -name '*.sh' ! -path ./build.sh -print0); sc=$(shellcheck -x -P SCRIPTDIR "${sh[@]}" >/dev/null 2>&1 && echo 0 || echo FAIL)
    chk=tests/tools/docs/check-headings.py; [ -f $chk ] || chk=/tmp/lvgl/tests/tools/docs/check-headings.py
    printf '%s actionlint=%s shellcheck=%s AGENTS=%s | %s | %s\n' "$(git -C "$OLDPWD" log -1 --format=%h $c)" $al $sc "$(wc -c < AGENTS.md)" \
      "$(python3 $chk | tail -1)" "$( [ -f tests/tools/docs/check-samples.sh ] && bash tests/tools/docs/check-samples.sh | tail -1)" )
  rm -rf $d
done
for c in $(git rev-list origin/dev..HEAD); do echo "$(git log -1 --format=%h $c) sig=$(git cat-file -p $c | grep -c 'BEGIN SSH SIGNATURE')"; done
git log --format=%B origin/dev..HEAD | grep -i co-authored-by | sort | uniq -c   # 只应有 noreply 地址
```

### A.9 合并前的链接与引用检查（以后的 PR 可复用）

链接检查沿用 PR #1 plan 附录 A.7 的循环，文件范围加上 `.github/`、`tests/tools/docs/`，并收集本 PR 与 PR #1 的正文和评论；Cloud Agent 中用 `gh auth token`（`cursor` 账户）取令牌，不用 `-u yuangezhizao`：

```bash
R=yuangezhizao/luckfox_pico_rkmpi_example
{ cat docs/superpowers/specs/*.md docs/superpowers/plans/*.md AGENTS.md .cursor/AGENTS.md .github/AGENTS.md docs/superpowers/AGENTS.md \
      .cursor/Dockerfile* .github/workflows/*.yml tests/tools/docs/*
  gh pr view 2 -R $R --json body,comments -q '.body, (.comments[].body)'; gh pr view 1 -R $R --json body -q .body
} | grep -oE 'https?://[^ )`"<>|，。；、（）]+' | sed 's/[.,]$//' | grep -v '[$*{}]' | sort -u > /tmp/urls.txt
TOKEN=$(gh auth token)
while read -r u; do
  base=${u%%#*}
  case $base in https://github.com/*|https://raw.githubusercontent.com/*) h=(-H "Authorization: token $TOKEN") ;; *) h=(-A 'Mozilla/5.0') ;; esac
  echo "$(curl -s -o /dev/null -w '%{http_code}' -L --max-time 20 "${h[@]}" "$base") $u"
done < /tmp/urls.txt | grep -v '^200 '
```

引用检查列出 spec 与 plan 中目标不存在的 §、附录、Task 与 Q 引用，逐条判断是否指向别的文档：

```bash
cat > /tmp/refcheck.py <<'EOF'
import re, sys
def heads(p):
    s = open(p, encoding="utf-8").read()
    hs = {m.group(1) for m in re.finditer(r"^#{2,4} (?:附录 )?([0-9A-Z]+(?:\.[0-9]+)*)", s, re.M)}
    return s, hs, set(re.findall(r"^### Task (\d+)：", s, re.M)), set(re.findall(r"^\*\*Q(\d+)\. ", s, re.M))
spec, plan = sys.argv[1:]
S, P = heads(spec), heads(plan)
for name, (s, hs, ts, qs) in ((spec, S), (plan, P)):
    for m in re.finditer(r"(spec |plan )?(?:§|附录 )([0-9A-Z]+(?:\.[0-9]+)*)", s):
        tgt = hs if m.group(1) is None else (S[1] if m.group(1) == "spec " else P[1])
        if m.group(2) not in tgt: print(f"§{m.group(2)}: …{s[max(0, m.start() - 40):m.end() + 5]!r}")
    for m in re.finditer(r"(?<![\w#])Q(\d+)(?!\d)", s):
        if m.group(1) not in S[3]: print(f"Q{m.group(1)}: …{s[max(0, m.start() - 40):m.end()]!r}")
    if name == plan:
        for m in re.finditer(r"Task (\d+)", s):
            if m.group(1) not in P[2]: print(f"Task {m.group(1)}: …{s[max(0, m.start() - 40):m.end()]!r}")
EOF
python3 /tmp/refcheck.py docs/superpowers/specs/<本 PR 的 spec>.md docs/superpowers/plans/<本 PR 的 plan>.md
```

### A.10 用桩运行 `build-image` 的分流逻辑（spec Q19；以后改 `build-image` 时复用）

本地无法真正推送 GHCR，也无法从非默认分支触发 `workflow_dispatch`；用桩代替 `docker`、`gh`，按场景设置事件名、`GITHUB_REF` 与 tag 是否存在，检查退出码与是否调用了构建（需要 python3-yaml、jq）：

```bash
G=/tmp/guard-test; mkdir -p $G/bin
cat > $G/extract.py <<'EOF'
import sys, yaml
j = yaml.safe_load(open(sys.argv[1], encoding="utf-8"))["jobs"]["build-image"]
run = next(s for s in j["steps"] if s.get("id") == "build")["run"]
run = run.replace("${{ github.repository_owner }}", "yuangezhizao").replace("${{ github.repository }}", "yuangezhizao/luckfox_pico_rkmpi_example")
open(sys.argv[2], "w", encoding="utf-8").write("set -euo pipefail\n" + run)
EOF
python3 $G/extract.py .github/workflows/build-luckfox-rkmpi-example.yml $G/step.sh
cat > $G/bin/docker <<'EOF'
#!/bin/bash
# imagetools inspect 按 INSPECT_OK 决定 tag 是否存在；buildx build 记录调用并写 metadata
if [[ "$*" == *"imagetools inspect"* ]]; then
  [ "${INSPECT_OK:-0}" = 1 ] || { echo "ERROR: not found"; exit 1; }
  [[ "$*" == *"--format"* ]] && echo "\"sha256:$(printf 'a%.0s' {1..64})\"" || echo "Name: x"; exit 0
fi
if [[ "$*" == *"buildx build"* ]]; then
  echo BUILD >> "$CALLS"; m=$(sed -n 's/.*--metadata-file \([^ ]*\).*/\1/p' <<<"$*")
  echo "{\"containerimage.digest\":\"sha256:$(printf 'b%.0s' {1..64})\"}" > "$m"
fi
EOF
printf '#!/bin/bash\necho VERIFY >> "$CALLS"; echo %s\n' "'[{\"verificationResult\":{}}]'" > $G/bin/gh; chmod +x $G/bin/*
run() {  # $1=名称 $2=事件 $3=FORCE_REBUILD $4=GITHUB_REF $5=tag 是否存在（1/0）；在本仓根目录运行
  export CALLS=$G/calls-$1; : > "$CALLS"
  env PATH="$G/bin:$PATH" GITHUB_EVENT_NAME=$2 EVENT_NAME=$2 FORCE_REBUILD=$3 GITHUB_REF=$4 REF_NAME=${4##*/} INSPECT_OK=$5 \
      GITHUB_OUTPUT=$(mktemp) GITHUB_STEP_SUMMARY=$G/summary-$1 bash $G/step.sh > $G/out-$1.log 2>&1
  printf '%-26s rc=%s calls=[%s] %s\n' "$1" $? "$(tr '\n' ' ' < "$CALLS")" "$(grep -h -m1 -E '❌|镜像来源' $G/out-$1.log $G/summary-$1 2>/dev/null | head -1 | cut -c1-60)"
}
run dispatch-force-feature workflow_dispatch true  refs/heads/feature-x 1   # rc=1，无 BUILD
run dispatch-first-feature workflow_dispatch false refs/heads/feature-x 0   # rc=1，无 BUILD
run dispatch-reuse-feature workflow_dispatch false refs/heads/feature-x 1   # rc=0，VERIFY
run dispatch-force-dev     workflow_dispatch true  refs/heads/dev       1   # rc=0，BUILD
run schedule-dev           schedule          true  refs/heads/dev       1   # rc=0，BUILD
run push-first-dev         push              false refs/heads/dev       0   # rc=0，BUILD
run pr-no-image            pull_request      false refs/pull/2/merge    0   # rc=1，无 BUILD
```
