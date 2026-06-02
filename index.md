
# 施工中!

## 还在施工中！

### 永远在施工中

- source1: [nodes]()
- source2: 


### live source as belows

source1: ~~[河南科技大学教育网.txt](https://faith-mian.github.io/2020-11-26河南科技大学教育网.txt) **360p less**~~

source2: ~~[全网直播源.txt](faith-mian.github.io/全网直播源.txt) )**HD grouped**~~

source3: ~~[4K频道.txt](faith-mian.github.io/4K频道.txt)**4KHD**~~



# 正文

这段更新日志主要涉及 Android 逆向、Hook 框架以及对抗加固和深度环境检测的技术细节。原文已经是繁体中文，如果你需要将其转换为简体中文，以下是翻译好的版本：

665-668 版本更新说明

将冲突检查延后到启动初始化之后执行，确保启动流程正确
修正 LSPlant 的 rwxp 内存权限泄漏所导致的环境异常问题
提升 seccomp 重导稳定性并更新 NPatch 的 LSPlant 兼容性钩子
加强 I/O 与路径检测绕过，补齐 APK 可见路径与 fd 挂载信息伪装

更多说明：

这次更新主要针对的是目前 Android 生态圈中中高强度的加固方案（如 360 加固等）以及深度的环境检测框架（如 Smallcheck）。这些检测手段不再仅限于简单的包名比对或 root 侦测，而是深入到 Linux 核心层与 ART 执行线程。

以下为具体“过掉”（绕过）的检测机制解析：

提升 seccomp 重导稳定性并更新 LSPlant 兼容性钩子

这个部分的更新主要解决了主动探测防御与 ART 执行线程逃逸的问题：

过掉“探测性崩溃（Crash-based Probing）检测”：
    检测原理： 侦测会故意向 openat 等系统调用传入“畸形的路径”或“不可读的内存指针”。如果你的 seccomp 拦截器（SIGSYS handler）直接去读取（解引用）这些指针，就会触发 Segmentation Fault，导致外挂进程崩溃，或者暴露出有工具在拦截 syscall 的事实。
    绕过方式： 更新后，不再于 SIGSYS 中直接解引用指针，而是先让核心去验证该路径是否合法并打开它，再通过 /proc/self/fd 解析回传的 fd。这让恶意探测无法引发拦截器崩溃，完美隐蔽了 seccomp 重导的存在。

过掉“ART 内联优化逃逸（Inline / AOT Escape）检测”：
    检测原理： 现代 Android (特别是 Android 8.0+) 的 ART 虚拟机会频繁利用 ProfileSaver 进行 AOT (Ahead-of-Time) 编译，并将部分频繁调用的方法“内联 (Inline)”。一旦目标方法被内联，调用者会直接执行机器码，不再经过方法的入口点，这会导致 LSPlant 等依赖入口点替换的 Java Hook 失效。防护系统会利用这一点，通过频繁触发特定逻辑并检查执行时间或路径，来判断是否被 Hook。
    绕过方式： 通过安装 ProfileSaver 兼容性钩子并强制对 dex2oat 关闭 inline，确保目标方法永远不会被编译成无法拦截的内联状态。这保证了 Hook 的绝对覆盖率，过掉了依赖内联机制的完整性校验。

加强 I/O 与路径检测绕过，补齐 APK 可见路径与 fd 挂载信息伪装

这部分是针对 VFS (虚拟文件系统) 级别的深度稽查，特别是 Smallcheck 这类极度吹毛求疵的环境检测工具：

过掉“Java 层 API 交叉比对检测”：
    检测原理： 应用不仅会通过 Context.getPackageCodePath() 获取 APK 路径，还会深入反射读取 LoadedApk、BoundApplication 内的 ApplicationInfo.sourceDir 等字段，甚至通过 java.io.File API 进行交叉比对。如果重导逻辑只覆盖了表层，底层字段仍指向你 patch 过的私有目录，就会被抓包。
    绕过方式： 启动后主动将这些深层框架变量恢复为原始 APK 的“可见路径”，并伪装 File API，让所有 Java 层的路径查询都得到一致的假象。

过掉“底层 Metadata (中介资料) 检测”：
    检测原理： 防护系统不调用 Java，直接通过 native 的 stat, lstat, statfs 或 statx 系统调用，去查验 APK 文件的“大小”、“Inode 节点”、“创建时间”或“磁区信息”。如果读取到的是补丁 APK 的信息，大小和时间绝对与原始 APK 不符。
    绕过方式： 将 native 文件 hook 扩充到所有涉及 file stat 的 syscall，强制回传原始 APK 的 Metadata，让文件属性在检测工具眼中完美无瑕。

过掉“挂载点特征 (Mount ID / FD) 深度比对”：
    检测原理： 这是 Smallcheck 的拿手好戏。即使你完美伪装了路径与大小，检测工具会去读取 /proc/self/fd/ 下打开的文件描述符，并进一步读取 /proc/*/fdinfo/<fd> 里面的 mnt_id (挂载点 ID)。如果你的补丁 APK 放在另一个分区（例如某个私有 data 目录或 OverlayFS），它的 mnt_id 会与预期中 /data/app/ 目录所在的 mnt_id 不同，重导行为直接曝光。
    绕过方式： 系统动态追踪被重导的 fd，并在检测读取 fdinfo 时，直接将其中的 mnt_id 覆写篡改为“可见路径（原始 APK）”应该要有的 mnt_id。这直接在 VFS 层面阻断了挂载点异常泄漏的问题。

过掉“虚拟文件系统 (FUSE) 探测”：
    检测原理： 检查 /dev/fuse 或相关挂载行为，来判断系统是否运作在被魔改的模块环境（某些模块会利用 FUSE 来做文件拦截）。
    绕过方式： 直接对 /dev/fuse 的查询回传“不存在”，减少异常挂载点暴露的面积。
