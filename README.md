# leopard-packages

LeopardOS 的**源包 / 本地二进制仓库层**。参考 Void 的 `void-packages` 模型，但不依赖远程包服务器。

当前职责：

- 定义安装介质 `/media` 里允许出现哪些包（`media.list`）；
- 从各组件 `out/` 收拢受控的本地仓库（`leopard-src sync`）；
- 锁定组件源码版本（`components.lock`，`tag + rev`）；
- 生成 / 清理 / 签名 `x86_64-repodata`；
- 打一个可通过即时通信软件手动分发的仓库包（`leopard-src bundle`）。

它不负责：ISO 引导、PE 组装、QEMU 测试。那些在 `leopard-os`。

目录：

1. 仓库结构
2. 主机依赖
3. 首次准备
4. 手动构建发行版
5. 管理发行版
6. 版本锁定：tag + rev
7. 签名
8. 手动分发（即时通信软件）
9. 发布流程（手动）
10. 常见问题
11. 当前限制
12. 命令速查

---

## 1. 仓库结构

```
leopard-project/
├── OpenXSKernel/          # 组件（上游）
├── OpenXSBoot/
├── XJ380-SDK/
├── XXCC-suite/
├── leopard-base/
├── leopard-rc/
├── leopard-getty/
├── leopard-oobe/
├── leopard-shell/
├── leopard-mountd/
├── leopard-utils/
├── leopard-coreutils/
├── leopard-xbps/
├── leopard-installer/
├── leopard-packages/      # 本仓库：源包 / 仓库层
│   ├── media.list
│   ├── sources.conf
│   ├── components.lock
│   ├── leopard-src
│   ├── etc/
│   ├── srcpkgs/           # 预留：未来 template 构建
│   └── hostdir/
│       └── binpkgs/       # 生成物：.xbps + x86_64-repodata
└── leopard-os/            # 镜像层：PE、ISO、预装镜像、QEMU 测试
    ├── live/pe.list
    ├── build_liveroot.sh
    ├── make_iso.sh
    └── img-tools/
```

依赖方向：

```
组件仓库 → 构建 → 各组件 out/*.xbps
                         ↓
              leopard-packages/leopard-src sync
                         ↓
              hostdir/binpkgs + x86_64-repodata
                         ↓
              leopard-os/build_liveroot.sh + make_iso.sh
                         ↓
                    ISO /media
                         ↓
              xbps-install -r /mnt -R /media
```

包层只产出仓库；镜像层只消费仓库。不要把镜像/QEMU 逻辑写进本仓库。

---

## 2. 主机依赖

基础：

- `git`
- `python3`
- `ninja`
- `clang` / `ld`
- `tar`、`sha256sum`（`bundle` 需要）
- `open`/`openssl`（签名时需要）

`xbps` 工具：

```sh
cd ../leopard-xbps
./fetch.sh      # 首次
./build.sh      # 产出 out/xbps-create、xbps-install、xbps-rindex ...
```

镜像 / 测试：

- `xorriso`、`dosfstools`、`mtools`
- `sgdisk`、`mkfs.ext4`
- `qemu-system-x86_64`、OVMF（`/usr/share/ovmf/OVMF.fd`）

---

## 3. 首次准备

把组件仓库和本仓库放在同一个父目录下（兄弟目录）：

```sh
mkdir -p ~/Projects/leopard-project
cd ~/Projects/leopard-project

for r in \
  OpenXSBoot OpenXSKernel XJ380-SDK XXCC-suite \
  leopard-base leopard-rc leopard-getty leopard-oobe \
  leopard-shell leopard-mountd leopard-utils leopard-coreutils \
  leopard-xbps leopard-installer leopard-packages leopard-os
do
  git clone "git@github.com:leopard-project/$r.git"
done
```

`XJ380-SDK` 有第三方 submodule：

```sh
cd XJ380-SDK
git submodule update --init --recursive
```

构建 `xbps`：

```sh
cd ../leopard-xbps
./fetch.sh
./build.sh
```

---

