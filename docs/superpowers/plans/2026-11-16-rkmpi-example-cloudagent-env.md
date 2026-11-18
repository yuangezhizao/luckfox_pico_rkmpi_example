# Luckfox Pico RKMPI Example — Cloud Agent 环境实施计划

> **For agentic workers:** 作者同意本 PR 不必严格遵循 superpowers 的执行流程（根 `AGENTS.md`「优先级」）；按下面的 Task 顺序执行，复选框（`- [ ]`）用于跟踪。

**Goal:** 把 lvgl 仓 `dev`（`11497f3`）的 Cloud Agent 环境整套移植到本仓：两份 Dockerfile（apt 清单逐组对齐，本仓暂不需要的四组注释掉）、`environment.json`、`install.sh`、`start.sh`、三份 `AGENTS.md`；在当前 VM 的 Docker 中完成 V1–V4 验证。

**Architecture:** 推送 spec 与 plan、开草稿 PR（Task 1）；两份 Dockerfile（Task 2）；JSON 与脚本（Task 3）；三份 `AGENTS.md`（Task 4）；V1 静态检查（Task 5）；V2–V4 镜像、交叉编译与 `install.sh`（Task 6）；整理提交、精简 plan、PR 正文（Task 7）；作者在 Cursor 中做 V5、V6（Task 8）。

**Tech Stack:** Dockerfile、bash、ShellCheck 0.9.0、Docker 29.9.2（fuse-overlayfs）、CMake、`gcc-arm-linux-gnueabihf`、`arm-rockchip830-linux-uclibcgnueabihf`、`gh`。

**Spec:** [`docs/superpowers/specs/2026-11-16-rkmpi-example-cloudagent-env-design.md`](../specs/2026-11-16-rkmpi-example-cloudagent-env-design.md)

---

## 1. Global Constraints

- 分支 `pi/cloudagent-env`，起点 `origin/dev` `5517825`；PR 指向 fork `yuangezhizao/luckfox_pico_rkmpi_example` 的 `dev`，开发时为草稿，执行者不合并。
- 参考源一律取 lvgl 仓 `dev` `11497f3`，不读 lvgl 仓工作区（其中有作者未提交的 rtk、tmux 改动，spec §2.2）。本次在 lvgl 仓的检出 `/workspace` 中用 `git show origin/dev:<路径>`；附录命令改用固定提交的 raw 链接，不依赖本机目录。
- 不改 `CMakeLists.txt`、`build.sh`、`example/`、`lib/`、`include/`、`install/`；`.pi/` 不提交；`git add` 只写明确的路径。
- 验证构建不得改写工作区：容器内以只读方式挂载仓库，部署目录用 `DESTDIR`（spec §2.1）；每个 Task 结束时 `git status --short` 只应出现本 Task 的文件与 `.pi/`。
- 提交身份 `Cursor Agent <cursoragent@cursor.com>`，SSH 签名；环境注入了 `GIT_AUTHOR_*`、`GIT_COMMITTER_*`，所有创建或改写提交的 git 命令前加 `env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL`。本仓未配置 Cursor 钩子，提交不带 `Co-authored-by`；推送前 `git log --format=%B origin/dev..HEAD | grep -i co-authored-by` 应无输出。
- rebase（含 fixup、reword）与 `--amend` 保留 author time，不用 `--reset-author`、`--ignore-date`、`--date`、`--committer-date-is-author-date`。
- 提交信息按 lvgl 仓根 `AGENTS.md`「提交约定」：`<type>(<scope>): <emoji> <描述>`；正文与 lvgl 仓近期提交相同，写成「- 」开头的要点，每条一行（不硬折行），写原因、不写过程，PR 一律写完整链接或 `owner/repo#N`，不写裸 `#N`（GitHub 会解析到本 fork 或上游）；末尾 `Refs: docs/superpowers/specs/2026-11-16-rkmpi-example-cloudagent-env-design.md §X`；spec 与 plan 放在 PR 的最后一个提交。
- 推送：remote 中内嵌的 `cursor[bot]` 令牌对本仓没有写权限（403），改用作者账户：`git -c credential.helper= -c credential.helper='!f(){ test "$1" = get || exit 0; echo username=yuangezhizao; echo "password=$(gh auth token -u yuangezhizao)"; }; f' push https://yuangezhizao@github.com/yuangezhizao/luckfox_pico_rkmpi_example.git pi/cloudagent-env`（先 `env -u GIT_ASKPASS`）；改写历史时写明 `--force-with-lease=pi/cloudagent-env:<远端旧提交的完整哈希>`；`gh` 的登录会被环境刷新覆盖，失效时请作者重新 `gh auth login`；输出中的令牌先打码。
- 新增或修改 `.sh` 先跑 `shellcheck -x -P SCRIPTDIR`，要求 0 条。
- 提交、推送前运行 `python3 /tmp/lvgl/tests/tools/docs/check-headings.py`（先按 `docs/superpowers/AGENTS.md` 克隆 lvgl 仓到 `/tmp/lvgl`），不为 0 处时中止。
- Docker 已在本 VM 运行（`sudo docker`），镜像名用 `rkmpi-env:active`、`rkmpi-env:alt`，体积对照用 `lvgl-env:*`，不动已有的 `dc-tools:*`；Cloud Agent 中没有 Docker 时按 `.cursor/AGENTS.md`「核对镜像内容」临时安装。

