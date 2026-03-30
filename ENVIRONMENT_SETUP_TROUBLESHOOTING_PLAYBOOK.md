# Chipyard 环境配置问题统一排障手册

## 1. 适用范围

本文用于统一处理 Chipyard 在环境初始化阶段的典型失败，重点覆盖：

1. `./build-setup.sh` 第 1 步（Conda environment setup）失败。
2. `conda-lock` 相关失败（缺失 lockfile、TLS/CA 证书链失败）。
3. 企业网络/代理环境下 HTTPS 访问失败。
4. Docker 环境中的 Chipyard 初始化流程偏差。

## 2. 当前已确认的主问题

基于当前仓库与日志，已确认两层问题：

1. 默认 full lockfile 缺失：
   - 期望：`conda-reqs/conda-lock-reqs/conda-requirements-riscv-tools-linux-64.conda-lock.yml`
   - 现状：目录通常只有 lean lockfile。
2. 在尝试生成 full lockfile 时，`conda-lock` 访问 `raw.githubusercontent.com` 发生 TLS 证书链校验失败：
   - `SSLCertVerificationError: self-signed certificate in certificate chain`

补充：当前 `glibc` 与配置一致（`SYS_GLIBC=2.31`, `DEFAULT_GLIBC=2.31`），本轮失败不是 glibc 版本分支导致。

## 3. 方案总览（按推荐顺序）

1. 方案 B（推荐）：修复 TLS/CA 链 + 生成 full lockfile + 走默认完整流程。
2. 方案 A（快速）：使用 `--use-lean-conda` 直接走 lean 路径。
3. 方案 C（临时）：手工准备环境后跳过 Step 1。
4. 方案 D（应急，不推荐长期）：临时关闭 SSL 校验。
5. 方案 E（应急替代）：使用本地 `pypi->conda` lookup 文件，避免在线拉取映射。
6. 方案 F（Docker 场景）：按 base/base-with-tools 分阶段构建，必要时改为容器内执行 `build-setup.sh`。

## 4. 详细方案

## 4.1 方案 B（推荐）：Fix the TLS/CA chain 后执行完整流程

### 步骤

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-lock-env

# 使用企业 CA 证书（示例路径）
conda config --set ssl_verify /path/to/corp-ca.pem
export REQUESTS_CA_BUNDLE=/path/to/corp-ca.pem
export SSL_CERT_FILE=/path/to/corp-ca.pem

# 生成 full lockfile
./scripts/generate-conda-lockfiles.sh

# 验证 full lockfile
test -f conda-reqs/conda-lock-reqs/conda-requirements-riscv-tools-linux-64.conda-lock.yml \
  && echo "full lockfile generated" \
  || echo "full lockfile missing"

# 执行默认完整初始化
./build-setup.sh
```

### 原理

1. `build-setup.sh` 默认依赖 full lockfile。
2. 生成 full lockfile 需要 `conda-lock` 拉取映射文件。
3. 正确配置 CA 后，HTTPS 信任链恢复，映射可下载，lockfile 可生成。

### 适用

1. 需要完整、可复现、长期稳定的环境。
2. 团队共享环境或后续 CI 对齐。

## 4.2 方案 A（快速）：使用 lean 模式

### 步骤

```bash
cd /home/gyk25/chipyard
./build-setup.sh --use-lean-conda
```

### 原理

1. 脚本切换到现有 lean lockfile。
2. 跳过部分重型步骤（如 FireSim/FireMarshal 相关步骤）。

### 适用

1. 先快速跑通。
2. 短期不依赖完整工具链全功能。

## 4.3 方案 C（临时）：跳过 Step 1

### 步骤

```bash
cd /home/gyk25/chipyard
source env.sh
./build-setup.sh -s 1
```

若后续报 `RISCV` 未定义：

```bash
export RISCV=/your/riscv/prefix
./build-setup.sh -s 1
```

### 原理

1. 跳过自动创建 conda 环境。
2. 假定你已手工准备好环境。

### 风险

1. 可复现性差。
2. 依赖漂移会在后续步骤暴露。

## 4.4 方案 D（应急，不推荐长期）：临时关闭 SSL 校验

### 步骤

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-lock-env

conda config --set ssl_verify false
export PYTHONHTTPSVERIFY=0

./scripts/generate-conda-lockfiles.sh
```

成功后立即回滚：

```bash
conda config --set ssl_verify true
unset PYTHONHTTPSVERIFY
```

### 风险

1. 存在中间人攻击风险。
2. 仅建议短时应急，且必须回滚。

## 4.5 方案 E（应急替代）：本地 lookup 文件避开在线映射下载

该方案用于 `conda-lock` 在线访问 `raw.githubusercontent.com` 失败时。

### 步骤

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-lock-env

LOOKUP=/tmp/grayskull_pypi_mapping.yaml
URL='https://raw.githubusercontent.com/regro/cf-graph-countyfair/master/mappings/pypi/grayskull_pypi_mapping.yaml'

# 下载映射（若证书仍受限，可在受控环境中先离线下载再拷贝）
curl -L "$URL" -o "$LOOKUP"

# 使用本地映射文件生成 lockfile
conda-lock --no-mamba --no-micromamba \
  --pypi_to_conda_lookup_file "$LOOKUP" \
  -f conda-reqs/chipyard-base.yaml \
  -f conda-reqs/chipyard-extended.yaml \
  -f conda-reqs/docs.yaml \
  -f conda-reqs/riscv-tools.yaml \
  -p linux-64 \
  --lockfile conda-reqs/conda-lock-reqs/conda-requirements-riscv-tools-linux-64.conda-lock.yml
```

### 原理

1. `conda-lock` 支持 `--pypi_to_conda_lookup_file`。
2. 使用本地文件可绕过运行时在线拉取映射。

### 适用

1. 网络策略严格。
2. 临时无法完成企业 CA 配置但需要继续推进。

## 4.6 方案 F（Docker 场景）

### 推荐流程

1. 先构建 `base` 镜像。
2. 在容器内执行 `build-setup.sh`（而不是依赖可能漂移的入口脚本）。
3. 成功后 `docker commit` 固化。

示例：

```bash
cd /home/gyk25/chipyard
CHIPYARD_HASH=$(git rev-parse HEAD)

docker build -f dockerfiles/Dockerfile \
  --target base \
  --build-arg CHIPYARD_HASH=$CHIPYARD_HASH \
  -t chipyard:base-$CHIPYARD_HASH .

docker run --name chipyard-bootstrap -it --privileged chipyard:base-$CHIPYARD_HASH bash
# 容器内执行：
# cd /root/chipyard
# ./build-setup.sh --use-lean-conda  (或修复 CA 后执行完整流程)

docker commit chipyard-bootstrap chipyard:tools-$CHIPYARD_HASH
docker rm chipyard-bootstrap
```

## 5. Miniforge 与 TLS 问题的关系

1. Miniforge 不能完全替代 TLS/CA 修复。
2. Miniforge解决的是发行版/包管理一致性，不解决企业代理证书信任链本质问题。
3. 只要访问 HTTPS 资源，CA 信任链仍必须正确配置。

## 6. 决策建议（实战）

1. 要长期稳定：优先方案 B。
2. 要最快解锁：先方案 A，再补做方案 B。
3. 要极短期过关：方案 D 或 E，但务必回到 B 做正规修复。
4. 在容器环境：优先方案 F，并提前处理 CA。

## 7. 验证清单

```bash
cd /home/gyk25/chipyard

# 1) full lockfile 是否存在
ls conda-reqs/conda-lock-reqs/

# 2) 第1步是否通过（观察日志是否进入 Step 2）
tail -n 80 build-setup.log

