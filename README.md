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
