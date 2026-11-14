# leopard-packages

LeopardOS 源包/仓库层，参考 Void 的 `void-packages` 模型。

当前是过渡阶段：

- `media.list`：安装介质 `/media` 仓库只包含这些目标系统包；
- `sources.conf`：包名到组件 `out/` 目录的映射；
- `components.lock`：组件仓库的 `tag + rev` 锁定，`rev` 权威；
- `leopard-src sync`：从组件 `out/` 取**每个包最新的一个** `.xbps`，清空并重建
  `hostdir/binpkgs`，然后生成 `x86_64-repodata`；
- `leopard-src repo`：只重建索引（`xbps-rindex -c` / `-r`）；
- `leopard-src lock`：查看锁定状态；`lock --update` 刷新 tag/rev；
- `leopard-src verify`：校验 tag 指向 rev、HEAD 与 lock 一致、media 包齐全。

后续会加入 `srcpkgs/<name>/template`，由 `leopard-src pkg <name>` 从源码构建，
取代 `sync` 的过渡模式。

```sh
./leopard-src sync
./leopard-src list
./leopard-src repo
./leopard-src lock
./leopard-src verify          # 开发检查：untagged 只警告
./leopard-src verify --release # 发版检查：untagged 直接失败
```

## tag + rev

`components.lock` 里：

```toml
[OpenXSKernel]
repo = "git@github.com:leopard-project/OpenXSKernel.git"
path = "../OpenXSKernel"
tag = "v2.1.0"
rev = "7a190ce2a452ab0164fd7a05e57a3b6bef7f026f"
```

- `tag` 是人读的版本标签；
- `rev` 是构建真正 checkout 的 commit；
- `verify` 会检查 `tag` 是否仍指向 `rev`，tag 被移动就失败；
- 组件还没有 tag 时，`verify` 给警告；`verify --release` 会失败。

发版流程：

```sh
# 组件仓库
git -C ../OpenXSKernel tag -s v2.1.0 -m "OpenXSKernel 2.1.0"

# 刷新 lock（会读取每个组件 HEAD 的 exact tag）
./leopard-src lock --update

# 开发/发版校验
./leopard-src verify
./leopard-src verify --release

# 重建本地仓库
./leopard-src sync
./leopard-src repo
```