# 3) 环境变量与工具
source env.sh
echo "$CONDA_PREFIX"
echo "$RISCV"
which verilator || true
```

## 8. 后续加固建议

1. 在 `build-setup.sh` 中增加 lockfile 存在性前置检查。
2. 生成 lockfile 前增加 TLS 连通性与证书提示。
3. 将 CA 配置流程写入团队 onboarding 文档。
4. 对 Dockerfile 中初始化入口增加兼容检测（`setup.sh` 不存在时回退 `build-setup.sh`）。

## 9. 结论

当前环境问题的核心不是 conda 本身，而是 lockfile 供应链（文件存在性 + 远端映射下载的 TLS 信任链）。

按工程优先级建议执行顺序：

1. 方案 B（正式修复）
2. 方案 A（快速保底）
3. 方案 C/D/E（仅在必要时短期使用）

## 10. 本次执行步骤拆解与分析（基于最新终端实操）

### 10.1 结论速览

1. `build-setup.sh` 当前新的失败点不在 Conda lockfile 阶段，而在 Step 2 子模块初始化。
2. 一次失败包含两层问题：
  - 主问题：`git submodule update --init --recursive` 访问 GitHub HTTPS 证书链失败（`self-signed certificate in certificate chain`）。
  - 次问题：脚本 `ERR` trap 在 `set -u` 下引用未初始化变量，触发附加报错 `submodule_name: unbound variable`，掩盖主错误可读性。

### 10.2 证据链（对应两份日志）

1. Step 1 已经完成并进入 Step 2：
  - `build-setup.log` 显示 `BEGINNING STEP 2: Initializing Chipyard submodules`。
2. Step 2 中多个仓库均出现相同 TLS 报错，属于系统性网络/证书信任链问题而非单仓库异常：
  - 典型报错：
    - `fatal: unable to access 'https://github.com/...': SSL certificate problem: self-signed certificate in certificate chain`
3. `init-submodules-no-riscv-tools.log` 与 `build-setup.log` 尾部一致，说明失败在子模块脚本内部重现，不是外层脚本偶发中断。
4. 收敛失败点：
  - 首个触发点之一是 `fpga/fpga-shells`，随后大量子模块同类失败。
  - 最终外层看到：
    - `Failed to clone 'fpga/fpga-shells' a second time, aborting`
    - `./scripts/init-submodules-no-riscv-tools-nolog.sh: line 221: submodule_name: unbound variable`

### 10.3 根因分析（软硬件工程视角）

1. 主根因：Git HTTPS 信任链不完整或被企业代理中间证书拦截。
  - 子模块 URL 全部是 `https://github.com/...`。
  - 当主机缺少企业根证书/中间证书时，Git/libcurl 无法完成证书链验证。
2. 次根因：脚本健壮性问题导致错误放大。
  - 文件：`scripts/init-submodules-no-riscv-tools-nolog.sh`
  - 条件：脚本启用了 `set -euo pipefail`，同时 `trap 'error_handler $LINENO "$submodule_name"' ERR`。
  - 在部分错误路径下 `submodule_name` 未初始化，`set -u` 导致 `unbound variable` 二次报错。
3. 对构建流程的影响：
  - 这属于环境级阻塞，不修复 CA/代理信任链，后续任何需要 HTTPS 拉取源码/依赖的步骤都会反复失败。

### 10.4 推荐修复方案（按优先级）

#### 方案 1（推荐，长期）修复 Git CA 信任链

1. 准备企业 CA 证书（PEM）。
2. 配置 Git 与系统证书：

```bash
cd /home/gyk25/chipyard

# 方式A：仅 Git 生效
git config --global http.sslCAInfo /path/to/corp-ca.pem

# 方式B：环境变量（当前 shell 生效）
export GIT_SSL_CAINFO=/path/to/corp-ca.pem
export CURL_CA_BUNDLE=/path/to/corp-ca.pem

# Debian/Ubuntu 可选：安装到系统证书（需要 sudo）
sudo cp /path/to/corp-ca.pem /usr/local/share/ca-certificates/corp-ca.crt
sudo update-ca-certificates
```

3. 连通性验证（必须先过）：

```bash
git ls-remote https://github.com/chipsalliance/rocket-chip-fpga-shells.git
git ls-remote https://github.com/ucb-bar/testchipip.git
```

4. 重跑子模块初始化：

```bash
cd /home/gyk25/chipyard
./scripts/init-submodules-no-riscv-tools.sh
# 或重新执行
./build-setup.sh
```

#### 方案 2（可选）改用 SSH 拉取子模块（绕开 HTTPS 证书链）

前提：已配置 SSH key 并可访问 GitHub。

```bash
cd /home/gyk25/chipyard
git submodule sync --recursive
git config --global url."git@github.com:".insteadOf "https://github.com/"
git submodule update --init --recursive
```

说明：该方案可绕过 HTTPS 证书问题，但团队统一性通常不如方案 1。

#### 方案 3（临时应急，不推荐长期）关闭 Git SSL 校验

```bash
cd /home/gyk25/chipyard
git -c http.sslVerify=false submodule update --init --recursive
```

风险：存在 MITM 风险，仅可短时应急，成功后务必回到方案 1。

### 10.5 脚本次级问题修复建议（提升可维护性）

为避免主错误被 `unbound variable` 噪声覆盖，建议修正 trap 参数展开：

```bash
# 当前
trap 'error_handler $LINENO "$submodule_name"' ERR

# 建议（submodule_name 未定义时给空串）
trap 'error_handler $LINENO "${submodule_name-}"' ERR
```

或在 trap 前初始化：

```bash
submodule_name=""
trap 'error_handler $LINENO "$submodule_name"' ERR
```

这不会改变主问题，但能显著提升故障定位效率。

### 10.6 本轮建议执行顺序（可直接照做）

1. 完成 CA 配置（方案 1）。
2. 用 `git ls-remote` 验证至少 2 个 GitHub 仓库可访问。
3. 先单跑 `./scripts/init-submodules-no-riscv-tools.sh` 验证 Step 2。
4. 通过后再跑 `./build-setup.sh` 全流程。
5. （可选）给子模块脚本应用 trap 健壮性修复，减少后续噪声报错。

### 10.7 验证通过标准

1. `init-submodules-no-riscv-tools.log` 不再出现 `SSL certificate problem: self-signed certificate in certificate chain`。
2. `build-setup.log` 不再在 Step 2 退出。
3. 不再出现 `submodule_name: unbound variable`。
4. 后续步骤能进入工具链/仿真组件构建阶段。

本节对应你刚执行的命令链：

```bash
cd /home/gyk25/chipyard && \
source "$(conda info --base)/etc/profile.d/conda.sh" && \
conda activate /home/gyk25/chipyard/.conda-lock-env && \
set -e && \
CA_FILE=/etc/ssl/certs/ca-certificates.crt && \
echo "[1/5] Configure CA path" && \
conda config --set ssl_verify "$CA_FILE" && \
export REQUESTS_CA_BUNDLE="$CA_FILE" && \
export SSL_CERT_FILE="$CA_FILE" && \
echo "[2/5] Verify effective conda ssl_verify" && \
conda config --show ssl_verify && \
echo "[3/5] Re-generate lockfiles" && \
./scripts/generate-conda-lockfiles.sh && \
echo "[4/5] Verify full lockfile exists" && \
test -f conda-reqs/conda-lock-reqs/conda-requirements-riscv-tools-linux-64.conda-lock.yml && \
ls -l conda-reqs/conda-lock-reqs/conda-requirements-riscv-tools-linux-64*.conda-lock.yml && \
echo "[5/5] Done"
```

### 10.1 逐步说明

1. `cd /home/gyk25/chipyard`
  - 切到仓库根目录，确保后续脚本相对路径有效。

2. `source "$(conda info --base)/etc/profile.d/conda.sh"`
  - 加载 conda shell 函数，使 `conda activate` 在当前 shell 可用。

3. `conda activate /home/gyk25/chipyard/.conda-lock-env`
  - 激活用于 lockfile 生成的专用环境。

4. `set -e`
  - 任一步失败即退出，避免“前一步失败但后续继续执行”导致误判。

