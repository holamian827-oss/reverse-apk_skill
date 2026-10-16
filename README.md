# Android Reverse Engineering Skill

面向 Android 应用分析与安全研究的实践手册与工具集合，收录常用工具、环境配置与分析流程，帮助研究者在获得授权的前提下，高效进行 APK/DEX 静态与动态分析。

✨ 项目简介

本仓库整理了 Android 逆向与动态分析过程中常用的方法与工具、环境搭建建议，以及常见问题的排查思路，目标是降低入门门槛并提升分析效率。仅供学习、研究和经授权的安全测试使用。

适用场景

- APK 文件结构与组件分析
- DEX 字节码与反编译代码审查
- Java / Kotlin 逻辑定位与研究
- 应用运行时行为观察与分析
- 动态调试、Frida 插桩与 Hook 测试
- Xposed / LSPosed 模块开发与调试
- 安全测试与漏洞复现
- 逆向分析常见问题排查

🛠️ 推荐工具

（按需选择，无需全部安装）

- JADX — DEX/APK 反编译与代码浏览
- Frida — 动态插桩与运行时函数 Hook
- Android SDK / ADB — 设备管理、安装与日志查看
- Android Studio — 构建、调试与 APK 分析
- Xposed / LSPosed — 运行时模块开发与实验

环境建议

- 一台可通过 ADB 连接的 Android 设备或模拟器。
- 根据目标应用选择匹配的 Android SDK 与工具版本。
- 确认工具与设备 CPU 架构（arm/arm64/x86）兼容。
- 使用 Frida 或 Xposed/LSPosed 时，请参照各项目官方安装说明。

🚀 快速开始（更好的 Git 指南）

1. 克隆仓库（HTTPS）：

   git clone https://github.com/holamian827-oss/reverse-apk_skill.git

   或使用 SSH（已配置公钥）：

   git clone git@github.com:holamian827-oss/reverse-apk_skill.git

2. 准备环境：

   - 安装所需工具（例如：JADX、Frida、Android SDK / platform-tools）。
   - 验证 ADB 能识别设备：

     adb devices

   - 验证工具可执行：

     jadx --version
     frida --version

3. 按照仓库中的文档和 SKILL.md 中的建议进行静态/动态分析。

如果你希望把本仓库作为子模块或依赖引入：

- 作为子模块：

  git submodule add https://github.com/holamian827-oss/reverse-apk_skill.git tools/reverse-apk-skill

- 仅下载 README：

  curl -L https://raw.githubusercontent.com/holamian827-oss/reverse-apk_skill/main/README.md -o README_remote.md

分析流程（推荐）

1. 静态分析：检查 APK、AndroidManifest.xml、资源与 DEX 文件。
2. 代码分析：用 JADX 等工具阅读反编译代码，定位关键类与调用链。
3. 动态分析：在受控且有授权的测试环境下监控运行时行为并做必要的 Hook。
4. 验证：结合静态与动态证据验证假设。
5. 记录：写明步骤、工具版本、复现条件与结论。

📖 使用建议与注意事项

- 优先做静态分析以缩小动态分析范围。
- 面对混淆或动态加载时，区分“确认结论”和“尚需验证的推测”。
- 不要在公共 issue/PR 中上传真实用户数据或凭证。
- 遵守法律与授权范围，未经授权不得进行侵入、数据窃取或规避访问控制的行为。

📂 项目结构

reverse-apk-skill/
├── README.md
└── SKILL.md

（实际结构请以仓库为准）

⚠️ 法律与伦理声明

本项目仅供学习、研究、兼容性评估和获得明确授权的安全测试使用。使用者需自行确保其行为符合适用法律法规与目标软件许可条款，对因使用本项目而产生的任何后果自行负责。

🤝 参与贡献

欢迎通过 Issues 或 Pull Requests 提交改进建议。请在 issue/PR 中说明环境、重现步骤与相关工具版本，且不要提交敏感信息。

📄 许可协议

本项目采用 GPL-3.0 许可。

如果本项目对你有帮助，欢迎 Star、Fork 并参与改进。