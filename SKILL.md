---
name: apk-reversing
description: Android APK 逆向分析工作流。基于本机已配好的 jadx MCP(jadx-gui 1.5.6 + jadx-ai-mcp 插件),覆盖环境自检、侦察、定位、调用链追踪、改名固化、故障排查。当用户要求分析/反编译 APK、查某功能或字符串的实现、追调用链、看 manifest 组件与权限、判断加固/混淆、改 smali、评估 APK 行为时使用。触发词:APK, apk, jadx, 反编译, 逆向, smali, dex, xref, 交叉引用, 脱壳。
metadata:
  author: ACER
  version: 1.0.0
  category: reverse-engineering
  requires: [jadx-mcp, bash]
---

# APK 逆向分析工作流

适用面:自有 APK、授权测试目标、CTF、安全评估范围内的样本。分析步骤本身无差别,只是别把结果用在没授权的目标上。

## 0. 环境事实(本机特化,别按官方文档试)

本机 jadx MCP 已配好,走 GUI 插件桥接。三条反直觉的点,踩过一遍:

| 坑 | 事实 |
|---|---|
| `uvx jadx-mcp-server` | **包从未发布到 PyPI**,必 404。只能用源码跑:`uv --directory C:\Users\ACER\jadx-ai\jadx-mcp-server run jadx_mcp_server.py`(依赖在 .venv,CPython 3.12) |
| 插件 jar 放哪 | 不是 `%APPDATA%\skylot\jadx\plugins\`,而是 **`%APPDATA%\skylot\jadx\config\plugins\dropins\`**(真实 configDir 多一层 `config`)。投 dropins 即被 `JadxExternalPluginsLoader` 全量扫描,不用走 GUI 安装框 |
| jadx CLI | **本机没有**。桌面 `jadx-gui-1.5.6-with-jre-win` 无 bin/jadx.bat,所以 `jadx plugins --install` 那套跑不了 |

插件在 GUI 内监听 **8650**,MCP server 走 **stdio** 连它。MCP 配置写在 `~/.claude.json` 顶层 `mcpServers.jadx`(command 用 uv.exe 绝对路径,因为 `~/.local/bin` 不在 Windows PATH)。

## 1. 开工自检(preflight)

工具报错九成是这三步没就位,先查再干活:

```bash
# 1) 插件在监听吗
netstat -ano | grep 8650          # 有 LISTENING 才通
# 2) MCP 配置在吗
grep -A6 '"jadx"' ~/.claude.json
# 3) 插件 jar 在 dropins 吗
ls "$APPDATA/skylot/jadx/config/plugins/dropins/"
```

GUI 侧人工确认两点:**已经打开目标 APK**(没加载 APK 时所有工具返回空/报错)、Plugins 菜单里插件是启用态。这两条我查不到,必须让用户确认或先请其打开。

## 2. 侦察:先拿全局,再钻细节

```
get_android_manifest            → 包名、权限、SDK 版本
get_manifest_component          → 组件清单;only_exported=true 直接筛攻击面
get_package_tree                → 包结构 + class_count,识别自带库 vs 业务代码
get_main_activity_class         → 入口 Activity
get_main_application_classes_names → Application 子类(常在构造函数里埋初始化/加固逻辑)
```

判断加固:包树里业务类极少但 dex 很大、`Application` 指向 `com.stub.StubApp`/`com.qihoo`/`s.h.e.l.l` 之类、字符串表几乎为空 → 先脱壳再谈分析,别硬啃。

## 3. 定位:从关键词切进去

```
search_classes_by_keyword  search_term=<关键词>  search_in=class,method,field,code
search_method_by_name      <方法名>              # 全库按名找
get_strings                资源 strings.xml      # URL / 密钥 / 埋点 key 常在这
```

关键词优先级:**用户可见文案 → 报错信息 → URL/路径 → 常量名**。中文字符串命中率最高,因为开发者很少给它们改名。混淆过的包直接跳到第 4 步用 manifest 组件当锚点。

## 4. 追踪:调用链靠 xref,不靠通读

```
get_xrefs_to_method / _field / _class   ← 核心。谁调用我,比我调用谁更有信息量
get_method_by_name        只取目标方法体,别 get_class_source 整类
get_fields_of_class       看状态字段反推数据结构
get_smali_of_class        伪代码失真/内联可疑时,拿 smali 对指令
```

自顶向下(从入口追到实现)和自底向上(从字符串/xref 反追调用方)交替用。遇到反射、动态代理、字符串拼接的方法名,伪代码会断链 —— 回 smali 找 `invoke-*` 与 `const-string` 的实际取值。

## 5. 固化:改名,让结论可交付

分析过程中持续用 `rename_class` / `rename_method` / `rename_field` / `rename_variable` / `rename_package`。改完 GUI 里符号名可读,后续 xref 结果自带语义,省掉反复回看。**改名只影响当前 jadx 工程视图,不动原 APK**,放心改。

交付产物按需:调用链结论 + 关键方法路径(`类#方法:行号`)+ 证据(smali 片段或字符串引用),必要时导出改名后的工程给用户。