5. `CA_FILE=/etc/ssl/certs/ca-certificates.crt`
  - 指定本机已验证可用的系统 CA bundle 路径。

6. `conda config --set ssl_verify "$CA_FILE"`
  - 将 conda 的 TLS 校验证书源切到系统 CA。

7. `export REQUESTS_CA_BUNDLE="$CA_FILE"` 与 `export SSL_CERT_FILE="$CA_FILE"`
  - 将 Python requests/SSL 的会话级证书路径也统一到该 CA。

8. `conda config --show ssl_verify`
  - 校验配置是否生效。你的输出为：
  - `ssl_verify: /etc/ssl/certs/ca-certificates.crt`
  - 说明 TLS/CA 配置步骤已经成功。

9. `./scripts/generate-conda-lockfiles.sh`
  - 触发 full lockfile 生成（目标文件名不带 `-lean`）。

10. `test -f ...` 和 `ls -l ...`
   - 仅在 lockfile 成功生成时执行，用于最终确认。

### 10.2 从输出判断当前状态

你这次执行中，生成阶段出现：

- `Locking dependencies for ['linux-64']...`
- `INFO:conda_lock.conda_solver:linux-64 using specs ...`
- 随后 `^C` 与 `Aborted!`

这说明：

1. 证书链问题已明显缓解，流程已经进入 conda-lock 求解阶段。
2. 当前失败不是 TLS 校验失败，而是进程被中断（通常是人工 Ctrl-C 或终端中断信号）。
3. 因为 `set -e`，一旦中断返回非零，命令链整体退出，最终 exit code 为 1。

### 10.3 关键结论

1. `Fix the TLS/CA chain` 配置动作已生效。
2. 当前阻塞不再是证书链，而是 lock 求解过程被打断。
3. 下一步应做“不中断重试”，而不是继续改 CA 配置。

### 10.4 建议的重试方式（避免再次中断）

方式 A：前台直接跑，期间不要中断

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-lock-env
export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
export SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
./scripts/generate-conda-lockfiles.sh
```

方式 B：后台跑并落日志（推荐）

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-lock-env
export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
export SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
nohup ./scripts/generate-conda-lockfiles.sh > /tmp/chipyard-lockgen.log 2>&1 &
tail -f /tmp/chipyard-lockgen.log
```

完成后验证：

```bash
test -f /home/gyk25/chipyard/conda-reqs/conda-lock-reqs/conda-requirements-riscv-tools-linux-64.conda-lock.yml \
  && echo "full lockfile generated" \
  || echo "full lockfile missing"
```

### 10.5 生成成功后的后续动作

```bash
cd /home/gyk25/chipyard
./build-setup.sh
```

如果只想先快速推进：

```bash
./build-setup.sh --use-lean-conda
```

## 11. 新问题：TLS 已通过后，pip wheel 平台不匹配

### 11.1 现象

`build-setup.log` 出现：

- `numpy-1.26.4-cp310-...whl is not a supported wheel on this platform`
- `conda run pip install --no-deps -r /tmp/... failed`

最终 Step 1 失败。

### 11.2 关键分析

该错误表面看是 wheel 平台不匹配，实质是 `pip` 解释器不匹配。

已验证事实：

1. lockfile 中 `python` 已锁定到 3.10（conda 包）。
2. `.conda-env` 本身也包含 `python3.10`。
3. 但执行
   - `conda run --prefix /home/gyk25/chipyard/.conda-env which pip`
   返回的是外部环境路径：`/home/gyk25/.conda/envs/nlp-gpu/bin/pip`。
4. 因此外部 Python 3.12 的 pip 去安装 cp310 wheel，触发“not a supported wheel”。

结论：

1. 这不是 numpy 包损坏。
2. 这是 PATH 被已有 conda 环境污染，`conda run --prefix` 子进程中优先命中了错误 pip。

### 11.3 根因机制

`conda run --prefix` 环境中，PATH 片段顺序为：

1. `/home/gyk25/chipyard/.conda-env/riscv-tools/bin`
2. `/home/gyk25/.conda/envs/nlp-gpu/bin`
3. `/home/gyk25/chipyard/.conda-env/bin`

由于外部环境 bin 目录在 `.conda-env/bin` 之前，`pip` 解析到外部环境，导致 Python ABI 不一致。

### 11.4 解决方案（推荐顺序）

#### 方案 11-A（推荐）：在“干净 shell”中重新执行 build-setup

在一个全新 shell 中执行：

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"

# 反复退出直到不在任何 conda env 中
while [ -n "${CONDA_PREFIX:-}" ]; do
  conda deactivate || break
done

# 清理可能影响 pip 的环境变量
unset PYTHONPATH
unset PIP_REQUIRE_VIRTUALENV
hash -r

# 维持已验证可用的 CA 配置
export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
export SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt

./build-setup.sh
```

#### 方案 11-B（强校验后再继续）

在运行 `build-setup.sh` 前先验证 `conda run --prefix` 使用的 pip：

```bash
/usr/local/anaconda3/bin/conda run --prefix /home/gyk25/chipyard/.conda-env which pip
```

期望输出应为：

- `/home/gyk25/chipyard/.conda-env/bin/pip`

若仍不是该路径，不要继续运行 build-setup，先回到方案 11-A 清理环境。

#### 方案 11-C（临时兜底）：强制使用解释器调用 pip

如果必须局部修复，可在相关命令中改为：

```bash
python -m pip install --no-deps -r <requirements>
```

这样可避免 PATH 解析到错误 pip。但该方案需要修改上游调用链（conda-lock 内部调用），维护成本高，不作为首选。

### 11.5 验证标准

1. `which pip` 与 `pip -V` 在 prefix 下都指向 `.conda-env`。
2. `build-setup.log` 不再出现 `numpy ... not a supported wheel on this platform`。
3. Step 1 成功结束，日志继续进入 Step 2。

## 12. 新问题：自动生成 full lockfile 时 sysroot_linux-64=2.31 不可解

### 12.1 现象

在自动构建 `conda-requirements-riscv-tools-linux-64.conda-lock.yml` 的流程中，日志报错：

1. `PackagesNotFoundError: sysroot_linux-64=2.31`
2. `Could not lock the environment for platform linux-64`
3. Step 1 退出失败。

### 12.2 已确认事实（基于实机查询）

1. 当前 channel 可见版本中，`sysroot_linux-64` 有 `2.17 / 2.28 / 2.34 / 2.39`。
2. `2.31` 当前不可用。
3. 失败发生在 lock 求解阶段（conda dry-run solve），不是 TLS 失败，也不是 pip 失败。

### 12.3 根因分析（软硬件工程视角）

这是一个“依赖约束与仓库可用包集合漂移”的问题：

1. 约束侧：`chipyard-base.yaml` 里 pin 了 `sysroot_linux-64=2.31`。
2. 供给侧：当前上游 channel 已不提供该精确版本。
3. 求解器行为：精确 pin 不可满足时直接失败。

### 12.3.1 为什么通常要求 sysroot_linux-64 版号 <= 系统 glibc 版号

核心原因是运行时 ABI 兼容边界。

1. `sysroot_linux-64` 代表构建时可见的 glibc 头文件与符号基线。
2. 如果用更高版本 sysroot（例如 2.39）在较低 glibc 主机（例如 2.31）上构建，编译产物可能链接到主机不存在的符号版本（如 `GLIBC_2.34+`）。
3. 产物在运行或加载时会报符号缺失（典型是 `version 'GLIBC_x.y' not found`）。

因此在“本机构建 + 本机运行/仿真”场景里，`sysroot` 版号通常应不高于目标运行机 glibc 版号，即：

- 推荐约束：`sysroot_linux-64 <= host_glibc`

工程上这样做的收益：

1. 降低二进制在当前服务器上的运行失败概率。
2. 降低跨团队机器复现时的 ABI 偏差。
3. 让工具链、仿真器与辅助工具在同一 glibc 能力边界内。

边界与例外（需要明确）：

1. 如果最终运行环境并非当前主机，而是容器/CI/集群节点，应该以“目标运行环境 glibc”为准，而非开发机。
2. 某些静态链接或自带 runtime 的组件受影响较小，但 Chipyard 流程中大量工具仍会受动态链接 ABI 约束。
3. 在当前项目里，还要叠加“频道可解性”约束：即使满足 `<=`，若该版本在 channels 已下线，同样无法求解。

对工程链路的影响：

1. 软件层：Conda 环境无法完成求解，锁文件无法生成。
2. 硬件/工具链层：交叉编译与主机构建依赖一致性被打断，导致后续 Verilator、FireMarshal、工具链附加构建无法进入。
3. 复现性层：环境清单与上游仓库状态不一致，旧 pin 失效。

### 12.4 解决方案（按优先级）

#### 方案 12-A（推荐，务实可落地）：将 sysroot pin 调整为当前可用版本后重锁

建议选 `2.39`（当前可用且维护活跃），步骤：

```bash
cd /home/gyk25/chipyard