## 4. 手动构建发行版

### 4.1 构建所有组件包

```sh
cd ~/Projects/leopard-project/leopard-os
sh img-tools/build_all.sh
```

它会：

1. 构建 `XJ380-SDK` + CRT，并打 `xact-crt` 包；
2. 按 `img-tools/packages.conf` 打包内核、引导、用户态组件；
3. 构建树内 kmod（`e1000-kmod`、`netserver-kmod`、`xhci-kmod`）；
4. 构建 `leopard-installer`（只进 PE，不装进目标系统）。

产物在各组件的 `out/*.xbps`。

只构建单个组件：

```sh
cd OpenXSKernel
python3 tools/gen_ninja.py
ninja
ninja pack          # 内核包
ninja pack-kmods    # e1000/netserver/xhci 模块包
```

### 4.2 生成受控本地仓库

```sh
cd ../leopard-packages
./leopard-src sync
```

`sync` 会：

- 清空 `hostdir/binpkgs`；
- 按 `media.list` 每个包只取最新一份 `.xbps`；
- 调用 `xbps-rindex -a`；
- 调用 `-c` / `-r` 清理索引；
- 生成 `hostdir/binpkgs/x86_64-repodata`。

查看：

```sh
./leopard-src list
ls hostdir/binpkgs
```

校验：

```sh
./leopard-src verify
```

开发阶段组件可能未打 tag，`verify` 只给 `untagged` 警告；正式发版用：

```sh
./leopard-src verify --release
```

### 4.3 生成安装介质 ISO

```sh
cd ../leopard-os
sh build_liveroot.sh
sh make_iso.sh
```

`build_liveroot.sh` 会：

- 调用 `leopard-packages/leopard-src sync`；
- 按 `live/pe.list` 复制最小 PE 文件；
- 按 `leopard-packages/media.list` 从 `hostdir/binpkgs` 复制 `/media` 包；
- 写入 live `initrc`、安装选项、权限配置。

产物：

```
leopard-os/leopard-installer.iso
```

### 4.4 运行全链路测试

```sh
cd ../leopard-os
BUILD=0 WORK=/tmp/leopard-test TIMEOUT=420 sh test_all.sh
```

预期：

```
安装 10/10
启动 3/3
```

### 4.5 生成预装镜像（可选）

注意：预装镜像目前仍走 `img-tools/packages.conf`，不是 `media.list`。

```sh
cd ../leopard-os
sh img-tools/make_image.sh
# 或者：
sh img-tools/build_all.sh
sh img-tools/assemble_image.sh
```

产物：

```
img-tools/XJ380.img
img-tools/XJ380.vmdk
```

---

## 5. 管理发行版

### 5.1 增加一个包到 `/media`

1. 在 `media.list` 增加包名；
2. 如果它是新组件，在 `sources.conf` 增加：

   ```conf
   <pkgname> <repo>/out
   ```

3. 重建：

   ```sh
   ./leopard-src sync
   ./leopard-src verify
   ```

### 5.2 从 `/media` 移除一个包

1. 从 `media.list` 删除；
2. `./leopard-src sync`。

`sync` 会清空 `binpkgs` 重建，不会留下旧包。

### 5.3 更新组件

```sh
cd ../OpenXSKernel
ninja pack
cd ../leopard-packages
./leopard-src sync
```

如果组件版本号没变，`xbps-rindex` 仍会接受新文件；但正式发布建议 bump `revision`。

### 5.4 旧包 / 废弃包

不要手动 `cp` 到 `hostdir/binpkgs`。统一走：

```sh
./leopard-src sync
./leopard-src repo
```

`repo` 会调用：

```sh
xbps-rindex -c hostdir/binpkgs
xbps-rindex -r hostdir/binpkgs
```

清理索引和 obsolete 包。

### 5.5 查看仓库内容

```sh
xbps-query -R --repository=hostdir/binpkgs -s <name>
xbps-query -R --repository=hostdir/binpkgs -S <pkgname>
```

### 5.6 目标系统离线仓库