## 2. Review Focus

- **注释掉的包行**：两份 Dockerfile 中被注释的四组是否只是去掉安装、不破坏续行；镜像中这些包确实不存在（可能被其他包作为依赖带入的要记录，spec §3.3）。
- **与 lvgl 仓的差异是否只有预期部分**：`environment.json`、`install.sh` 逐字相同；`start.sh` 只差 `repo_slug`；Dockerfile 的非注释差异只有四组包行（附录 A.1）。
- **`install.sh` 对 `~/.cursor` 的依赖**：平台预先建好 `755 ubuntu:ubuntu` 的 `~/.cursor`（lvgl 仓 PR #9 plan P4）；本地验证时先手动建立，记录不建时的表现。
- **`AGENTS.md` 的事实**：构建命令、`DESTDIR`、`install/` 入库、resolution order 都与实测、官方文档一致；不引用本仓不存在的文件。
- **镜像体积**：去掉四组后比 lvgl 仓镜像小多少，记录即可。

## 3. 提交

提交按改动对象拆分，以 `git log` 为准（spec §5）。开发期 spec 与 plan 最先推送供作者审阅，审阅与追问后的修订以 fixup、`amend!` 并入对应提交，Task 7 用 `git rebase -i --autosquash` 整理并把 spec 与 plan 的提交移到末尾（附录 A.7）；作者在审阅中提出的要求、跨仓评论与 Codex 审查见 Task 1、Task 7。

## 4. Tasks

### Task 1：spec 与 plan，推送并开草稿 PR ✅