# 备份
cp conda-reqs/chipyard-base.yaml conda-reqs/chipyard-base.yaml.bak.$(date +%Y%m%d-%H%M%S)

# 将 sysroot pin 从 2.31 改为 2.39
sed -i 's/^\([[:space:]]*-\s*sysroot_linux-64=\).*/\12.39/' conda-reqs/chipyard-base.yaml

# 重新生成 lockfile
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-lock-env
export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
export SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
./scripts/generate-conda-lockfiles.sh
```

成功后再执行：

```bash
./build-setup.sh
```

#### 方案 12-B（保守兼容）：改为 2.34 并验证

如果你希望减少与历史 glibc 语义差距，可尝试 `2.34`。流程同 12-A，只替换版本号。

#### 方案 12-C（追求历史复现）：引入历史快照 channel

若团队必须保持旧版本完全复现，可使用历史快照镜像或内部包缓存仓库，恢复 `sysroot_linux-64=2.31` 可见性。

适用场景：

1. 论文/发布版本需要字节级复现。
2. 旧流水线基于固定二进制快照。

代价：

1. 运维复杂度高。
2. 对镜像与缓存治理要求高。

### 12.5 风险评估与回归建议

将 sysroot 从 2.31 升级到 2.34/2.39 后，建议做一轮最小回归：

1. 环境与工具链健康检查：

```bash
source env.sh
which gcc
which verilator
which spike || true
```

2. 编译链路检查：

```bash
cd /home/gyk25/chipyard/sims/verilator
make launch-sbt SBT_COMMAND=";project chipyard; compile"
```

3. 关键软件回归（至少一个基础 workload）

```bash
cd /home/gyk25/chipyard/tests
# 按团队已有最小回归目标执行
```

### 12.6 推荐执行顺序

1. 先采用 12-A（2.39）打通流程。
2. 立即跑最小回归确认无回归。
3. 若团队对环境漂移敏感，再评估 12-C 建立历史包快照机制。

## 13. 补充定位：sysroot_linux-64=2.31 的来源与调用链

以下是拉取 sysroot_linux-64=2.31 的精确来源位置与代码链路。

1. 版本定义源（真正决定版本的位置）
: [conda-reqs/chipyard-base.yaml](conda-reqs/chipyard-base.yaml#L19)
: 关键片段
  - - sysroot_linux-64=2.31

2. build-setup 脚本读取该版本的位置
: [build-setup.sh](build-setup.sh#L189)
: 关键片段
  - DEFAULT_GLIBC=$(grep -i "sysroot_linux-64=" conda-reqs/chipyard-base.yaml | awk -F= '{print $2}')

3. build-setup 触发 lockfile 重生成的位置
: [build-setup.sh](build-setup.sh#L185)
: [build-setup.sh](build-setup.sh#L193)
: 关键片段
  - $CYDIR/scripts/generate-conda-lockfiles.sh

4. lockfile 生成脚本消费该依赖定义的位置
: [scripts/generate-conda-lockfiles.sh](scripts/generate-conda-lockfiles.sh#L26)
: 关键片段
  - -f "$REQS_DIR/chipyard-base.yaml"

5. 日志中确认求解器最终请求该版本的位置
: [build-setup.log](build-setup.log#L156)
: [build-setup.log](build-setup.log#L195)
: 关键片段
  - sysroot_linux-64 2.31.*
  - PackagesNotFoundError: sysroot_linux-64=2.31

结论：

1. 特定版本 2.31 的根源是配置文件 [conda-reqs/chipyard-base.yaml](conda-reqs/chipyard-base.yaml#L19) 的硬编码 pin。
2. 其余脚本只是读取并传递该约束到 conda-lock 求解过程。

## 14. 补充定位：chipyard-base.yaml 被自动修改的触发原码与修复方案

### 14.1 触发原码（精确位置）

自动修改 `chipyard-base.yaml` 的代码在 [build-setup.sh](build-setup.sh#L188) 到 [build-setup.sh](build-setup.sh#L193)：

1. 读取系统 glibc：
  - [build-setup.sh](build-setup.sh#L188)
  - `SYS_GLIBC=$(ldd --version | awk '/ldd/{print $NF}')`
2. 读取 yaml 中 sysroot 版本：
  - [build-setup.sh](build-setup.sh#L189)
  - `DEFAULT_GLIBC=$(grep -i "sysroot_linux-64=" conda-reqs/chipyard-base.yaml | awk -F= '{print $2}')`
3. 触发条件（不相等即改写）：
  - [build-setup.sh](build-setup.sh#L190)
  - `if [ "$SYS_GLIBC" != "$DEFAULT_GLIBC" ]; then`
4. 实际改写命令（并生成 `.bak` 备份）：
  - [build-setup.sh](build-setup.sh#L192)
  - `sed -i.bak "s/^\([[:space:]]*-\s*sysroot_linux-64=\).*/\1$SYS_GLIBC/" conda-reqs/chipyard-base.yaml`

`.bak` 说明：

1. `sed -i.bak` 会把改写前文件保存为同目录的 `chipyard-base.yaml.bak`。
2. 当脚本再次运行并再次改写时，`.bak` 可能被覆盖，属于单份滚动备份，不是版本化历史。

### 14.2 为什么这会导致你当前中断

根据当前实测：

1. 系统 glibc 是 `2.31`。
2. 当前 yaml 的 sysroot 是 `2.34`。
3. 因为二者不一致，脚本会自动把 yaml 改回 `2.31`。
4. 但当前 channel 中 `sysroot_linux-64=2.31` 不可用，随后 lock 求解失败并中断。

这是一种“自动修正逻辑与仓库可用包集合漂移冲突”的问题。

### 14.3 解决方案（推荐顺序）

#### 方案 14-A（推荐）：保留可解版本，禁用自动回写到 2.31

目标：避免脚本把 `2.34`/`2.39` 自动改回不可解的 `2.31`。

操作建议：

1. 将 `chipyard-base.yaml` 的 sysroot 固定在当前可解版本（例如 2.34 或 2.39）。
2. 修改 [build-setup.sh](build-setup.sh#L190) 逻辑：
  - 不直接以 `SYS_GLIBC != DEFAULT_GLIBC` 为改写条件。
  - 增加“目标版本是否在当前 channels 可解”的检测。
  - 仅在可解时才回写，否则保留原版本并输出 warning。

#### 方案 14-B（流程隔离）：锁文件生成阶段不修改源 yaml

目标：避免构建脚本对源依赖清单做 in-place 改写。

做法：

1. 复制 `chipyard-base.yaml` 到临时文件。
2. 仅对临时文件做 sysroot 替换并求解。
3. 原始 `chipyard-base.yaml` 保持不变。

优点：

1. 降低“脚本副作用导致仓库状态漂移”的风险。
2. 对多次重试和团队协作更安全。

#### 方案 14-C（短期手工恢复）

当已发生自动改写并导致失败时：

```bash
cd /home/gyk25/chipyard

