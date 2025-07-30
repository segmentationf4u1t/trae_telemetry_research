好的，这是您提供的技术分析报告的中文翻译：

# Trae IDE 遥测与隐私分析：深入探究字节跳动的 VSCode 分支

## 执行摘要

本分析报告旨在审视在字节跳动（ByteDance）的 Visual Studio Code 分支版本 Trae IDE 中发现的、令人担忧的隐私及社区管理问题。主要发现包括：即使用户在设置中禁用了遥测，遥测数据仍持续传输；广泛的数据收集行为；以及对技术性批评进行社区审查的令人担忧的做法。

## 1. 背景与方法论

在为个人项目评估开发环境时，我进行了一项对比分析，结果揭示了 Trae IDE 中存在重大的隐私问题，这些问题值得进行更深入的调查。

**测试环境：**
- 操作系统：Microsoft Windows 11 专业版
- CPU：Intel Core™ i7-14700KF
- 内存：64GB
- 测试项目：在每个 IDE 中加载完全相同的代码库
- 监控工具：System Informer, Fiddler Everywhere

## 2. 网络流量与遥测调查

网络监控显示，Trae IDE 持续向字节跳动的基础设施建立出站连接：



*图1：Trae 的网络活动，显示其定期连接到字节跳动服务器*

**已识别的主要端点：**
- `http://mon-va.byteoversea.com`
- `http://maliva-mcs.byteoversea.com`
- `https://mon-va.byteoversea.com/monitor_browser/collect/batch/?biz_id=marscode_nativeide_us`

### 遥测配置测试

#### 禁用遥测

我尝试通过标准设置界面禁用遥测：



*图2：在用户设置中禁用遥测*

#### 意外发现

禁用遥测并未减少网络活动。相反，它：
- 维持了与 `mon-va.byteoversea.com` 和 `maliva-mcs.byteoversea.com` 的现有连接
- **增加**了向批量收集端点的请求频率

在禁用遥测的情况下，这些调用仍然会迅速累积——在测试期间，单个批次的数据量可高达 53,606 字节。在活跃使用编辑器的大约 7 分钟内，我观察到大约 500 次调用，在这么短的时间内总共传输了高达 26 MB 的数据。



*图3：调用次数增加*

## 3. 数据传输分析

### 批量遥测负载

即便禁用了遥测，Trae 仍继续传输详细的使用数据：

```json
{
	"ev_type": "batch",
	"list": [
		{
			"ev_type": "custom",
			"payload": {
				"name": "icube_ai_ckg_request",
				"type": "event",
				"metrics": {
					"cost_time": 5
				},
				"categories": {
					"ckg_method": "refreshToken",
					"status": "Failed",
					"err_msg": "None",
					"ckg_err_code": "SUCCEED"
				}
			},
			"common": {
				"bid": "marscode_nativeide_us",
				"user_id": "已隐去 :)",
				"device_id": "已隐去 :)",
				"session_id": "已隐去 :)",
				"env": "prod",
				"timestamp": 1753636703370,
				"sdk_version": "1.12.7",
				"sdk_name": "SDK_SLARDAR_WEB",
				"pid": "AI Agent",
				"view_id": "AI Agent 1753636703370",
				"context": {
					"tracing_id": "已隐去 :)",
					"span_id": "59a833c5359d4978",
					"product": "nativeIDE",
					"machine_id": "已隐去 :)",
					"user_id": "已隐去 :)",
					"user_name": "已隐去 :)",
					"build_version": "1.0.16066",
					"app_version": "2.0.2",
					"build_time": "2025-07-21T05:08:14.915Z",
					"quality": "stable",
					"eventVersion": "1.95",
					"arch": "x64",
					"system": "win32",
					"remote_arch": "x86_64",
					"remote_system": "windows",
					"scope": "marscode",
					"biz_user_id": "已隐去 :)",
					"user_is_login": "true",
					"device_id": "已隐去 :)",
					"user_unique_id": "已隐去 :)",
					"organization": "",
					"vscode_version": "1.100.3",
					"region": "US",
					"aiRegion": "US",
					"os_name": "windows",
					"os_version": "Microsoft Windows 11 Pro",
					"cpu": "Intel",
					"is_ssh": "false",
					"app_language": "en",
					"chat_mode": "0",
					"identity": "1",
					"identity_str": "Pro",
					"pro_period": "0",
					"has_package": "0",
					"is_freshman": "0",
					"channel_name": "common",
					"extra": "{\"cpu_brand\":\"Core™ i7-14700KF\",\"cpu_family\":\"6\",\"cpu_speed\":3.4,\"device_manufacturer\":\"ASUS\",\"device_model\":\"System Product Name\",\"memory\":68523634688,\"language\":\"en-us\"}"
				}
			}
		}
	]
}
```
虽然这可能也与身份验证有关，但它包含了大量的硬件规格信息，这是为什么？