## 6. Token 纪律

- 侦察靠 `get_package_tree` + manifest,别 `get_all_classes` 全量拉(几千行直接吃掉预算)。
- 读代码用 `get_method_by_name` 精确取,**不要** `get_class_source` 拉整个类;大类必炸。
- 用 `get_xrefs_*` 代替"把相关类都读一遍"。
- 搜索结果默认 page 20 条,按需翻页,别一次性 count 拉满。
- 换样本前 `clear_cache`;怀疑命中率低时 `get_cache_stats` 看缓存状态。

## 7. 故障速查

| 现象 | 原因 / 处置 |
|---|---|
| 所有工具返回空 | GUI 没打开 APK,或插件没启用(Plugins 菜单) |
| 连接被拒 / 超时 | 插件没起,或 8650 被占:`netstat -ano \| grep 8650` 找 PID |
| MCP 工具在列表里不存在 | `~/.claude.json` 的 `mcpServers.jadx` 丢了/uv.exe 路径错了,重启 Claude Code |
| 方法体是 `native` 或空 | 加固/动态加载;找 `System.loadLibrary` 与 dex 释放点,或转 frida 侧 |
| 改名后混乱 | 改太早。先把调用链摸清再批量改名 |

## 8. 收尾

- 结论要能落到**文件:符号**粒度,别只给"大概在某处"。
- 涉及安全结论(如"这里能越权")时,给复现要素与触发条件,别只下判断。
- 本机环境若变动(插件降级、jadx 换版),同步更新本文件第 0 节的表。

## 9. 遇到 MT Protect / 同类加固(实测路径)

- **字符串加密 ≠ 无从下手**。先抽 dex 原始字符串(`re.findall(rb'[ -~]{4,}', dex)`),未加密的英文日志串和 UI 字面量通常还在;用 `search_in=code` + `package=<业务包>` 限定,再用 xref 反向收敛。实例:`"SMB (VIP)"` → 菜单类 → `if (C2839.m4692())` 会员 boolean,一步到咽喉点。
- **权限被搬进 native 时,找「整类都是 `static native`」的类**。它通常是唯一咽喉点(实例:16 个方法全 native,含 7 个 boolean 权限 + 到期 long)。在 JNI 桥替换 Java getter 即可,不要去啃 native 机器码。
- **jadx 注释 `renamed from: l.xxx` 就是运行时真名**,可直接 `Java.use()`;`pNNN` 包前缀是 jadx 造的假名。要逐字符核对:在 dex 里搜描述符 `Ll/<名字>;` 并按 `U+XXXX` 码点打印,别用肉眼比阿拉伯/叙利亚字符。
- **frida 附加窗口**。自保护型加固(native 自己 ptrace 自己)在启动约 5 秒后会让 `frida -p` 以 "unexpectedly timed out while waiting for stop" 失败。规则:**启动后 2 秒内 attach**;`frida -f` spawn 模式会被卡死,用 `am start` + 立即 attach。
- **改包死、hook 活**。签名/完整性校验在 native `start` 里跑,二次打包会 ANR。凡「解锁 / 去校验」类需求优先运行时 hook 或 LSPosed 模块,别改 APK。
- **hook 类要能跳版本**:别写死混淆名,挂 `ClassLoader.loadClass` + `Class.forName`,用结构指纹认类(声明方法数 + 全部 static native + 某个无参返回 `java.util.Map`);已知版本再加一条快速通道。

## 10. 运行时 hook 加固应用的三个坑(实测踩过)

