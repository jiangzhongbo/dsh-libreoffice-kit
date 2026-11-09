# Linux native 引擎在云 DSH 沙盒（1 CPU / 1 GiB）的适配报告

日期：2026-11-09

## 结论

| 项 | 结果 |
|---|---|
| 1 CPU / 1 GiB 沙盒能否跑 DSH 运行时 | ✅ 可以 |
| 1 CPU / 1 GiB 沙盒能否跑文档转换（官方 wasm） | ❌ OOM Killed（需 1.2 GB） |
| 1 CPU / 1 GiB 沙盒能否跑文档转换（自建 native） | ✅ 可以（峰值 265 MB） |

修复的关键：**在 Debian 12（glibc 2.36）上重建引擎**。原 Ubuntu 24.04 构建需要 `GLIBC_2.38`，云沙盒是 glibc 2.36，kit 按 `prebuilds.json` 的 `glibcMinimum` 拒绝加载并静默回退 wasm，wasm 在 1 GiB 下必然 OOM。

## 新引擎

| 项 | 值 |
|---|---|
| 构建环境 | Debian 12 bookworm / glibc 2.36 / clang 19.1.7 |
| 产物 | `packages/linux-arm64-glibc`，837 文件，229 MB |
| `bin/libreoffice-kit` | 216,051,184 B（206 MiB） |
| sha256 | `4abd4fdc8ed44ca5f47962a39859fe63e5537b3a41dbe404a0f812585dae1aa1` |
| **glibcMinimum** | **2.35**（原 2.38） |
| 校验 | `scripts/verify-artifacts.mjs` 通过 |
| 构建耗时 | 约 35 分钟（-j12，峰值内存 3.0 GiB） |

旧包（Ubuntu 24.04 / glibc 2.38）备份在 `/tmp/pkg-ubuntu2404-backup`。

## 1c1g 实测（容器 `--cpus=1 --memory=1g --memory-swap=1g`）

### 启动
- 到监听端口 **1.36 s**
- 峰值 RSS **176 MB**
- CPU **1.39 s**（≈1 核打满）
- 构成：node 22 基线 ~45 MB + `dsh-base` / `dsh-web-app` 插件加载 + HTTP 服务

### 稳定（空闲 30 s）
- RSS **171 MB**
- CPU **1.4%**（30 s 内 0.43 s）

### 处理文件（native）

| 文档 | 耗时 | CPU | 进程峰值 RSS | 输出 |
|---|---|---|---|---|
| report-3p.docx（3 页） | 0.83 s | 0.82 s | 220 MB | 78 KB |
| report-30p.docx（30 页） | 2.36 s | 2.37 s | 232 MB | 123 KB |
| deck-30s.pptx（30 页） | 3.84 s | 3.86 s | 265 MB | 206 KB |

运行时 + 转换同时存在时，整容器 anon 峰值 **252 MB**。

### 对照：官方 wasm
```
wall=8612ms CPU=8616ms 峰值RSS=820MB  → Killed (OOM)
```
需要 1.2 GB，1 GiB 下无法完成。


## 极端文档压测（同一 1c1g 容器）

| 文档 | 规模 | 耗时 | CPU | 峰值 RSS | 输出 | 结果 |
|---|---|---|---|---|---|---|
| doc-200p.docx | 400 页文本 | 1.26 s | 1.26 s | 253 MB | 310 KB | ✅ |
| doc-images.docx | 20 × 3000×2000 图 | 2.64 s | 2.66 s | 241 MB | 111 KB | ✅ |
| doc-huge-image.docx | 单张 8000×8000 | 2.02 s | 2.00 s | **424 MB** | 85 KB | ✅ |
| sheet-50k.xlsx | 5 万行 × 8 列 | 4.09 s | 4.05 s | 299 MB | 3.3 MB | ✅ |
| sheet-200k.xlsx | 20 万行 × 8 列 | 19.3 s | 19.2 s | 332 MB | 13 MB | ✅ |
| deck-80s.pptx | 80 页 + 每页大图 | 12.4 s | 12.4 s | 301 MB | 220 KB | ✅ |
| deck-200s.pptx | 200 页 + 每页大图 | 30.4 s | 30.3 s | 369 MB | 291 KB | ✅ |

**wasm 对照**（同一容器、同一字体、隐藏 native 包）：

```
wasm: wall=3557ms cpu=3350ms peak=1050MB rc=137 (SIGKILL/OOM)
```

## 边界（kit 默认限制）

| 限制 | 默认值 | 说明 |
|---|---|---|
| maxInputBytes | 64 MiB | 超过直接拒绝 |
| maxOutputBytes | 128 MiB | 超过删除产物并报错 |
| timeoutMs | 120 s | 单次转换超时 |
| maxImageResolution | 144 DPI | 输出 PDF 图片降采样，所以 8000×8000 不会撑爆 |

结论：在默认限制内，**最坏实测 424 MB**，1 GiB 有约 2 倍余量；但并发两个大文档就会超。

## 还差一步：字体

沙盒镜像当前零字体，转换会报：
```
{"code":"unavailable","error":"No usable fonts were found."}
```
测试时临时装了 `fonts-dejavu-core`（6 个字体）才跑通；`missingFonts` 仍有 `Arial` / `Songti SC` / `Courier`。

生产建议在镜像里加：
- `fonts-dejavu-core`（~1 MB，兜底）
- `fonts-liberation`（Arial 度量兼容）
- `fonts-noto-cjk`（中文，~60 MB）

## 复现

见同目录 `Dockerfile.dsh-builder`。

```sh
docker build -f Dockerfile.dsh-builder -t lokit-builder:bookworm12 .
docker run --rm -v /path/to/libreoffice-kit:/src -w /src --user 1000:1000 \
  -e HOME=/tmp -e CC=clang-19 -e CXX=clang++-19 \
  -e CC_FOR_BUILD=clang-19 -e CXX_FOR_BUILD=clang++-19 \
  lokit-builder:bookworm12 bash -c 'git config --global --add safe.directory "*"; \
    node scripts/build-native.mjs --platform linux-arm64-glibc --jobs 12 \
      --build /src/.build/native-linux-arm64-glibc-bb --tarballs /src/.build/tarballs'
```

注意：LibreOffice 26.8 要求 clang ≥ 18，bookworm 的 clang-16 不够，需 clang-19；build 侧默认 gcc-12 也会被拒绝（要求 GCC ≥ 13），必须把 `CC_FOR_BUILD` / `CXX_FOR_BUILD` 一并指向 clang-19。