### 用户活动追踪

另一个端点 (`maliva-mcs.byteoversea.com`) 接收了更细粒度的用户交互数据：

```json
[
	{
		"events": [
			{
				"event": "_be_active",
				"params": "{\"start_time\":1753636688373,\"end_time\":1753636688373,\"url\":\"vscode-file://vscode-app/c:/Users/segfault/AppData/Local/Programs/Trae/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html\",\"referrer\":\"\",\"title\":\"read.md - tauri-app - Trae\",\"event_index\":1753636640233}",
				"local_time_ms": 1753636688374,
				"is_bav": 0,
				"session_id": "已隐去 :)"
			},
			{
				"event": "icube_time_on_ide",
				"params": "{\"entryTime\":1753636462123,\"now\":1753636687360,\"workspaceId\":\"88eda3be5f237760cfadaa7abbd89c24\",\"duration\":1964,\"lastActiveTime\":1753636685396,\"isEditorFocused\":false,\"activeEditorTypeId\":\"workbench.editors.files.fileEditorInput\",\"isMouseOrKeyboardActive\":true,\"isAiActive\":false,\"activeEditorProviderId\":\"\",\"activeEditorLanguage\":\"markdown\",\"activeEditorFileExt\":\".md\",\"isFocus\":\"not_focus\",\"isVisible\":\"hidden\",\"event_index\":1753636640232}",
				"local_time_ms": 1753636688372,
				"is_bav": 0,
				"session_id": "已隐去 :)"
			}
		],
		"user": {
			"user_unique_id": "已隐去 :)",
			"user_id": "已隐去 :)",
			"user_is_login": true,
			"device_id": "已隐去 :)"
		},
		"header": {
			"app_id": 677332,
			"app_version": "2.0.2",
			"os_name": "windows",
			"os_version": "Microsoft Windows 11 Pro",
			"device_model": "System Product Name",
			"language": "en-us",
			"region": "US",
			"app_language": "en",
			"platform": "electron",
			"sdk_version": "5.1.25",
			"sdk_lib": "js",
			"timezone": 2,
			"tz_offset": -7200,
			"resolution": "2560x1440",
			"browser": "Chrome",
			"browser_version": "132.0.6834.210",
			"referrer": "",
			"referrer_host": "",
			"width": 2560,
			"height": 1440,
			"screen_width": 2560,
			"screen_height": 1440,
			"tracer_data": "{\"$utm_from_url\":1}",
			"custom": "{\"icube_uid\":\"7472574750745953285\",\"biz_user_id\":\"7472574750745953285\",\"is_special_uuid\":false,\"machine_id\":\"9531bb512f2c1aff9c2ed17dd9e342026f0e1e78326b35ee01e03f6439b869ca\",\"arch\":\"x64\",\"system\":\"win32\",\"scope\":\"marscode\",\"organization\":\"\",\"build_version\":\"1.0.16066\",\"vscode_version\":\"1.100.3\",\"tenant\":\"marscode\",\"aiRegion\":\"US\",\"quality\":\"stable\",\"build_time\":\"2025-07-21T05:08:14.915Z\",\"icube_main_uid\":\"f8207401-9c52-4583-82d3-e1e44e2da0f0\",\"window_id\":2,\"workspace_id\":\"88eda3be5f237760cfadaa7abbd89c24\",\"os_release\":\"10.0.26100\",\"os_build\":\"26100\",\"device_manufacturer\":\"ASUS\",\"cpu\":\"Intel\",\"cpu_brand\":\"Core™ i7-14700KF\",\"cpu_vendor\":\"GenuineIntel\",\"cpu_family\":\"6\",\"cpu_model\":\"183\",\"cpu_stepping\":\"1\",\"cpu_speed\":3.4,\"memory\":68523634688,\"is_ssh\":false,\"chat_mode\":0,\"identity\":\"1\",\"identity_str\":\"Pro\",\"pro_period\":\"0\",\"has_package\":\"0\",\"is_freshman\":\"0\",\"channel_name\":\"common\"}"
		},
		"local_time": 1753636689,
		"verbose": 1
	}
]
```