- **`getString(-1)` 会直接杀进程**。加固应用常用 -1 作「资源查找失败」哨兵(id 名→资源 id 的查找表看不到键就返回 -1),那条分支一旦被走到就 `Resources$NotFoundException` → main 线程 FATAL → 用户看到「一进就大退」。处置:`hookAllMethods(Resources.class, "getText"/"getString")`,在 `afterHookedMethod` 里只在 `param.getThrowable()!=null && args[0] 是 int 且 <0` 时 `setThrowable(null) + setResult("")`。**只堵负值 id,正常取值不碰**,既修崩溃又不改语义。
- **盲目「全 boolean 置 true」会造出无效组合**。权限位往往不是独立的:某几个位同时为真才会走进一条依赖缺失资源的UI分支。所以强制之前先用二分定位(`off=` 排除单个位再启动验证 crash),并保留一个「逐位开关」配置项给用户。
- **调用者追踪会被框架污染**。XposedBridge/框架回调自身也混淆(帧名像 `g.a`、`e2.proceed`、`c3.callback`),按包名排除永远只能拿到框架帧。反过来做:**只保留看起来属于目标应用的帧**(单字母包、含非 ASCII 类名、或应用自己的包前缀),才能拿到真实功能调用点。
- **崩溃定位靠 logcat,不靠应用自己弹的框**。应用内的错误弹框会把数字改写(实测 `0xffffffff` 被显示成 `0x0`),`adb logcat -d | grep -A40 "FATAL EXCEPTION"` + `grep " <pid> "` 拿到的才是真实值。
- **用户说“登录失效/数据乱了”时先查应用自己的日志**:`/data/data/<pkg>/files/log/` 里的 `process-exit-history` 会写明 `clear data` / `exit_self` / `SIGNALED`,能分辨是应用自保护还是外部动作(清数据/重装)造成的。

## 11. Hermes 里驱动 jadx MCP(没连接器时的走法)

Hermes 侧没有 jadx MCP 连接器,但 GUI 插件的 HTTP 桥在 `127.0.0.1:8650` 明文可打,直接用 urllib/curl 就行。现成客户端:`~/work/jd.py`(`python jd.py search 卡密 "" code 0 20`)。

| 用途 | 端点 | 参数 |
|---|---|---|
| 健康检查 | GET /health | — |
| 包树 | GET /package-tree | — |
| manifest | GET /manifest | — |
| 类源码 | GET /class-source | class_name |
| 方法体 | GET /method-by-name | class_name, method_name[, method_signature] |
| 全类名 | GET /all-classes | offset, count |
| 关键字搜索 | GET /search-classes-by-keyword | search_term, package, search_in, offset, count |
| 方法名搜索 | GET /search-method | method_name |
| xref | GET /xrefs-to-class、/xrefs-to-method、/xrefs-to-field | class_name[, method_name/field_name], offset, count |
| 资源清单 | GET /list-all-resource-files-names | offset, count |
| 读资源 | GET /get-resource-file | file_name |
| 重命名 | GET /rename-class、/rename-method、/rename-field、/rename-package、/rename-variable | … |
| 清缓存 | POST /cache-clear | — |

端点名官方文档里没有,直接 grep `C:\Users\ACER\jadx-ai\jadx-mcp-server\src\server\tools\*.py` 的 `get_from_jadx("` / `endpoint="`。代码级搜索要给足超时(600s+),大 APK 要几分钟。

## 12. 假名 vs 真名:写 hook 前必须回 dex 核对

两个方向都会骗人:

- **`/all-classes` 返回的是 jadx 去混淆后的别名**(`com.lark.injector.C1959U`、`AbstractC1942C`),**不是运行时真名**;导出的 Java 源码用的是**短真名**(`com.lark.injector.U`)。
- 判据只有两条:①源码里 `/* JADX INFO: renamed from: com.lark.injector.y */` 注释就是运行时真名;②**grep dex 描述符**:`re.finditer(rb'L([A-Za-z0-9_/$]{1,40});', dex)`。
- `pNNN` 包前缀、`CXXXX` 类前缀都是 jadx 造的假名。实例:`p032l0.p` 真名 `l0.p`,`p066z0.AbstractC0224a` 真名 `z0.a`;`B0.g` 本身就是真名。

## 13. 插件化应用(Shadow / lark 授权):插件类不在 lpparam.classLoader 里

宿主 base.apk + 插件 plugin.apk(Shadow)结构下,插件类由 Shadow 自己的 ClassLoader 加载:`XposedHelpers.findClass("B0.g", lpparam.classLoader)` 必然 ClassNotFound;frida 的 `Java.use()` 默认也用 app ClassLoader,一样失败。

抓插件 ClassLoader(按可靠性排):

1. 挂 `android.content.ContextWrapper.attachBaseContext(Context)`(框架方法一定挂得上),`afterHookedMethod` 里判 `param.thisObject.getClass().getName().startsWith("<插件 Application 包前缀>")`,是就取 `getClassLoader()`。时机在插件 Application attach,早于一切业务代码。
2. 挂插件 Application 基类的 `attachBaseContext`(如 `com.lark.shadow.core.runtime.ShadowApplication`)。
3. 兜底挂 `java.lang.ClassLoader.loadClass`,类名前缀过滤,命中即抓,装完 `Unhook.unhook()` 卸掉,别留热路径开销。