# 若存在 .bak，先恢复
test -f conda-reqs/chipyard-base.yaml.bak && cp conda-reqs/chipyard-base.yaml.bak conda-reqs/chipyard-base.yaml

# 若 .bak 不存在，则手工改回可解版本（2.34 或 2.39）
sed -i 's/^\([[:space:]]*-\s*sysroot_linux-64=\).*/\12.34/' conda-reqs/chipyard-base.yaml
```

随后重新生成 lockfile 并验证。

### 14.4 工程建议

1. 将 `.bak` 机制替换为“带时间戳备份”或 git checkpoint，避免备份被覆盖。
2. 将 sysroot 版本策略写入团队规范：
  - 优先使用“可解且已回归验证”的版本，而不是盲目追随系统 glibc。
3. 对 `build-setup.sh` 增加明确日志：
  - 打印“即将修改 yaml”的原因、目标值、以及可解性检查结果。

## 15. 日志二次复盘（2026-03-29 最新状态）

本节基于你要求重新详细阅读后的两份最新日志：

1. `build-setup.log`（341 行）
2. `init-submodules-no-riscv-tools.log`（337 行）

### 15.1 当前编译状态结论

1. 这次运行已经不再卡在 SSL/证书链问题。
2. Step 2（子模块初始化）已实质完成，并成功进入 Step 3。
3. 当前唯一阻塞点是 Step 3 前置环境不满足：
  - `build-setup.sh: ERROR: If conda initialization skipped, $RISCV variable must be defined.`

### 15.2 关键证据链

1. 日志开头直接出现：
  - `!!!!! WARNING: No conda environment detected...`
  这说明本次 shell 中未激活 conda 环境。
2. 仍然进入了 Step 2：
  - `BEGINNING STEP 2: Initializing Chipyard submodules`
3. 子模块日志尾部显示大量 `checked out` 成功记录，无 `fatal:`。
4. `build-setup.log` 明确显示已经进入：
  - `BEGINNING STEP 3: Building toolchain collateral`
5. 随后立即在 Step 3 失败，报错为 `$RISCV` 未定义。

### 15.3 根因分析（为什么 Step 2 成功但 Step 3 失败）

结合脚本逻辑（`build-setup.sh`）：

1. Step 3 中存在分支：
  - 若执行了 Step 1（Conda setup），则使用 `PREFIX=$CONDA_PREFIX/$TOOLCHAIN_TYPE`。
  - 若跳过了 Step 1，则强制要求 `RISCV` 已定义，否则报错退出。
2. 你的日志特征符合“Step 1 被跳过，但 RISCV 没有提前导出”的路径。
3. 因为 Step 2 主要是 `git submodule` 操作，不依赖 `RISCV`，所以可以成功；而 Step 3 需要安装/构建 toolchain collateral，必须知道 prefix 路径。

### 15.4 与上一轮问题的关系

1. 上一轮核心问题：GitHub HTTPS 证书链导致子模块克隆失败。
2. 本轮核心问题：证书问题已缓解，流程前移到 toolchain 前置变量校验。
3. 这属于“阻塞点迁移”而不是“问题反复”。

### 15.5 建议修复方案（当前最匹配）

#### 方案 15-A（推荐）走标准 conda 路径，不跳过 Step 1

适用：希望长期可复现，尽量与官方默认流程一致。

1. 先激活 conda 并加载项目环境：

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
source env.sh
```

2. 继续执行后续步骤时显式跳过已完成的 Step 2（避免重复拉子模块）：

```bash
./build-setup.sh -s 2
```

说明：是否需要跳过 Step 1 取决于 `.conda-env` 是否已经完整可用。若 `.conda-env` 未就绪，保留 Step 1；若已就绪，见方案 15-B。

#### 方案 15-B（快速恢复）保留“跳过 Step 1”，补齐 RISCV

适用：当前 `.conda-env` 已可用，目标是尽快推进到 Step 3+。

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
source env.sh
export RISCV="/home/gyk25/chipyard/.conda-env/riscv-tools"
./build-setup.sh -s 1 -s 2
```

要点：

1. `-s 1` 表示跳过 conda 初始化。
2. `-s 2` 表示跳过子模块初始化（因为本轮已完成）。
3. 关键是 `RISCV` 必须在当前 shell 已定义。

### 15.6 执行前自检清单（建议先跑）

```bash
cd /home/gyk25/chipyard
echo "CONDA_DEFAULT_ENV=${CONDA_DEFAULT_ENV:-<unset>}"
echo "CONDA_PREFIX=${CONDA_PREFIX:-<unset>}"
echo "RISCV=${RISCV:-<unset>}"
test -d /home/gyk25/chipyard/.conda-env && echo ".conda-env exists" || echo ".conda-env missing"
```

### 15.7 通过标准

1. 不再出现 `ERROR: If conda initialization skipped, $RISCV variable must be defined.`
2. `build-setup.log` 从 Step 3 继续推进到后续步骤（ctags / precompile / firesim / marshal / circt 等，视 skip 参数而定）。
3. `init-submodules-no-riscv-tools.log` 保持无 `fatal:` 级别错误。

### 15.8 已执行修复过程（CA + 连接验证）

本小节记录已在当前仓库中实际执行并通过的操作，用于复现与审计。

#### A. 配置 Git CA（仓库本地）

执行命令：

```bash
cd /home/gyk25/chipyard
CA_FILE=/etc/ssl/certs/ca-certificates.crt
git config --local http.sslCAInfo "$CA_FILE"
git config --local http.sslVerify true

# 下面的命令在kimi k2.5的帮助下提出（真正解决问题）
git config --global http.sslCAInfo /etc/ssl/certs/ca-certificates.crt
```

验证命令：

```bash
git config --show-origin --get http.sslCAInfo
git config --show-origin --get http.sslVerify
```

验证结果（通过）：

1. `http.sslCAInfo` 来源为 `file:.git/config`，值为 `/etc/ssl/certs/ca-certificates.crt`。
2. `http.sslVerify` 来源为 `file:.git/config`，值为 `true`。

#### B. 验证 GitHub HTTPS 连通性

执行命令：

```bash
cd /home/gyk25/chipyard
git ls-remote https://github.com/riscv-software-src/riscv-isa-sim.git | head -n 2
git ls-remote https://github.com/chipsalliance/rocket-chip.git | head -n 2
```

验证结果（通过）：

1. 两个仓库都成功返回 `HEAD` 与分支引用哈希。
2. 未出现 `SSL certificate problem: self-signed certificate in certificate chain`。

#### C. 结论

1. 当前仓库内 Git HTTPS 证书链已恢复可用。
2. 后续若仍构建失败，优先排查 Step 3 的环境变量/路径问题（如 `RISCV`），而非 CA 问题。

## 16. 最新问题复盘（Step 8 / FireMarshal 子模块）

### 16.1 精确错误定位（build-setup.log）

本轮 `build-setup.log` 总计约 15505 行，失败发生在 Step 8：

1. 进入 Step 8：`BEGINNING STEP 8: Setting up FireMarshal`
2. 触发命令：
  - `git submodule update --progress --filter=tree:0 --init boards/default/linux ...`
3. 关键报错：
  - `error: RPC failed; curl 56 Recv failure: Connection reset by peer`
  - `fatal: expected 'packfile'`
  - `fatal: could not fetch ... from promisor remote`
  - `fatal: Unable to checkout ... in submodule path 'boards/default/linux'`
4. 外层退出：
  - `build-setup.sh: Build script failed with exit code 128 at step 8: Setting up FireMarshal`

### 16.2 现象与状态解读

1. `boards/default/distros/br/buildroot` 与 `boards/default/firmware/opensbi` 已 checkout 成功。
2. 失败集中在 `boards/default/linux`（体量最大，网络最敏感）。
3. FireMarshal 的子模块初始化脚本确实使用了 `--filter=tree:0`（部分克隆），会依赖后续按需从 promisor remote 拉取对象。
4. 当前仓库 CA 已修复且 `git ls-remote` 可通，因此这次不是证书链问题，而是传输过程中的连接重置导致 packfile 不完整。

### 16.3 根因分析

主根因：大仓库（Linux 子模块）在部分克隆/大对象传输阶段发生网络中断（`Connection reset by peer`），导致 promisor pack 获取不完整，最终 checkout 失败。

常见触发条件：

1. 企业代理/网关对长连接或大流量传输不稳定。
2. HTTP/2 在特定网络设备下偶发中断。
3. 部分克隆 + promisor 按需抓取在抖动网络下更容易暴露失败。

### 16.4 解决方案（按推荐顺序）

#### 方案 16-A（推荐）仅重试 Step 8，并切换 Git 为 HTTP/1.1

```bash
cd /home/gyk25/chipyard