### 数据收集范围

该遥测系统捕获的数据包括：
- **系统信息**：硬件规格、操作系统详情、架构
- **使用模式**：活跃时间、会话时长、功能使用情况
- **性能指标**：响应时间、资源消耗
- **唯一标识符**：机器 ID、用户 ID、设备指纹
- **工作区详情**：项目信息、文件路径（经过混淆）

👉 [在 Streamable 上观看网络请求视频](https://streamable.com/e/agr0a2?loop=0)

## 4. 社区管理问题

### 自动审查

当我尝试在 Trae 的 Discord 服务器上讨论这些发现时，我遭遇了令人担忧的审查行为：




https://discord.com/channels/1320998163615846420/1335032920825391/1398374824987852891

观察到的问题包括：

1.  **自动惩罚**：提及“追踪”（tracking）问题会立即触发为期 7 天的禁言。
2.  **压制技术讨论**：合法的安全顾虑被当作破坏性行为处理。

## 25年7月28日更新

在被禁言三天后，社区管理员 "Rdap" 确认了禁言是自动触发的，并且看起来已经解除了。然而，触发自动警告的消息对于拥有相应权限的用户来说，在 Discord 聊天中仍然可见。我不确定为什么警告没有被立即解除，尤其是当时工作人员已经清楚地看到了事情的经过。

禁言的截图显示：

"Nie można opublikować tego elementu, ponieważ zawiera on treści zablokowane przez ten serwer. Zawartość ta może być wyświetlana również przez właścicieli serwera."

翻译成中文是：

“此内容无法发布，因为它包含了被该服务器屏蔽的内容。服务器所有者可能仍能看到此内容。”

## 5. 隐私与安全影响

### 数据主权担忧

- **持续收集**：无视用户偏好，遥测数据持续收集。
- **细粒度追踪**：传输详细的系统和使用信息。
- **境外数据处理**：信息被路由到字节跳动（中国公司）的基础设施。
- **唯一身份识别**：多个持久性标识符使得长期追踪成为可能。

### 信任与透明度问题

- **误导性设置**：遥测开关似乎不起作用。
- **未公开的行为**：对数据收集行为没有明确的披露。
- **社区压制**：对技术性批评采取审查而非沟通的方式。


*本分析于2025年7月进行，使用了 Trae IDE PRE-2.0.2 和 2.0.2 版本。网络流量通过标准监控工具捕获，所有发现均可复现。鼓励社区成员自行测试并通过适当渠道分享结果。*

虽然本研究的核心内容为手动撰写，但借助了大语言模型（LLM）来修正蹩脚的英语、语法错误并增强书面表达效果。

Discord: cryptux
x (twitter): https://x.com/CookingCodes