frida 侧同理:`Java.enumerateClassLoaders()` + `Java.ClassFactory.get(loader)` 才 `use()` 得到插件类。

## 14. Vector(兼容 LSPosed 的框架)开模块:用 CLI,别手起 daemon

- 框架在 `/data/adb/modules/zygisk_vector/`,`service.sh` 开机起 daemon(带 `unshare` 私有命名空间);CLI 是 `/data/adb/lspd/cli`(连 `/data/adb/lspd/.cli_sock`)。
- 命令:`cli status`;`cli modules ls`(纯文本会输出 No records found,**要 `--json`**);`cli modules enable|disable <pkg...>`;`cli scope ls|add|set|rm <module> <pkg/user_id>`。
- 配置真源 `/data/adb/lspd/config/modules_config.db`(SQLite:`modules` 表带 enabled 列,`scope` 表 module_id→包名)。CLI 连不上时可 pull 下来用 python sqlite3 只读看,确认模块有没有启用、作用域有没有目标包。
- **绝不要手动起 daemon**:`su -c '<moddir>/daemon ...'` 绕过开机时序,和已在跑的 system_server 注入打架,**直接把 system_server 顶重启(软重启)**,现象是 adb 掉设备。CLI 报 `Connection refused` 就等它自己起,不要自己拉。
- 模块启用 + 作用域就绪后,目标应用要**强杀重启**才吃新配置。

## 15. 本机 frida 姿势

- frida-server 在 `/data/local/tmp/frida-server`,**必须以 root 跑**(`su -c /data/local/tmp/frida-server`),否则 `frida -U -f <pkg>` 报 `need Gadget to attach on jailed Android`。
- frida 17 的 CLI **没有 `--no-pause`**(unrecognized arguments);`-f` spawn 后自动 resume。
- CLI 没有 child gating;要连子进程(如插件进程 `:lark_pluginX`)一起 hook:用 Python API 的 `device.enable_child_gating()`,或子进程起来后按 pid `-p` attach,或直接写 Xposed 模块(挂全部进程,最省事)。

## 16. 云授权(lark「卡密」)类应用的完整打法(实测样本 com.nxnb.cn)

- 结构:宿主(壳/运行时)+ `assets/plugin.apk`(授权 UI,Shadow 插件)+ `assets/larkap.apk`(广告模块)。
- 总闸在宿主:`com.lark.injector.LarkInject.launchPlugin()` 里判 `tempKV("task_success")`;在它之前调 `com.lark.injector.hostlib.HostLib.taskSuccess()` → 授权插件整个不启动,卡密流程消失(一行 hook 解决)。
- 插件侧总闸:`B0.g.e(PluginApp, boolean)` 是所有注册/激活/Html 任务的共同判据(true = 该任务已满足、不弹);`l0.e.a(config)` 返回空 ArrayList = 一关都不用过,立刻 taskSuccess。注意 **Html 任务本身就是付费弹窗**,别把它一起判过。
- 插件类名真名:`z0.a`(任务基类)、`z0.k/z0.s/z0.l/z0.o/z0.c/z0.A`(注册/试用/公告/密码/更新/绑定/分享)、`l0.e`(调度)、`l0.p`(RegisterCore,`d(code)` 是卡密校验)、`l0.l`(Success)、`l0.a`(本地状态,`a()` = getEndTrial)、`r0.g`(@JavascriptInterface "lark" 桥,`getTrialStatus`/`verify`/`trial`)。
- 服务器接口**无鉴权**:`GET https://<host>/feature/config?app_id=..&device_id=..&uid=..&channel_id=1000&app_version_code=..` 就能拿到完整配置(含下发的激活页 HTML+JS),`/feature/register/verify?code=` 是卡密校验。host 从 `HostLib.getBestHost()`/`getCfHost()` 来(样本里是 `a1.gxogyub.org`、`cf.gxogyub.org`,还有 IP 直连)。
- 宿主配置是 RSA/PKCS1 加密的 base64,私钥硬编码在 `hostlib/ConfigDecoder` 里,可直接解出 JSON 看 `xpCheck`/`vpnCheck`/`cleanData` 这些开关(样本里全是关的)。
- 另有到期看门狗 `lark.WATCHDOG.<appId>`(`com.lark.injector.U` 收)和插件侧的窗口面积 <70% 自杀检查(`MainActivity.k(Configuration)`),都要单独屏蔽。
