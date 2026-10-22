# 微信防发错提醒（WeChat Chat Guard）

切微信窗口或切换聊天对象时，屏幕顶部弹出大字提示当前聊天对象，防止发错人。

## ✨ 功能特性

- **焦点检测**：微信窗口获得焦点时自动检测当前聊天对象
- **大字提示**：屏幕顶部居中弹出半透明提示框，显示当前聊天对象
- **声音提醒**：可在托盘菜单开关提示音
- **双路识别**：
  - 优先使用 UIAutomation 读取微信无障碍树（准确、零延迟）
  - 无障碍树不可用时自动降级为截图 + OCR 识别聊天标题
- **优雅降级**：两种方式都读不到时，弹通用提示提醒用户自行确认
- **纯脚本**：PowerShell 编写，无需编译安装，双击即用
- **单实例**：重复启动自动退出，不会弹出多个提示
- **托盘菜单**：测试提示 / 开关声音 / 查看日志 / 退出

## 📦 使用方法

1. 下载 `wechat-guard.ps1` 和 `wechat-guard.vbs` 到同一文件夹
2. 双击 `wechat-guard.vbs` 静默启动（右下角出现托盘图标）
3. 切换到微信窗口，或在微信内切换聊天对象，即可看到顶部提示

> 也可以命令行运行：`powershell -ExecutionPolicy Bypass -File wechat-guard.ps1`
>
> 测试模式：`powershell -ExecutionPolicy Bypass -File wechat-guard.ps1 -Test`

## 🔧 配置说明

编辑 `wechat-guard.ps1` 顶部的 `$script:Cfg`：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| PollMs | 250 | 检测间隔（毫秒） |
| ShowMs | 1000 | 提示停留时间（毫秒） |
| Width | 560 | 提示框宽度 |
| Top | 72 | 提示框距屏幕顶部距离 |
| Beep | $true | 是否开启提示音 |

## 🧠 工作原理

### UIA 路径（首选）
通过 `System.Windows.Automation` 获取微信主窗口的无障碍树，
查找 `mmui::ChatSessionCell` 类名的列表项，
取 `SelectionItemPattern.IsSelected` 为 true 的项的 `Name` 属性。

### OCR 路径（兜底）
某些微信实例（如双开的第二个号）不提供无障碍树，此时：
1. `PrintWindow(hwnd, hdc, PW_RENDERFULLCONTENT)` 抓取窗口内容
2. 在左侧会话列表区域定位绿色高亮选中行，确定聊天面板左边界
3. 截取右上角聊天标题区域
4. 裁剪 + 放大 2 倍 + 二值化（阈值 150）+ 补边 20px
5. 调用 Windows 自带 OCR（`zh-Hans-CN`）识别
6. 清洗结果：去空格、去时间、去末尾群人数、去开头被误识别的圆形图标噪声

## 📋 环境要求

- Windows 10 / 11
- PowerShell 5.1+（系统自带）
- .NET Framework 4.5+（系统自带）
- （可选）中文文本识别语言包——用于 OCR 兜底，未安装时 UIA 仍可正常工作

## 📝 日志

运行日志保存在脚本同目录的 `wechat-guard.log`，可通过托盘菜单打开。


（以下为ai提示词 可自行生成一份属于自己的）
帮我写一个 Windows 上的"微信防发错提醒"小工具，纯 PowerShell 脚本，不要编译、不要装依赖，双击就能用。

【功能需求】
1. 每 250ms 检测一次前台窗口。如果当前焦点窗口是微信主窗口，就读出"当前选中的聊天对象是谁"，然后在屏幕顶部居中弹出一个大字提示框，显示这个聊天对象的名字，1 秒后淡出。
2. 在微信内部切换会话时也要弹，不能只在切窗口时弹。
3. 如果当前停在微信的"通讯录""发现"等其他页签，不要弹，避免误导。
4. 右下角托盘图标，右键菜单要有：
   - 测试一下提示
   - 开启声音（可勾选，点击切换提示音开关，默认开）
   - 打开日志
   - 退出
5. 单实例运行，重复启动直接退出，避免弹两次。
6. 日志写到脚本同目录的 wechat-guard.log。

【技术实现要求，这些是重点，别自己换方案】
1. 主路径用 UIAutomation（Add-Type -AssemblyName UIAutomationClient / UIAutomationTypes）：
   - 微信主窗口 UIA 根节点的 ClassName 是 mmui::MainWindow
   - 会话列表项 ClassName 是 mmui::ChatSessionCell，ControlType 是 ListItem
   - 用 SelectionItemPattern.IsSelected 判断哪一项被选中，取它的 Current.Name 第一行
   - 左侧导航栏项 ClassName 是 mmui::XTabBarItem，Name 为"微信"的那项用来判断是否在聊天页
2. 必须有 OCR 兜底！有些微信实例（尤其是双开的第二个号）根本不提供无障碍树，UIA 读到 0 个节点。这种情况要自动降级为截图识别：
   - 用 PrintWindow(hwnd, hdc, 2)（PW_RENDERFULLCONTENT）抓窗口，窗口被遮挡也能抓到
   - 在左侧会话列表区域（x 从 3% 到 28% 宽度）找绿色高亮选中行（微信选中色 #07C160，判据 g>140 && r<125 && b<175 && g-r>60 && g-b>25），用它的右边界定位聊天面板左边界
   - 截取右上角聊天标题区域，裁剪 + 放大 2 倍 + 二值化（灰度阈值 150）+ 周围补 20px 黑边，再送 Windows 自带 OCR
   - OCR 引擎用 [Windows.Media.Ocr.OcrEngine]::TryCreateFromLanguage(zh-Hans-CN)，拿不到再回退 TryCreateFromUserProfileLanguages()
   - 调用 WinRT 异步方法要自己写 Await 封装（用 WindowsRuntimeSystemExtensions 的 AsTask 反射）
   - 标题区域做哈希，没变化就跳过 OCR，每秒最多识别一次
3. OCR 结果必须清洗，否则会显示乱码：
   - 去掉所有空格、去掉第一行里的时间戳
   - 去掉标题末尾的括号补充信息（群人数等）
   - 关键：OCR 会把标题栏的圆形图标（搜索放大镜、头像、状态圆点）误识别成 o / O / 0，如果名字里含中文，就把开头所有非中文字符剥掉（纯英文名不要动）
4. 提示框用 WPF 无边框窗口：AllowsTransparency + Topmost + ShowInTaskbar=false + ShowActivated=false，并且用 SetWindowLong 加 WS_EX_NOACTIVATE | WS_EX_TRANSPARENT | WS_EX_TOOLWINDOW，保证它不抢焦点、点不中，否则会干扰你在微信里打字。
5. 启动时调 SetProcessDPIAware()，提示框用物理像素定位（GetSystemMetrics + SetWindowPos），否则高 DPI 缩放下位置会偏。
6. 脚本文件必须保存为 UTF-8 with BOM，否则在别人电脑的 PowerShell 5.1 上中文会变乱码。
7. 再配一个 wechat-guard.vbs 静默启动器，内容就是 WScript.Shell.Run 调用同目录的 ps1，窗口模式 0、不等待。

【验证要求】
写完以后帮我列出：
- 我怎么判断它到底走的是 UIA 还是 OCR（日志要能看出来）
- 如果对方电脑上完全不弹提示，我该看日志里哪几行
- 我该怎么把日志内容发给你定位问题
