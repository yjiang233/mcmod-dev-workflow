# 环境与构建排查

本文件记录本机 Windows 中文账户下反复踩过的坑，均为实测结论。**版本号、端口、代理地址会变，使用时以工程当前配置为准**（记录时间 2026-11-12）。

## 中文用户名导致的编码问题

- Windows 账户名含中文时，PATH 里会出现 U+FFFD 替换字符，Java 转 GBK 时 LWJGL 抛 `InvalidPathException`。
  - 处理：构建脚本里把原生 DLL 解到纯 ASCII 路径（例如 `build/natives`），并注入 `org.lwjgl.librarypath`；启动前过滤 PATH 中含 U+FFFD 的条目。
- Gradle worker 的 argfile 也会因中文路径损坏。ForgeGradle 6 工程（1.20.1）里所有 gradle 命令需把 `GRADLE_USER_HOME` 指向一个**纯 ASCII 路径**，否则单测/构建 worker 直接失败。JDK 用 toolchain 17 或 21 + `options.release=17`。

## 依赖与网络

- `maven.neoforged.net` 直连不通时，改用官方 CDN `neoforged.forgecdn.net/releases`；可在 `settings.gradle` 里重写指向官方仓库的 URL。
- CDN 可能漏掉纯 POM 聚合模块（例如 `net.neoforged:minecraft-dependencies`），需要本地 `local-maven/` 补一份 Mojang 官方 version json 重建的 POM，**勿删**。
- 环境变量里的全局代理可能已失效（表现为 CONNECT 502），而本机另一个端口的代理可用。仓库级 `git config --local http.proxy` 优先于环境变量，配好后直接 push 即可，不要每次手改环境变量。
- 第三方 mod jar 常只在 CurseForge / Modrinth 有；用官方 API 查 file id 再拼直链，慢时走代理。1.20.1 的 mod jar 多为 SRG 映射，compileOnly 直接引会编译不过，必须全部走 `fg.deobf(...)`。

## 启动与验证

- 开发启动若不带自动进世界参数，重启客户端会停在主菜单；需要手动进世界，服务端状态才为真。
- 内嵌 HTTP 服务端口被占用会自动顺延，实际端口写在运行目录的 `.port` 文件里。**排查「改了没效果」的第一动作**：查该端口的监听 PID，与进程列表比对，确认 curl 打到的是刚启动的新实例，而不是仍活着的旧客户端。旧进程占端口时新实例会启动失败，但请求全落在旧进程上。
- 单机窗口失焦会静默停 tick，走服务端线程队列的调试端点全部超时——这是暂停导致的，不是代码问题。玩家正在操作客户端时不要抢着传送或截图。
- 客户端运行时构建会覆盖 `build/classes`，导致已加载的类找不到。重启客户端前先确认进程已完全退出。
- 前端资源有缓存时，给静态资源 URL 附版本号即可刷新，不必重启游戏。

## 调试端点体系

可选的一种做法：在调试开关（如 `-PdebugApi`）下挂一组 HTTP 端点，把「搭示例结构 / 清除 / 传送 / 转视角 / 开关 GUI / 截图 / 查询方块实体」做成只读或低风险操作，便于自动化自查。

设计约束：
- 涉及游戏对象的操作必须投递到主线程队列，在 tick 里执行，HTTP 线程不要直接碰世界。
- 「关界面」这类纯客户端操作不要塞进服务端线程，否则游戏暂停时调不动。
- 这类端点只在开发模式启用，不随正式构建暴露。