落地：本 spec 与 plan；草稿 PR [yuangezhizao/luckfox_pico_rkmpi_example#1](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/pull/1)。

- [x] `check-headings.py` 0 处；spec 与 plan 推送，开草稿 PR（`-R` 指定 fork、`--base dev`）。
- [x] 按作者审阅修改：PR 正文中的 PR 与提交哈希改为完整链接；提交正文改为要点；For agentic workers 补出处（先为 spec 开头新增的「流程」，合并前改为根 `AGENTS.md`「优先级」，Task 7）。

### Task 2：两份 Dockerfile ✅

落地：`.cursor/Dockerfile`、`.cursor/Dockerfile.luckfox_pico`。

- [x] 以 lvgl 仓 `origin/dev` 的同名文件为底，用一次性脚本按行替换生成（不保存，§6 第 7 条），未改动的行与 lvgl 仓逐字相同；四组包行注释掉，组名后标「本仓暂不安装，原因见上方说明」。
- [x] `docker build --check` 两份均无告警。

### Task 3：`environment.json`、`install.sh`、`start.sh` ✅

落地：`.cursor/environment.json`、`.cursor/install.sh`、`.cursor/start.sh`；`environment.json`、`install.sh` 的 blob 与 lvgl 仓相同（`47922a7`、`99af772`）。

- [x] ShellCheck 0 条。

### Task 4：三份 `AGENTS.md` ✅

落地：根 `AGENTS.md`、`.cursor/AGENTS.md`、`docs/superpowers/AGENTS.md`。

- [x] 文中的相对链接与本仓路径都存在；根文件 59 行、8,157 字节。
- [x] 事实核对：命令行 `-DCMAKE_INSTALL_PREFIX` 无效（`cmake_install.cmake` 的目标仍是源码树内 `install/`）；`check-headings.py` 不带参数时检查当前目录下的 spec 与 plan。

### Task 5：V1 静态检查 ✅

- [x] 附录 A.1，结果见 §5.1。

### Task 6：V2–V4 ✅

- [x] 附录 A.2–A.4，结果见 §5.2–§5.4。

### Task 7：整理提交、精简 plan、PR 正文 ✅

- [x] `git rebase -i --autosquash origin/dev`：fixup 并入，spec、plan 的提交移到末尾；author time 不变、无 `Co-authored-by`、逐个提交 `check-headings.py` 为 0。
- [x] 精简 plan：已完成 Task 只留落地指针与勾选。
- [x] `--force-with-lease` 推送，更新 PR 正文。
- [x] 按 spec Q10 在 lvgl 仓 PR #17 发[评论](https://github.com/yuangezhizao/luckfox_pico_lvgl_example/pull/17#issuecomment-6919060613)列出跨仓的两处，并在本 PR 发[评论](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/pull/1#issuecomment-6919070433)指向它。
- [x] 对照实际状态完整复查 spec、plan 与 PR 正文；可供以后参考的命令与脚本写入附录 A.5–A.7（一次性的 Dockerfile 生成脚本不保存，§6 第 7 条）。
- [x] 合并前准备：spec 状态行缩短为 PR #1–#4 的写法；本 plan 已是精简后的形式，不再重写；不做完整的 A、B、C 审查（不改代码，两份脚本与 lvgl 仓逐字相同，Dockerfile 的非注释差异只有四组包行，V1–V4 实测通过，本会话也没有子代理工具），改为逐个请求 spec、plan、三份 `AGENTS.md`、Dockerfile 注释、PR 正文与两条评论中的全部链接，均为 200（附录 A.7 末尾的命令；lvgl 仓的评论另行请求）；lvgl 仓评论中指向本仓的链接改为最终提交；本仓暂无 CI。满足门槛后把 PR 改为 ready，不合并。
- [x] 作者要求 spec 开头与 lvgl 仓对齐：删去「流程」（与根 `AGENTS.md`「优先级」重复，For agentic workers 改引后者），「PR」并入「分支」行（同 lvgl 仓 10-06 spec），新增「参考实现」列出被移植文件来自 lvgl 仓的哪些 PR；核对 PR 时发现 spec §2.5 把 `qemu-user` 的来源写成 PR #8，实为 PR #7，一并修正。改动使最终提交的哈希变化，lvgl 仓 PR #17 评论与 Codex 安全评论回复中指向本仓的链接随之更新。
- [x] Codex 审查（`6d35929`）：代码审查未发现问题；安全审查对 `start.sh` 第 109 行提出 P2「停止向 tailnet 暴露无认证出站代理」（1054/1055 经 Serve 发布）。核实：行为属实，但「仅供 Agent 使用」的前提不对，Serve 两个代理是 luckfox-pico 08-30 spec §2.6 的有意设计，§2.8 已接受其风险；`start.sh` 与 lvgl 仓逐字相同。作者决定不修改，用作者账户在线程中[回复](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/pull/1#discussion_r4587552920)（开头标明回复的模型，格式沿用 lvgl 仓 PR #9 对 Codex 安全评论的回复），不标记 resolved；spec §6 补记这一取舍。
- [x] 再次完整复查 spec、plan 与 PR 正文：spec §5 写明 PR 已改为 ready，§6 补 tailnet 端口暴露的取舍；本 Task 补记 Codex 审查；PR 正文概述的参考 PR 与 spec「参考实现」对齐，评审注意补 Codex 审查。

### Task 8：作者在 Cursor 中执行 V5、V6（下一个 PR 开发时）

合并后，作者在 Cursor 中为本 fork 首次启动 Cloud Agent（合并前不要启动，spec §6）：

- [x] 确认 Dashboard 中新建的 saved environment 的 **environment.json location** 指向本仓 `.cursor/environment.json`。
- [x] V5：首个 Build `SUCCEEDED`，日志中 `install.sh` exit 0（grilling `curl` 成功、`openssh-server` 安装、host key 生成）。
- [x] V6：Agent 中 `whoami` 为 `ubuntu`；`start` 输出 `start: complete`；`tailscale status` 的主机名为 `cursor-agent-for-luckfox-pico-rkmpi-example-<bcId 前 8 位>`；`tailscale serve status` 有 5901 / 1054 / 1055；从另一台机器经 tailnet `ssh ubuntu@<主机名>` 成功、`root` 被拒；`stat -c '%U:%G %a' ~/.cursor ~/.cursor/skills/grilling/SKILL.md` 为 `ubuntu:ubuntu`（`install.sh` 依赖平台预建 `~/.cursor`，§6 第 2 条）；按根 `AGENTS.md` 的命令完成 glibc、uClibc 构建。
- [x] 结果记入下一个 PR 的 spec 或 plan；发现问题时在下一个 PR 修复。

以上各项在 [yuangezhizao/luckfox_pico_rkmpi_example#2](https://github.com/yuangezhizao/luckfox_pico_rkmpi_example/pull/2) 的 Cloud Agent 中核对，结果见 spec [`2026-11-17-rkmpi-example-github-actions-ci-design.md`](../specs/2026-11-17-rkmpi-example-github-actions-ci-design.md) §2.5，未发现问题。V6 中 `start` 的输出未进入平台日志，以 tailscaled、Serve 与 sshd 均在运行代替；tailnet SSH 由作者从本地 Mac mini 执行，`ubuntu` 登录成功，`root` 被拒（`Permission denied (publickey)`，sshd 的生效配置为 `permitrootlogin no`）。

## 5. 验证证据

2026-11-16 在当前 VM（pi，Docker 29.9.2，fuse-overlayfs）执行。

### 5.1 V1 静态检查

- `environment.json`、`install.sh` 与 lvgl 仓 `origin/dev` 逐字相同；`start.sh` 只差第 3 行 `repo_slug`。
- 两份 Dockerfile 去掉注释行后，与 lvgl 仓的差异只有四组包行被移除（活动版 4 行、备选版 4 行），其余差异都是注释。
- `.cursor/` 6 个文件模式均为 `100644`，与 lvgl 仓相同；ShellCheck 0 条；`check-headings.py` 2 个文件 0 处。

### 5.2 V2 镜像

| 项 | `rkmpi-env:active` | `rkmpi-env:alt` |
|---|---|---|
| `docker build --check` | 无告警 | 无告警 |
| 体积（lvgl 仓 `11497f3` 同名镜像） | 956 MB（1.32 GB，−364 MB） | 1.07 GB（1.46 GB，−381 MB） |
| 默认用户 | `ubuntu` uid 1000，`HOME=/home/ubuntu`，`sudo -n` 可用 | 同左（`useradd` 创建） |
| 需求包 34 个 | 齐全 | 齐全 |
| 注释掉的包与 Mesa 三包、`libgbm-dev` | 均不在 | 只有 `pkg-config`（FROM 已有） |
| 工具版本 | gcc 13.3.0、cmake 3.28.3、make 4.3、tailscale 1.104.7 | gcc 11.4.0、cmake 3.22.1、make 4.3、tailscale 1.104.7 |

### 5.3 V3 容器内交叉编译

两份镜像中以 `ubuntu` 身份、仓库只读挂载，稀疏检出 `yuangezhizao/luckfox-pico` `dev`（`665c7f1`）：glibc、uClibc 各 5 个例程共 20 次全部成功；产物均为 ELF32 / ARM，interpreter 分别为 `/lib/ld-linux-armhf.so.3`、`/lib/ld-uClibc.so.0`，属主 `ubuntu`，编译输出 `warning:` 0 条；前后 `git status --short` 相同。稀疏检出共 321 MB，其中工具链 233 MB、`.git` 89 MB。

### 5.4 V4 `install.sh`

两份镜像均为：预建 `~/.cursor`（755）后连续运行两次，退出码 0 / 0；host key 指纹不变；`~/.cursor/skills`、`grilling/` 为 `ubuntu:ubuntu 750`，`SKILL.md` 为 `ubuntu:ubuntu 644`，SHA-256 `10ff989e7498b23b5acb49d5048f11dcd906757d2f79c5cdf8a00001381296f2`（与 lvgl 仓 PR #3 相同）；`openssh-server` 活动版 `1:9.6p1-3ubuntu13.20`、备选版 `1:8.9p1-3ubuntu0.18`。

## 6. 与计划的偏离

1. **推送账户**：remote 中的 `cursor[bot]` 令牌对本仓没有写权限，推送 403；`gh` 的作者登录又被环境刷新覆盖。作者重新登录后改用作者账户推送与创建 PR（§1）。
2. **不预建 `~/.cursor` 时 grilling 不可读**：`curl --create-dirs` 把 `~/.cursor` 建为 `root:root 750`，`install.sh` 只 `chown` `skills/`，`ubuntu` 读不到 `SKILL.md`。lvgl 仓 PR #9 已在平台确认 `~/.cursor` 由平台预建为 `755 ubuntu:ubuntu`；脚本与 lvgl 仓逐字相同，本 PR 不改，列入 Task 8 的核对项。
3. **uClibc 工具链体积**：lvgl 仓 `AGENTS.md` 写「约 233 MB」，那是工具链目录本身；稀疏检出连同 `.git` 共 321 MB。先改写为「约 320 MB」，作者审阅后按 spec Q10 删去体积数字（会过时）；lvgl 仓的同一处在其 PR #17 下留评论，随 lvgl 仓下一个 PR 处理。
4. **根 `AGENTS.md` 体积**：初稿 8,459 字节，超过 8 KiB 预算（出处见 spec Q11），精简 `build.sh`、去向表与测试一节的措辞后为 8,173 字节，删去工具链体积后为 8,157 字节。
5. **RUN 中组名后的注释**：Task 2 初稿写「本仓暂不安装，见 spec §3.3」并带完整路径，过长；原因已写在 RUN 上方的分组说明，改为「本仓暂不安装，原因见上方说明」（spec §3.3 同步）。
6. **作者审阅后的修订**：PR 正文中裸写的「PR #1–#3、#9」被 GitHub 解析到本 fork 或上游，改为完整链接，提交哈希、镜像名与官方文档也改为链接；提交正文由整段改为「- 」开头的要点，并写进根 `AGENTS.md`「提交约定」；plan 开头的 For agentic workers 沿用 lvgl 仓 PR #17 的写法，出处先写在 spec 开头新增的「流程」，合并前改为根 `AGENTS.md`「优先级」；实现后按 spec Q10、Q11 删去工具链体积、补 8 KiB 的出处，并发跨仓评论。
7. **Dockerfile 生成脚本不保存**：Task 2 的 `gen-dockerfiles.py` 按 lvgl 仓 `11497f3` 的行号做替换，只能用一次；产物已入库，与 lvgl 仓的差异用附录 A.1 的 diff 即可复现。以后 lvgl 仓的 Dockerfile 改动，按其提交的 diff 同步到本仓两份文件即可。

## 附录 A

命令都在本仓根目录运行。需要：Docker（Cloud Agent 中按 `.cursor/AGENTS.md`「核对镜像内容」临时安装）；lvgl 仓克隆到 `/tmp/lvgl`（`docs/superpowers/AGENTS.md`「检查」一条的命令），供文档检查与体积对照使用；与 lvgl 仓逐字比对取固定提交 `11497f3` 的 raw 文件。

### A.1 V1 静态检查

```bash
ref() { curl -fsSL "https://raw.githubusercontent.com/yuangezhizao/luckfox_pico_lvgl_example/11497f3d57d1e74eeade30a07e79a19c0f7f2369/$1"; }
diff <(ref .cursor/environment.json) .cursor/environment.json && echo json-same
diff <(ref .cursor/install.sh) .cursor/install.sh && echo install-same
diff <(ref .cursor/start.sh) .cursor/start.sh
for f in Dockerfile Dockerfile.luckfox_pico; do
  echo "== $f"
  diff <(ref ".cursor/$f" | grep -v '^\s*#') <(grep -v '^\s*#' ".cursor/$f")
done
git ls-files -s .cursor/   # 6 个文件均应为 100644，与 lvgl 仓相同
shellcheck -x -P SCRIPTDIR .cursor/install.sh .cursor/start.sh && echo shellcheck-0
python3 /tmp/lvgl/tests/tools/docs/check-headings.py
wc -c AGENTS.md
```

### A.2 V2 镜像

```bash
mkdir -p /tmp/empty-ctx   # Dockerfile 不 COPY，用空目录作构建上下文
for t in active:Dockerfile alt:Dockerfile.luckfox_pico; do
  tag=${t%%:*}; f=${t#*:}
  sudo docker build --check -f ".cursor/$f" /tmp/empty-ctx
  sudo docker build -q -t "rkmpi-env:$tag" -f ".cursor/$f" /tmp/empty-ctx
done
need="sudo curl gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf build-essential cmake git ca-certificates iproute2 jq conntrack dmidecode fping htop iftop iotop ncdu neofetch ngrep nmap psmisc screen tcpdump traceroute tree file locales openssh-client openssl rsync unzip vim wget xz-utils tailscale"
off="libcjson-dev libdrm-dev libsdl2-dev pkg-config qemu-user imagemagick xdotool xvfb python3-markdown-it libgl1-mesa-dri libglx-mesa0 libgl1 libgbm-dev"
for tag in active alt; do
  sudo docker run --rm "rkmpi-env:$tag" bash -c "
    whoami; id; echo HOME=\$HOME; sudo -n true && echo sudo-ok
    for p in $need; do dpkg-query -W -f '\${db:Status-Abbrev}' \$p 2>/dev/null | grep -q '^ii' || echo MISSING \$p; done
    for p in $off; do dpkg-query -W -f '\${db:Status-Abbrev}' \$p 2>/dev/null | grep -q '^ii' && echo PRESENT \$p; done
    tailscale version | head -1; arm-linux-gnueabihf-gcc --version | head -1; cmake --version | head -1; make --version | head -1"
done
# 体积对照：lvgl 仓的同名镜像（/tmp/lvgl 为 dev 的浅克隆；本次为 11497f3）
for t in active:Dockerfile alt:Dockerfile.luckfox_pico; do
  sudo docker build -q -t "lvgl-env:${t%%:*}" -f "/tmp/lvgl/.cursor/${t#*:}" /tmp/empty-ctx
done
sudo docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}' | grep -E 'rkmpi-env|lvgl-env'
```

### A.3 V3 容器内交叉编译

```bash
git status --short > /tmp/rkmpi-status-before
for tag in active alt; do
  sudo docker run --rm -v "$PWD:/src:ro" "rkmpi-env:$tag" bash -euo pipefail -c '
    git clone -q --depth 1 --filter=blob:none --sparse -b dev https://github.com/yuangezhizao/luckfox-pico.git /tmp/luckfox-pico
    git -C /tmp/luckfox-pico sparse-checkout set tools/linux/toolchain
    echo "luckfox-pico $(git -C /tmp/luckfox-pico log --oneline -1 | cut -c1-60)  $(du -sh /tmp/luckfox-pico | cut -f1)"
    for libc in glibc uclibc; do
      for ex in luckfox_pico_rtsp_opencv luckfox_pico_rtsp_opencv_capture luckfox_pico_rtsp_retinaface luckfox_pico_rtsp_retinaface_osd luckfox_pico_rtsp_yolov5; do
        b=/tmp/build-$libc-$ex
        if [ $libc = glibc ]; then e="GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-"; else e="LUCKFOX_SDK_PATH=/tmp/luckfox-pico"; fi
        env $e cmake -S /src -B $b -DEXAMPLE_DIR=example/$ex -DEXAMPLE_NAME=$ex -DLIBC_TYPE=$libc > $b.log 2>&1
        env $e make -C $b -j"$(nproc)" >> $b.log 2>&1
        env $e make -C $b install DESTDIR=/tmp/stage >> $b.log 2>&1
        bin=$(find /tmp/stage -path "*/install/$libc/${ex}_demo/$ex" -type f)
        cls=$(readelf -h "$bin" | awk -F: "/Class|Machine/{gsub(/^ +/,\"\",\$2); printf \"%s/\", \$2}")
        interp=$(file -b "$bin" | grep -o "interpreter [^,]*")
        echo "$libc $ex | $cls | $interp | warnings=$(grep -c "warning:" $b.log || true) | owner=$(stat -c %U "$bin")"
      done
    done'
done
git status --short | diff /tmp/rkmpi-status-before - && echo worktree-unchanged
```

`CMakeLists.txt` 的安装目录指向 `/src/install/…`，只读挂载下直接 `make install` 会失败，故用 `DESTDIR`；这同时验证了根 `AGENTS.md` 中推荐的写法。

### A.4 V4 `install.sh`

```bash
for tag in active alt; do
  sudo docker run --rm -v "$PWD:/src:ro" -w /src "rkmpi-env:$tag" bash -uo pipefail -c '
    install -d -m 0755 "$HOME/.cursor"   # 模拟平台预建的 ~/.cursor
    sudo -n -E bash .cursor/install.sh > /tmp/i1.log 2>&1; echo "install#1 rc=$?"
    k1=$(ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub)
    sudo -n -E bash .cursor/install.sh > /tmp/i2.log 2>&1; echo "install#2 rc=$?"
    k2=$(ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub)
    [ "$k1" = "$k2" ] && echo hostkey-stable
    stat -c "%U:%G %a %n" "$HOME/.cursor" "$HOME/.cursor/skills" "$HOME/.cursor/skills/grilling" "$HOME/.cursor/skills/grilling/SKILL.md"
    sha256sum "$HOME/.cursor/skills/grilling/SKILL.md" | cut -c1-64
    dpkg-query -W -f "\${Package} \${Version}\n" openssh-server'
done
# 不预建 ~/.cursor 时（§6 第 2 条）
sudo docker run --rm -v "$PWD:/src:ro" -w /src rkmpi-env:active bash -uo pipefail -c '
  sudo -n -E bash .cursor/install.sh > /tmp/i1.log 2>&1; echo "rc=$?"
  stat -c "%U:%G %a %n" "$HOME/.cursor"
  cat "$HOME/.cursor/skills/grilling/SKILL.md" > /dev/null && echo readable || echo NOT-readable'
```

### A.5 包的复核（spec §2.4；以后增删 apt 包时复用）

```bash
# 平台 Aptfile（只在 Cursor Cloud Agent 中存在）：sha256 与包数，变化时同步两份 Dockerfile 的注释框
f=/usr/local/share/vnc-desktop.Aptfile
sha256sum "$f"; grep -v '^\s*#' "$f" | grep -cv '^\s*$'
# 两个 FROM 已装的包（只计 ii）：apt 取舍只看这里，不在 FROM 才写
U='ubuntu:24.04@sha256:4fbb8e6a8395de5a7550b33509421a2bafbc0aab6c06ba2cef9ebffbc7092d90'
L='luckfoxtech/luckfox_pico:1.0@sha256:915d44588085826cbeda4b969dbbe7d5e54bf779ba36cda3c5072ee9533e0417'
for x in U L; do
  eval img=\$$x
  sudo docker run --rm --entrypoint dpkg-query "$img" -W -f '${db:Status-Abbrev} ${Package}\n' \
    | awk '$1=="ii"{print $2}' | sort > /tmp/from-$x.txt
  echo "$x installed=$(wc -l < /tmp/from-$x.txt)"   # 本次 92 / 366
done
A="ca-certificates git sudo curl g++-arm-linux-gnueabihf gcc-arm-linux-gnueabihf build-essential cmake libcjson-dev libdrm-dev libsdl2-dev pkg-config qemu-user imagemagick xdotool xvfb python3-markdown-it iproute2 jq conntrack dmidecode fping htop iftop iotop ncdu neofetch ngrep nmap psmisc screen tcpdump traceroute tree file locales openssh-client openssl rsync unzip vim wget xz-utils openssh-server procps"
printf '%-26s %-8s %-8s\n' pkg ubuntu luckfox
for p in $A; do
  u=$(grep -qx "$p" /tmp/from-U.txt && echo IN || echo -)
  l=$(grep -qx "$p" /tmp/from-L.txt && echo IN || echo -)
  printf '%-26s %-8s %-8s\n' "$p" "$u" "$l"
done
# 注释写法：RUN 续行中以 # 开头的行在解析时整行删除，被注释的包行保留行尾 \ 也不影响续行
mkdir -p /tmp/dftest && cat > /tmp/dftest/Dockerfile <<'EOF'
FROM ubuntu:24.04@sha256:4fbb8e6a8395de5a7550b33509421a2bafbc0aab6c06ba2cef9ebffbc7092d90
RUN echo start \
      # 组名
      a1 \
      # 注释掉的组
      # b1 b2 \
      c1 \
    && echo end
EOF
sudo docker build --check /tmp/dftest
sudo docker build --no-cache --progress=plain /tmp/dftest 2>&1 | grep 'start a1'   # 应为 start a1 c1
```

### A.6 本仓现状实测（spec §2.1、§2.3、§2.5；qemu 加载冒烟是后续 PR 的起点）

```bash
# 在仓库副本中做，避免改写已入库的 install/；工具链见根 AGENTS.md「项目与构建」
rm -rf /tmp/rkmpi-test && cp -a . /tmp/rkmpi-test && cd /tmp/rkmpi-test
export GLIBC_COMPILER=/usr/bin/arm-linux-gnueabihf-
ex=luckfox_pico_rtsp_yolov5; b=/tmp/rkmpi-b-glibc-$ex; d=install/glibc/${ex}_demo
cmake -S . -B $b -DEXAMPLE_DIR=example/$ex -DEXAMPLE_NAME=$ex -DLIBC_TYPE=glibc > /dev/null
make -C $b -j"$(nproc)" install > /dev/null
git status --short install/                       # make install 改写已跟踪的二进制
# glibc 符号版本：新产物与已提交的二进制（本次均为 GLIBC_2.34）
arm-linux-gnueabihf-objdump -T $d/$ex | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1
git show HEAD:$d/$ex > /tmp/committed.bin
arm-linux-gnueabihf-objdump -T /tmp/committed.bin | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1
arm-linux-gnueabihf-readelf -p .comment /tmp/committed.bin | head -3
git checkout -q -- install
# DESTDIR 不改工作区；命令行 -DCMAKE_INSTALL_PREFIX 无效（安装目标仍在源码树内）
make -C $b install DESTDIR=/tmp/stage > /dev/null && git status --short install/ | wc -l   # 0
cmake -S . -B /tmp/b-prefix -DEXAMPLE_DIR=example/$ex -DEXAMPLE_NAME=$ex -DLIBC_TYPE=glibc -DCMAKE_INSTALL_PREFIX=/tmp/xprefix > /dev/null 2>&1
grep -o 'file(INSTALL DESTINATION "[^"]*"' /tmp/b-prefix/cmake_install.cmake | head -1
# build.sh 非交互运行（2 = glibc，1 = 第一个例程），同样会改写 install/
printf '2\n1\n' | ./build.sh > /tmp/build-sh.log 2>&1; tail -1 /tmp/build-sh.log; git status --short install/; git checkout -q -- install
# qemu 加载冒烟（需要 qemu-user）：所有 .so 解析成功、进入 main()，随后因缺 RkLunch-stop.sh、模型与设备而 abort
timeout 10 qemu-arm -L /usr/arm-linux-gnueabihf -E LD_LIBRARY_PATH=$PWD/$d/lib:$PWD/lib/glibc $d/$ex
# uClibc 工具链体积：工具链目录本身与 .git（本次 233 MB + 89 MB）
du -sh /tmp/luckfox-pico/.git /tmp/luckfox-pico/tools/linux/toolchain/*
```

### A.7 提交整理（Task 7；以后的 PR 可复用）

```bash
E="env -u GIT_AUTHOR_NAME -u GIT_AUTHOR_EMAIL -u GIT_COMMITTER_NAME -u GIT_COMMITTER_EMAIL"
# 只改内容：并入某个提交
$E git commit --fixup=<提交>
# 同时改说明：amend! 提交，第一行为「amend! <原标题>」，空行后是完整的新说明；只改说明时加 --allow-empty
{ echo "amend! <原标题>"; echo; cat /tmp/new-msg.txt; } > /tmp/amend.txt
$E git commit --allow-empty -F /tmp/amend.txt
# autosquash，并把 spec 与 plan 的提交（含其 fixup）移到最后；改写的说明不经过钩子，author time 不变
SPEC_SUBJECT='新增 Cloud Agent 环境 spec 与 plan' GIT_SEQUENCE_EDITOR='python3 /tmp/seq.py' \
  $E git -c core.hooksPath=/dev/null rebase -i --autosquash origin/dev
# 逐个提交核对：author time、签名、文档检查、根 AGENTS.md 字节数；最后确认没有 Co-authored-by
for c in $(git rev-list --reverse origin/dev..HEAD); do
  printf '%s %s sig=%s ' "$(git rev-parse --short $c)" "$(git log -1 --format=%ad --date=format:%H:%M:%S $c)" "$(git cat-file -p $c | grep -c 'BEGIN SSH SIGNATURE')"
  d=$(mktemp -d); git archive $c | tar -x -C $d
  (cd $d && python3 /tmp/lvgl/tests/tools/docs/check-headings.py | tail -1; [ -f AGENTS.md ] && wc -c < AGENTS.md)
  rm -rf $d
done
git log --format=%B origin/dev..HEAD | grep -ci co-authored-by   # 0
# 用作者账户推送（cursor[bot] 对本仓没有写权限）
old=$(git ls-remote https://github.com/yuangezhizao/luckfox_pico_rkmpi_example.git refs/heads/pi/cloudagent-env | cut -f1)
GIT_TERMINAL_PROMPT=0 env -u GIT_ASKPASS git -c credential.helper= \
  -c credential.helper='!f(){ test "$1" = get || exit 0; echo username=yuangezhizao; echo "password=$(gh auth token -u yuangezhizao)"; }; f' \
  push --force-with-lease=pi/cloudagent-env:$old https://yuangezhizao@github.com/yuangezhizao/luckfox_pico_rkmpi_example.git pi/cloudagent-env 2>&1 \
  | sed 's/gh[a-z]_[A-Za-z0-9_]*/***/g'
# 合并前的链接检查：文档、PR 正文与评论中的链接逐个请求（GitHub 链接带作者令牌，私有内容也能查）
{ cat docs/superpowers/specs/*.md docs/superpowers/plans/*.md AGENTS.md .cursor/AGENTS.md docs/superpowers/AGENTS.md .cursor/Dockerfile*
  gh pr view 1 -R yuangezhizao/luckfox_pico_rkmpi_example --json body,comments -q '.body, (.comments[].body)'
} | grep -oE 'https?://[^ )`"<>|，。；、（）]+' | sed 's/[.,]$//' | grep -v '[$*]' | sort -u > /tmp/urls.txt   # 排除模板与本段 case 中的通配模式
TOKEN=$(gh auth token -u yuangezhizao)
while read -r u; do
  base=${u%%#*}
  case $base in
    https://github.com/*|https://raw.githubusercontent.com/*) h=(-H "Authorization: token $TOKEN") ;;
    *) h=(-A 'Mozilla/5.0') ;;
  esac
  echo "$(curl -s -o /dev/null -w '%{http_code}' -L --max-time 20 "${h[@]}" "$base") $u"
done < /tmp/urls.txt | sort
```

`/tmp/seq.py`：

```python
# GIT_SEQUENCE_EDITOR：把标题含 $SPEC_SUBJECT 的提交及其后的 fixup / squash 行移到 todo 末尾
import os
import sys

path = sys.argv[1]
key = os.environ["SPEC_SUBJECT"]
todo = [l for l in open(path, encoding="utf-8").read().split("\n") if l and not l.startswith("#")]
block, rest, i = [], [], 0
while i < len(todo):
    if todo[i].startswith(("pick ", "p ")) and key in todo[i]:
        block.append(todo[i])
        i += 1
        while i < len(todo) and todo[i].split()[0] in ("fixup", "f", "squash", "s"):
            block.append(todo[i])
            i += 1
    else:
        rest.append(todo[i])
        i += 1
if not block:
    sys.exit("找不到标题含 %r 的 pick 行" % key)
open(path, "w", encoding="utf-8").write("\n".join(rest + block) + "\n")
```