# 减少代理/网关下的 HTTP2 兼容问题
git config --global http.version HTTP/1.1

# 降低并发，减少网络压力
git config --global submodule.fetchJobs 1

# 仅重跑 Step 8（其余步骤按需跳过）
source "$(conda info --base)/etc/profile.d/conda.sh"
source env.sh
./build-setup.sh -s 1 -s 2 -s 3 -s 4 -s 5 -s 6 -s 7
```

说明：该方案对现有流程侵入最小，优先尝试。

#### 方案 16-B（增强稳定性）先单独修复 FireMarshal 子模块，再继续 build-setup

```bash
cd /home/gyk25/chipyard/software/firemarshal

# 同步子模块 URL 配置
git submodule sync --recursive

# 先尝试仅拉取最容易失败的 linux 子模块（单线程）
git -c http.version=HTTP/1.1 -c submodule.fetchJobs=1 \
  submodule update --init --progress boards/default/linux

# 成功后再补齐其余目标
./init-submodules.sh
```

若上面成功，再回到仓库根目录继续：

```bash
cd /home/gyk25/chipyard
./build-setup.sh -s 1 -s 2 -s 3 -s 4 -s 5 -s 6 -s 7
```

#### 方案 16-C（必要时）清理失败子模块后重拉

```bash
cd /home/gyk25/chipyard/software/firemarshal

git submodule deinit -f boards/default/linux
rm -rf boards/default/linux

git -c http.version=HTTP/1.1 -c submodule.fetchJobs=1 \
  submodule update --init --progress boards/default/linux
```

说明：适用于子模块工作树已损坏或处于不一致状态时。

### 16.5 验证标准

1. `build-setup.log` 中不再出现：
  - `curl 56 Recv failure: Connection reset by peer`
  - `expected 'packfile'`
  - `could not fetch ... from promisor remote`
2. Step 8 能继续执行到 FireMarshal 后续子步骤（或完成 Step 8）。
3. `git -C software/firemarshal submodule status boards/default/linux` 不再报错且状态稳定。

### 16.6 工程化建议

1. 对网络敏感环境，默认设置 `http.version=HTTP/1.1` 与 `submodule.fetchJobs=1`。
2. 在 CI/服务器上为 Step 8 增加“失败后重试一次”的封装。
3. 对 `boards/default/linux` 这类大子模块，优先单独拉取成功后再执行全量初始化。

## 17. 最新问题复盘（Step 8 参数兼容性错误）

### 17.1 精确定位

在执行：

```bash
./build-setup.sh riscv-tools -s 1 -s 2 -s 3 -s 4 -s 5 -s 6 -s 7
```

后，`build-setup.log`（16 行）显示：

1. 进入 Step 8：`BEGINNING STEP 8: Setting up FireMarshal`
2. 触发命令：
   - `git submodule update --progress --filter=tree:0 --init ...`
3. Git 直接打印 `usage: git submodule ...`（说明参数解析失败，而非网络传输失败）
4. 最终退出：`build-setup.sh: Build script failed with exit code 1 at step 8`

### 17.2 根因分析

1. 当前环境 Git 版本：`2.25.1`。
2. 该版本的 `git submodule update -h` 用法中不支持 `--filter`。
3. `software/firemarshal/init-submodules.sh` 固定使用了 `--filter=tree:0`，导致命令在参数解析阶段立即失败。

结论：这是“Git 版本能力与脚本参数不匹配”的兼容性问题，不是 CA、网络或磁盘问题。

### 17.3 已实施修复

已对 `software/firemarshal/init-submodules.sh` 做兼容改造：

1. 运行时检测 `git submodule update -h` 是否支持 `--filter`。
2. 支持时保留 `--filter=tree:0`。
3. 不支持时自动去掉该参数，回退到普通子模块拉取。

核心逻辑：

```bash
FILTER_ARGS=()
if git submodule update -h 2>&1 | grep -q -- '--filter'; then
  FILTER_ARGS=(--filter=tree:0)
fi

git submodule update --progress "${FILTER_ARGS[@]}" --init ...
```

### 17.4 解决方案建议（可选）

1. 短期（已落地）：使用上述脚本兼容分支，保证当前环境可继续。
2. 中期：升级 Git 至支持 `submodule update --filter` 的版本（建议 >= 2.30），可保留部分克隆优势。
3. 长期：在环境初始化前增加 Git capability precheck（版本或 `--filter` 可用性），避免运行时失败。

### 17.5 验证标准

1. Step 8 不再出现 `usage: git submodule ...`。
2. `build-setup.log` 中 Step 8 进入真正的子模块下载/checkout流程。
3. 若后续失败，错误应转为网络或仓库层面的可诊断问题，而非参数兼容性错误。

## 18. 最新问题复盘（Step 8 / index.lock 遗留）

### 18.1 精确错误定位

修复 17 节后再次执行同一命令，`build-setup.log` 报错变为：

1. Step 8 已进入 FireMarshal 子模块初始化。
2. 兼容分支生效：`git submodule update --progress --init ...`（不再含 `--filter`）。
3. 新的失败信息：
  - `fatal: Unable to create '.../modules/riscv-linux/index.lock': File exists.`
  - 提示有另一个 git 进程或历史崩溃遗留锁文件。

### 18.2 根因分析

1. 这是 Git 仓库锁保护机制触发，不是网络、CA 或权限问题。
2. 常见来源：
  - 前一次 `git submodule update` 被中断（Ctrl-C / 会话断开）；
  - Git 子进程异常退出，未清理 lock 文件。
3. 本次实测中 lock 文件路径为：
  - `/home/gyk25/chipyard/.git/modules/software/firemarshal/modules/riscv-linux/index.lock`
  且未检测到对应活动 git 进程，判定为“陈旧锁”。

### 18.3 解决方案

#### 方案 18-A（推荐）安全清理陈旧锁并重试

```bash
cd /home/gyk25/chipyard

LOCK=/home/gyk25/chipyard/.git/modules/software/firemarshal/modules/riscv-linux/index.lock

# 1) 先确认没有活跃 git 进程
ps -ef | grep git | grep -v grep

# 2) 若确认无活跃进程，再删除陈旧锁
rm -f "$LOCK"

