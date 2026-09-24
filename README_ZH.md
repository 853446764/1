# DFReroot

通过 Dirty Frag 在 Android 上实现持久化 Root。
稳定的二阶段 Root 方案：先用其他漏洞（如 ghostlock）获取一次临时 Root，
持久化一个系统 UID 应用，之后用 Dirty Frag 完成所有后续 Root。

## 支持设备

仅在 Galaxy S26 OneUI 8.5（samsung/m1qjpnx/m1q:16/BP4A.251205.006/S942QOPU1AZDE_SJP1AZDE:user/release-keys）上测试，其他版本可能也适用。

## 背景

`ghostlock` 漏洞在 Android 上不太稳定，每次重启都要重新获取临时 Root，还要担心内核崩溃。本方案通过 Dirty Frag 实现舒适的二阶段持久化 Root。

## 工作流程

```
临时 Root (ghostlock) -> DFInstaller 注入密钥 -> 软重启
  -> 安装 DFReroot (android.uid.system) -> 执行 DirtyFrag -> Root (ksud)
```

1. **DFInstaller**（`com.polygraphene.df.installer`，普通应用）以 Root 身份编辑
   `/data/system/packages.xml`，将签名密钥插入 `android.uid.system` 共享用户的 `pastSigs`。
2. 软重启后 PMS 重新读取 `packages.xml`。
3. **DFReroot**（`com.polygraphene.df.reroot`，`sharedUserId="android.uid.system"`）
   以系统 UID 安装，重启后持久存在。
4. DFReroot 从 `system_server` 跳转至 `network_stack`（可 `dlopen` 且持有
   `CAP_NET_ADMIN`），通过 Dirty Frag 修补 vendor/libc/libc++，加载微型 LKM
   将 SELinux 设为宽容模式，最后延迟加载 **ksud**。

## 前提条件

1. 能获取临时 Root 的漏洞
2. 内核存在 Dirty Frag 漏洞

## 使用方法

1. 通过其他漏洞获取临时 Root（如 ghostlock）。
2. 从 [Release](https://github.com/polygraphene/DFReroot/releases) 安装 `df_installer_(版本).apk`，在 Root 管理器中授权。
3. 点击 **注入** -> **软重启**（重启框架；PMS 重新读取 `packages.xml`）。
4. 点击 **安装 DFReroot**（通过 `pm install`，重启后需再次 `su`）。
5. 打开 DFReroot，点击 **执行 DirtyFrag**。

`df_reroot.apk` 已内置于 `df_installer.apk`，无需单独下载。

## 卸载

1. 点击 **移除密钥** 撤销对 `packages.xml` 的修改。
2. 卸载 DFReroot 和 DFInstaller。

## 构建

```sh
# 生成 app/keystore.jks
$ ./create-keystore.sh
# 生成 df_reroot.apk + df_installer.apk
$ ANDROID_NDK_HOME=(NDK 路径) ANDROID_HOME=(SDK 路径) ./build.sh
```

LKM 重新构建需要 Docker（GKI DDK），见 `dirtyfrag-lkm/build.sh`。
从[我的 fork](https://github.com/polygraphene/KernelSU/tree/kdp-612-3.3.0) 的 kdp-612-3.3.0 分支构建 ksud。

## 项目结构

- `installer/` — DFInstaller：`PackagesXml`/`Abx`（packages.xml 读写，支持 ABX），
  `InjectMain`（Root `app_process` 入口），`SysKey`/`SigKey`（证书读取，兼容 v1/v2/v3），GUI。
- `app/` — DFReroot：`StageHop`（system_server -> network_stack），
  `DirtyFrag`（JNI 桥接），`KsudStage`，原生代码 `exp.c` + `stage1.S`（arm64），`dirtyfrag.ko`。
- `dirtyfrag-lkm/` — LKM 源码（解析 `kallsyms_lookup_name`，清除 `selinux_state.enforcing`）。
- `logtestexe/` — logcat-socket 测试程序。

## 注意事项

- `packages.xml` 首次操作时会备份为 `packages.xml.bak-df-installer`。

## 致谢

- [Dirty Frag by @V4bel](https://github.com/V4bel/dirtyfrag) - Dirty Frag 漏洞的发现与利用
- [LSPromise by @LSPosed](https://github.com/LSPosed/LSPromise) - 在 Android 上利用 Dirty Frag
- [AbxOverflow by @michalbednarski](https://github.com/michalbednarski/AbxOverflow) - 持久化 `system_server` 特权的思路