把 bundle 解到目标盘：

```sh
mkdir -p /system/repo
tar -xzf leopard-repo-YYYYMMDD.tar.gz -C /system/repo
mkdir -p /config/xbps.d
echo 'repository=/system/repo' > /config/xbps.d/repo.conf
xbps-install -S
xbps-install <package>
```

目标盘检查：

```sh
xbps-query -l
xbps-pkgdb -a
```

---

## 6. 版本锁定：tag + rev

`components.lock`：

```toml
[OpenXSKernel]
repo = "git@github.com:leopard-project/OpenXSKernel.git"
path = "../OpenXSKernel"
tag = "v2.1.0"
rev = "7a190ce2a452ab0164fd7a05e57a3b6bef7f026f"
```

规则：

- `tag` 是人读版本；
- `rev` 是真正 checkout 的 commit；
- `verify` 检查 tag 是否仍指向 rev；
- `verify --release` 要求每个组件都有 exact tag。

组件打 tag：

```sh
git -C ../OpenXSKernel tag -s v2.1.0 -m "OpenXSKernel 2.1.0"
git -C ../OpenXSBoot   tag -s v2.1.0 -m "OpenXSBoot 2.1.0"
```

`-s` 是 GPG 签名 tag；没有配置 GPG 时用 annotated tag：

```sh
git -C ../OpenXSKernel tag -a v2.1.0 -m "OpenXSKernel 2.1.0"
```

打完推送到远端：

```sh
git -C ../OpenXSKernel push origin v2.1.0
git -C ../OpenXSBoot   push origin v2.1.0
```

刷新 lock：

```sh
./leopard-src lock --update
```

查看：

```sh
./leopard-src lock
```

校验：

```sh
./leopard-src verify
./leopard-src verify --release
```

开发阶段可以没有 tag；正式发版不能。

---

## 7. 签名

生成 RSA PEM 私钥：

```sh
openssl genrsa -des3 -out privkey.pem 4096
# 或
ssh-keygen -t rsa -b 4096 -m PEM -f privkey.pem
```

签名仓库：

```sh
export LEOPARD_SIGN_KEY="$PWD/privkey.pem"
export LEOPARD_SIGNEDBY="LeopardOS"
./leopard-src repo
```

等价于：

```sh
xbps-rindex --privkey privkey.pem --sign --signedby "LeopardOS" hostdir/binpkgs
xbps-rindex --privkey privkey.pem --sign-pkg hostdir/binpkgs/*.xbps
```

公钥要单独分发给用户，并导入 xbps keyring。没有签名时，至少用 `SHA256SUMS` 校验传输完整性。

---

## 8. 手动分发（即时通信软件）

没有远程包服务器时，用 `bundle`：

```sh
./leopard-src bundle
# → hostdir/leopard-repo-YYYYMMDD.tar.gz
# → 打印 size 和 sha256
```

包内：

```
*.xbps
x86_64-repodata
SHA256SUMS
README.txt
media.list
components.lock
```

发送压缩包 + sha256。对方：

```sh
sha256sum leopard-repo-YYYYMMDD.tar.gz
mkdir -p /system/repo
tar -xzf leopard-repo-YYYYMMDD.tar.gz -C /system/repo
cd /system/repo
sha256sum -c SHA256SUMS
mkdir -p /config/xbps.d
echo 'repository=/system/repo' > /config/xbps.d/repo.conf
xbps-install -S
xbps-install <package>
```

聊天软件限制大小时：

```sh
BUNDLE_SPLIT_MB=8 ./leopard-src bundle
```

会生成：

```
leopard-repo-YYYYMMDD.tar.gz.part-00
leopard-repo-YYYYMMDD.tar.gz.part-01
...
leopard-repo-YYYYMMDD.tar.gz.parts.sha256
```

对方：

```sh
cat leopard-repo-YYYYMMDD.tar.gz.part-* > leopard-repo.tar.gz
```

---

## 9. 发布流程（手动）