# 3) 重试 Step 8
./build-setup.sh riscv-tools -s 1 -s 2 -s 3 -s 4 -s 5 -s 6 -s 7
```

#### 方案 18-B（更稳）先单独恢复 FireMarshal 子模块

```bash
cd /home/gyk25/chipyard/software/firemarshal
rm -f /home/gyk25/chipyard/.git/modules/software/firemarshal/modules/riscv-linux/index.lock
./init-submodules.sh
```

成功后回根目录继续执行 Step 8。

### 18.4 验证标准

1. 不再出现 `index.lock: File exists`。
2. Step 8 可继续到实际拉取/checkout 阶段。
3. `build-setup.sh` 不再以 step 8 / exit code 1 立即退出。

## 19. 最新问题复盘（Step 8 / SSL timeout + promisor fetch 失败）

### 19.1 精确定位（build-setup.log）

本轮最新 `build-setup.log` 关键内容：

1. 进入 Step 8：`BEGINNING STEP 8: Setting up FireMarshal`
2. 失败发生在 FireMarshal 子模块初始化命令阶段：
  - `git submodule update --progress --filter=tree:0 --init ...`
3. 关键报错链：
  - `error: RPC failed; curl 28 SSL connection timeout`
  - `fatal: unable to write request to remote: Broken pipe`
  - `fatal: could not fetch ... from promisor remote`
  - `fatal: Unable to checkout ... in submodule path 'boards/default/linux'`
4. 外层退出：
  - `build-setup.sh: Build script failed with exit code 128 at step 8`

### 19.2 根因分析

1. 这次不是证书链错误（`self-signed certificate`），而是连接超时（`curl 28`）。
2. 出错对象仍是大子模块 `boards/default/linux`，网络稳定性要求最高。
3. 使用 `--filter=tree:0` 时属于部分克隆，后续对象依赖 promisor remote 按需获取；当网络超时/断流时，更容易出现 `promisor remote` 相关失败。
4. `Broken pipe` 表示连接中途被远端/中间设备/本地网络栈中断，属于传输层问题。

### 19.3 解决方案（按推荐顺序）

#### 方案 19-A（推荐）降低网络中断敏感性后重试 Step 8

```bash
cd /home/gyk25/chipyard

# 1) 使用 HTTP/1.1，降低部分代理/网关下的 HTTP2 兼容性问题
git config --global http.version HTTP/1.1

# 2) 关闭低速超时触发，避免大仓库慢链路被误判中断
git config --global http.lowSpeedLimit 0
git config --global http.lowSpeedTime 999999

# 3) 子模块串行抓取，降低并发流量峰值
git config --global submodule.fetchJobs 1

# 4) 重试 Step 8
./build-setup.sh riscv-tools -s 1 -s 2 -s 3 -s 4 -s 5 -s 6 -s 7
```

#### 方案 19-B（更稳）先单独拉取 Linux 子模块，再继续 build-setup

```bash
cd /home/gyk25/chipyard/software/firemarshal

# 确保没有陈旧锁
rm -f /home/gyk25/chipyard/.git/modules/software/firemarshal/modules/riscv-linux/index.lock

# 先单拉最重的 linux 子模块（单线程 + HTTP/1.1）
git -c http.version=HTTP/1.1 -c submodule.fetchJobs=1 \
  submodule update --init --progress boards/default/linux

# 再补齐其余子模块
./init-submodules.sh
```

然后回到根目录继续 Step 8。

#### 方案 19-C（必要时）避免 promisor 路径，改为完整拉取

当网络对部分克隆极不稳定时，临时避免 `--filter=tree:0`：

1. 在 `software/firemarshal/init-submodules.sh` 中临时去掉 `--filter=tree:0`。
2. 执行一次完整子模块拉取后再恢复脚本。

说明：完整拉取数据量更大，但可减少 promisor 后续补抓失败概率。

### 19.4 验证标准

1. `build-setup.log` 不再出现以下错误：
  - `curl 28 SSL connection timeout`
  - `unable to write request to remote: Broken pipe`
  - `could not fetch ... from promisor remote`
2. `boards/default/linux` 能成功 checkout 到目标 commit。
3. Step 8 不再以 exit code 128 退出。

### 19.5 工程建议

1. 对大仓库（如 Linux）优先采用“单仓库先成功、再全量”的策略。
2. 在不稳定网络环境中，将 `submodule.fetchJobs` 固定为 1。
3. 为 Step 8 增加自动重试包装（至少 2 次）。

## 20. 19-A 执行后仍失败的复盘（Step 8 持续超时）

### 20.1 最新日志定位

在按 19-A 执行后，`build-setup.log` 仍在 Step 8 失败，关键报错为：

1. `fatal: unable to access 'https://github.com/firesim/linux.git/': SSL connection timeout`
2. `fatal: could not fetch ... from promisor remote`
3. `fatal: Unable to checkout ... in submodule path 'boards/default/linux'`
4. `build-setup.sh: Build script failed with exit code 128 at step 8`

### 20.2 根因更新

1. 证书链问题已不是主因；当前是“到 `firesim/linux` 的长连接超时”。
2. 失败仍集中在大仓库 `boards/default/linux`，传输窗口长，最容易触发超时。
3. promisor 路径（部分克隆）在不稳定链路下会进一步放大失败概率。

### 20.3 新方案（本轮推荐）

#### 方案 20-A（推荐）默认禁用 filter，避免 promisor 路径

已在 `software/firemarshal/init-submodules.sh` 实施：

1. 默认不传 `--filter=tree:0`。
2. 仅在显式设置 `FIREMARSHAL_USE_FILTER=1` 且 Git 支持时才启用 filter。

这样可降低 promisor 按需补抓导致的 checkout 失败概率。

#### 方案 20-B 单独拉取 linux 子模块并重试（稳定优先）

```bash
cd /home/gyk25/chipyard/software/firemarshal

# 清理可能的陈旧锁
rm -f /home/gyk25/chipyard/.git/modules/software/firemarshal/modules/riscv-linux/index.lock

# 单线程、HTTP/1.1，先只处理最重子模块
git -c http.version=HTTP/1.1 -c submodule.fetchJobs=1 \
  submodule update --init --progress boards/default/linux

# 成功后补齐其余子模块
./init-submodules.sh
```

#### 方案 20-C 若仍超时，使用 SSH 通道绕过 HTTPS 超时链路

前提：SSH key 已配置并可访问 GitHub。

```bash
cd /home/gyk25/chipyard/software/firemarshal
git config --local submodule.riscv-linux.url git@github.com:firesim/linux.git
git submodule sync --recursive
git submodule update --init --progress boards/default/linux
```

### 20.4 验证标准

1. Step 8 日志不再出现 `SSL connection timeout`。
2. `boards/default/linux` 成功 checkout 到指定 commit。
3. `build-setup.sh` 能从 Step 8 继续向后执行。

### 20.5 回退说明

若后续想恢复部分克隆优化，可显式启用：

```bash
cd /home/gyk25/chipyard/software/firemarshal
FIREMARSHAL_USE_FILTER=1 ./init-submodules.sh
```

## 21. 20-A 后仍失败的根因收敛（历史 promisor 状态残留）

### 21.1 最新日志特征

在 20-A（默认禁用 `--filter`）后再次执行 Step 8，`build-setup.log` 新报错为：

1. `error: RPC failed; HTTP 408 curl 22 The requested URL returned error: 408`
2. `fatal: expected 'packfile'`
3. `fatal: could not fetch ... from promisor remote`
4. `fatal: Unable to checkout ... in submodule path 'boards/default/linux'`

### 21.2 关键排查证据

已确认 `init-submodules.sh` 本次调用中 `FILTER_ARGS=()`，即没有再传 `--filter=tree:0`。

但进一步检查发现 `boards/default/linux` 对应 gitdir 中仍保留历史 partial clone 元数据：

1. `remote.origin.promisor=true`
2. `remote.origin.partialclonefilter=tree:0`

路径：

1. `/home/gyk25/chipyard/.git/modules/software/firemarshal/modules/riscv-linux`

因此，即使当前命令不再显式带 `--filter`，该子模块仍按 promisor 方式取对象，网络波动（HTTP 408）时继续触发 `packfile/promisor` 失败。

### 21.3 根因结论

主根因不是 20-A 逻辑失效，而是“历史 partial clone 状态未被重置”。

20-A 只影响“新一次 update 命令参数”，不会自动清除已存在子模块 gitdir 的 promisor 配置。

### 21.4 解决方案（推荐）

#### 方案 21-A（推荐）彻底重建 `boards/default/linux` 子模块（去 promisor）

```bash
cd /home/gyk25/chipyard

