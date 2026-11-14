# leopard-packages

LeopardOS 源包/仓库层，参考 Void 的 `void-packages` 模型。

当前是过渡阶段：

- `media.list`：安装介质 `/media` 仓库只包含这些目标系统包；
- `sources.conf`：包名到组件 `out/` 目录的映射；
- `leopard-src sync`：从组件 `out/` 取**每个包最新的一个** `.xbps`，清空并重建
  `hostdir/binpkgs`，然后生成 `x86_64-repodata`；
- `leopard-src repo`：只重建索引（`xbps-rindex -c` / `-r`）。

后续会加入 `srcpkgs/<name>/template`，由 `leopard-src pkg <name>` 从源码构建，
取代 `sync` 的过渡模式。

```sh
./leopard-src sync
./leopard-src list
./leopard-src repo
```