```sh
# 1. 组件发版
git -C ../OpenXSKernel tag -s v2.1.0 -m "OpenXSKernel 2.1.0"
git -C ../OpenXSBoot   tag -s v2.1.0 -m "OpenXSBoot 2.1.0"
# ... 其余组件

# 2. 构建所有包
cd ../leopard-os
sh img-tools/build_all.sh

# 3. 刷新 lock
cd ../leopard-packages
./leopard-src lock --update

# 4. 发版校验
./leopard-src verify --release

# 5. 重建仓库
./leopard-src sync
./leopard-src repo

# 6. 手动分发包
./leopard-src bundle

# 7. 构建 ISO
cd ../leopard-os
sh build_liveroot.sh
sh make_iso.sh

# 8. 全链路测试
BUILD=0 WORK=/tmp/leopard-release TIMEOUT=420 sh test_all.sh

# 9. 打发行版 tag
cd ../leopard-packages
git tag -s leopardos-2026.11 -m "LeopardOS 2026.11"
git push origin leopardos-2026.11
```

发布物建议一起归档：

- `leopard-installer.iso`
- `leopard-repo-*.tar.gz`
- `.sha256` 校验文件
- `components.lock`
- ISO 的 sha256

---

## 10. 常见问题

### `no .xbps for <pkg> in <dir>`

组件还没构建，或 `sources.conf` 路径不对。

```sh
cd ../<component>
ninja pack
```

### `expected 1 package in binpkgs, found 0`

先重建仓库：

```sh
./leopard-src sync
```

### `missing x86_64-repodata`

```sh
./leopard-src repo
```

### `verify` 报 `untagged`

开发阶段是警告；发版前给组件打 exact tag：

```sh
git -C ../<component> tag -s v<version> -m "..."
./leopard-src lock --update
```

### `HEAD ... != lock ...`

组件有新提交，但 lock 还是旧 commit。二选一：

```sh
./leopard-src lock --update       # 接受新 HEAD
# 或
git -C ../<component> checkout <rev>   # 回到锁定版本
```

### `xbps-rindex` 签名失败

- 只支持 RSA PEM 私钥；
- 私钥有 passphrase 时设置 `XBPS_PASSPHRASE`；
- 一个仓库只能用一个 RSA key。

### 聊天软件阻止 `.tar.gz`

用 `BUNDLE_SPLIT_MB` 拆分；仍然被阻止时，把扩展名临时改成 `.bin`，接收端改回并核对 sha256。

### `/media` 还在 live-root.img 里

当前 `leopard-os/build_liveroot.sh` 把 `/media` 放在 live root 内，启动时会和 PE 一起解到 RAM。真正“介质上的 `/media`”需要把 media 镜像从 `live-root.img` 拆出去，并补引导器/内核访问方式，尚未实现。

---

## 11. 当前限制

- 还没有 `srcpkgs/<name>/template`；`sync` 仍是过渡形态，从组件 `out/` 收包。
- 还没有 `leopard-src pkg` 从 `components.lock` 的 `rev` 构建。
- 没有远程包服务器；分发靠 `bundle`。
- 预装镜像仍走 `img-tools/packages.conf`，安装器 ISO 走 `media.list`，两份清单尚未统一。
- `/media` 仍在 `live-root.img` 内。

---

## 12. 命令速查

```sh
# 仓库
./leopard-src sync          # 从组件 out/ 重建受控仓库
./leopard-src repo          # 只重建/清理索引（可签名）
./leopard-src list          # 列出 media.list
./leopard-src bundle        # 打可分发压缩包
./leopard-src lock          # 查看 tag/rev
./leopard-src lock --update # 刷新 tag/rev
./leopard-src verify        # 开发校验（untagged 警告）
./leopard-src verify --release
./leopard-src clean         # 删除 hostdir

# 镜像
cd ../leopard-os
sh img-tools/build_all.sh   # 打包所有组件
sh build_liveroot.sh        # 最小 PE + /media
sh make_iso.sh              # 生成 ISO
sh test_all.sh              # 安装 + 启动测试
```