# 1) 反初始化并清理工作树
git -C software/firemarshal submodule deinit -f boards/default/linux
rm -rf software/firemarshal/boards/default/linux

# 2) 清理对应子模块 gitdir（关键）
rm -rf .git/modules/software/firemarshal/modules/riscv-linux

# 3) 同步配置并重新初始化（此时不走 filter）
git -C software/firemarshal submodule sync --recursive
git -C software/firemarshal -c http.version=HTTP/1.1 -c submodule.fetchJobs=1 \
  submodule update --init --progress boards/default/linux

# 4) 补齐其余 FireMarshal 子模块
cd software/firemarshal
./init-submodules.sh
```

#### 方案 21-B（增强稳定性）增加重试包装

```bash
cd /home/gyk25/chipyard/software/firemarshal
for i in 1 2 3; do
  git -c http.version=HTTP/1.1 -c submodule.fetchJobs=1 \
    submodule update --init --progress boards/default/linux && break
  echo "retry $i failed, sleep 20s"
  sleep 20
done
```

### 21.5 验证标准

1. `git --git-dir=.git/modules/software/firemarshal/modules/riscv-linux config --get remote.origin.promisor` 无输出。
2. `build-setup.log` 不再出现 `promisor remote` / `expected 'packfile'` / `HTTP 408`。
3. Step 8 成功通过，流程继续推进。

## 22. Step 8 已拆分脚本说明

为便于定位 FireMarshal 阶段问题，已将 `build-setup.sh` 中 step 8/9 逻辑拆分为独立脚本：

1. `scripts/firemarshal-setup.sh`

支持分阶段运行：

1. 只跑 step 8 子模块初始化：

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-env
./scripts/firemarshal-setup.sh --phase init
```

2. 只跑 step 9 预编译：

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-env
./scripts/firemarshal-setup.sh --phase precompile
```

3. 一次性跑 init + precompile：

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-env
./scripts/firemarshal-setup.sh --phase all
```

与 `build-setup.sh` 的对应关系：

1. Step 8 调用：`firemarshal-setup.sh --phase init`
2. Step 9 调用：`firemarshal-setup.sh --phase precompile`

这样在出现问题时可以精确缩小范围到 init 或 precompile 子阶段。

## 23. 分段执行实测记录（init / precompile）

按要求分别执行了两段：

1. `./scripts/firemarshal-setup.sh --phase init`
2. `./scripts/firemarshal-setup.sh --phase precompile`

并将日志分别落盘为：

1. `firemarshal-init.log`（对应退出码文件：`firemarshal-init.exit`）
2. `firemarshal-precompile.log`（对应退出码文件：`firemarshal-precompile.exit`）

### 23.1 init 阶段结果

1. 退出码：`1`
2. 日志行数：`1`
3. 日志内容：

```text
/home/gyk25/chipyard/.conda-env/etc/conda/activate.d/activate-riscv-tools.sh: line 65: RISCV: unbound variable
```

### 23.2 precompile 阶段结果

1. 退出码：`1`
2. 日志行数：`1`
3. 日志内容：

```text
/home/gyk25/chipyard/.conda-env/etc/conda/activate.d/activate-riscv-tools.sh: line 65: RISCV: unbound variable
```

### 23.3 问题归因

两段均在“conda activate 阶段”提前退出，尚未进入 FireMarshal 业务逻辑。

关键点：

1. `firemarshal-setup.sh` 使用了 `set -euo pipefail`。
2. `conda activate` 期间会 source `activate-riscv-tools.sh`。
3. 在 `set -u`（nounset）打开时，`activate-riscv-tools.sh` 中对未定义 `RISCV` 的引用触发 `unbound variable`，导致脚本立即退出。

### 23.4 对应解决方案

推荐在 `scripts/firemarshal-setup.sh` 中对 `conda activate` 做“局部关闭 nounset”保护：

```bash
set +u
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate "$CONDA_ENV_PATH"
set -u
```

备选方案：在激活前先定义默认 `RISCV`，例如：

```bash
export RISCV="/home/gyk25/chipyard/.conda-env/riscv-tools"
```

### 23.5 下一步建议

1. 先应用 23.4 的脚本修复。
2. 重新分别执行 init/precompile 两段。
3. 更新日志（`firemarshal-init.log` / `firemarshal-precompile.log`）并继续定位 Step 8 内部问题。

## 24. 分段复测结果（修复 conda 激活后）

在修复 `firemarshal-setup.sh` 的 conda 激活逻辑后，再次分别执行：

1. `./scripts/firemarshal-setup.sh --phase init`
2. `./scripts/firemarshal-setup.sh --phase precompile`

结果如下。

### 24.1 init 阶段

1. 退出码：`128`
2. 关键错误：

```text
fatal: Unable to create '/home/gyk25/chipyard/.git/modules/software/firemarshal/modules/boards/firechip/drivers/iceblk-driver/index.lock': File exists.
fatal: Unable to checkout 'e2b1d25aef89601d52a70784401796a1d2845721' in submodule path 'boards/firechip/drivers/iceblk-driver'
```

结论：

1. `RISCV: unbound variable` 问题已不再出现。
2. 当前新阻塞点是 FireMarshal 子模块 `iceblk-driver` 的陈旧 `index.lock`。

### 24.2 precompile 阶段

1. 退出码：`1`
2. 日志显示已进入 FireMarshal build 流程，随后：`Received SIGINT`

结论：

1. `precompile` 阶段未出现明确的新编译错误指纹。
2. 本次失败是外部中断（SIGINT），不能据此判定 precompile 本身存在构建故障。

### 24.3 根因与修复方案

#### 根因

1. 历史中断残留 `index.lock`，导致 `git submodule update` 无法继续 checkout。

#### 方案 24-A（推荐）清理陈旧锁并重跑 init

```bash
cd /home/gyk25/chipyard

# 1) 确认没有活跃 git 进程
ps -ef | grep git | grep -v grep

# 2) 清理本次卡住的 lock
rm -f /home/gyk25/chipyard/.git/modules/software/firemarshal/modules/boards/firechip/drivers/iceblk-driver/index.lock

# 3) 重跑 init
./scripts/firemarshal-setup.sh --phase init
```

#### 方案 24-B（更稳）批量清理 FireMarshal 子模块残留锁

```bash
cd /home/gyk25/chipyard
find .git/modules/software/firemarshal -name index.lock -type f -print -delete
./scripts/firemarshal-setup.sh --phase init
```

### 24.4 验证标准

1. `init` 阶段不再出现 `index.lock: File exists`。
2. `init` 阶段完成后，再执行 `precompile`，且不人为中断（避免 SIGINT）。
3. `precompile` 日志中出现正常完成标记（或至少进入稳定的编译执行阶段且无 fatal）。

## 25. 已落地修复（precompile 前清理 LD_LIBRARY_PATH）

基于 `br-base` 日志中 Buildroot 报错：

1. `You seem to have the current working directory in your LD_LIBRARY_PATH environment variable.`

已在 `scripts/firemarshal-setup.sh` 中实现自动修复：

1. 新增 `sanitize_ld_library_path()`。
2. 在 `run_precompile_phase()` 中调用该函数。
3. 自动移除 `LD_LIBRARY_PATH` 中的空元素（例如 `::`）和 `.`，避免被 Buildroot 识别为“当前目录污染”。

实现效果：

1. 用户无需手工 `unset LD_LIBRARY_PATH`。
2. precompile 阶段默认以更干净的动态库搜索路径执行。

建议复测命令（在完备终端执行）：

```bash
cd /home/gyk25/chipyard
source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /home/gyk25/chipyard/.conda-env

./scripts/firemarshal-setup.sh --phase precompile > firemarshal-precompile.log 2>&1
echo $? > firemarshal-precompile.exit
```
