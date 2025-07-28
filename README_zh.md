好的，这是您提供的技术分析报告的中文翻译。

---

<img width="684" height="426" alt="image" src="https://github.com/user-attachments/assets/10235641-331c-41b6-9c8f-bc126e660ccd" />

# Trae IDE 性能与遥测分析：深入剖析字节跳动的 VSCode 分支

## 执行摘要

本分析报告旨在审视 Trae IDE（字节跳动基于 Visual Studio Code 的分支版本）中发现的、令人担忧的性能与隐私问题。主要发现包括：过度的资源消耗（其进程数高达 33 个，而 VSCode 仅为 9 个）、即使用户在设置中禁用了遥测功能，数据依然会持续传输，以及存疑的社区管理行为。

## 1. 背景与方法论

在为个人项目评估开发环境时，我对三款流行的 IDE 进行了对比分析：Visual Studio Code、Cursor 和 Trae（字节跳动的 VSCode 分支）。分析结果显示，它们在资源使用和网络行为上存在显著差异，值得进行更深入的调查。

**测试环境：**
- 操作系统：Microsoft Windows 11 专业版 (Pro)
- CPU：Intel Core™ i7-14700KF
- 内存 (RAM)：64GB
- 测试项目：每个 IDE 加载完全相同的代码库
- 监控工具：System Informer, Fiddler Everywhere

## 2. 资源消耗分析

### 进程数量与内存占用

初步测试揭示了资源消耗上的巨大差异：

| IDE      | 进程数 | 内存占用 | 性能影响 | 项目大小 |
|----------|---------------|--------------|-------------------| -----|
| VS Code  | 9             | ~0.9 GB      | 基准          | 107 个文件 Rust + TS|
| Cursor   | 11            | ~1.9 GB      | 2.1 倍内存       | 107 个文件 Rust + TS|
| **Trae** | **33**        | **~5.7 GB**  | **6.3 倍内存**   | 107 个文件 Rust + TS|

**更新后指标 (v2.0.2):**
- 进程数从 33 个减少到约 13 个
- 内存占用降至约 2.5 GB

| IDE      | 进程数 | 内存占用 | 性能影响 | 版本 | 项目大小 |
|----------|---------------|--------------|----------------|------------|--------------|
| VS Code  | 9             | ~0.9 GB      | 基准          | 1.102.2  | 107 个文件 Rust + TS    |
| Cursor   | 11            | ~1.9 GB      | 2.1 倍内存       | 1.2.4  | 107 个文件 Rust + TS  |
| **Trae** | **11**        | **~1.9 GB**  | **2.1 倍内存**    | **2.0.2** | 107 个文件 Rust + TS |

*不含 node_modules 和 target 目录



*图 1：Trae 产生的进程数是 VSCode 的 3.7 倍，消耗的内存是其 6.3 倍*

### 社区反馈与部分解决

在 Trae 的 Discord 服务器上报告此问题后（[参考链接](https://discord.com/channels/1320998163615846420/1335032920850825391/1397999824389017742)），开发团队承认了该问题。2.0.2 版本解决了一些问题，将进程数减少了约 20 个。然而...

## 3. 网络流量与遥测调查

接下来，网络监控揭示了其与字节跳动基础设施之间存在持续的出站连接：



*图 2：Trae 的网络活动，显示其定期连接到字节跳动的服务器*

**已识别的主要端点：**
- `http://mon-va.byteoversea.com`
- `http://maliva-mcs.byteoversea.com`
- `https://mon-va.byteoversea.com/monitor_browser/collect/batch/?biz_id=marscode_nativeide_us`

### 遥测配置测试

#### 禁用遥测

我尝试通过标准设置界面禁用遥测功能：



*图 3：在用户设置中禁用遥测*

#### 意外的结果

禁用遥测并未减少网络活动。相反，它：
- 维持了与 `mon-va.byteoversea.com` 和 `maliva-mcs.byteoversea.com` 的现有连接。
- **增加**了到批量收集端点 (batch collection endpoint) 的请求频率。

在禁用遥测的情况下，这些调用仍然会迅速累积——在测试期间，单次批量传输的数据量可达 53,606 字节。在活跃使用编辑器时，我观察到在约 7 分钟内产生了约 500 次调用，总计传输了高达 26 MB 的数据。


*图 4：调用次数增加*

## 4. 数据传输分析

### 批量遥测载荷

即便禁用了遥测，Trae 仍持续传输详细的使用数据：

```json
{
	"ev_type": "batch",
	"list": [
		{
			"ev_type": "custom",
			"payload": {
				"name": "icube_ai_ckg_request",
				"type": "event",
				"metrics": { "cost_time": 5 },
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
虽然这可能与身份验证有关，但它包含了大量的硬件规格信息，这是为什么？

### 用户活动追踪

另一个端点 (`maliva-mcs.byteoversea.com`) 接收更细粒度的用户交互数据：

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

该遥测系统捕获：
- **系统信息**：硬件规格、操作系统详情、架构
- **使用模式**：活跃时间、会话时长、功能使用情况
- **性能指标**：响应时间、资源消耗
- **唯一标识符**：机器 ID、用户 ID、设备指纹
- **工作区详情**：项目信息、文件路径（已混淆）

👉 [在 Streamable 上观看网络请求视频](https://streamable.com/e/agr0a2?loop=0)

## 5. 社区管理问题

### 自动化的内容审查

当我尝试在 Trae 的 Discord 服务器上讨论这些发现时，我被直接禁言了。



https://discord.com/channels/1320998163615846420/1335032920850825391/1398374824987852891

<s>

1. **关键词过滤**：“track” (追踪) 这个词被添加到了自动黑名单中。
</s>

2. **自动惩罚**：提及追踪问题会立即触发 7 天的禁言。
3. **压制技术讨论**：合法的安全顾虑被当作破坏性行为处理。

## 25年7月28日更新

禁言发生三天后，社区管理员“Rdap”确认该禁言是自动触发的，并且现在似乎已经解除。[点此查看详情](https://github.com/segmentationf4u1t/trae_telemetry_research/issues/3)

然而，对于拥有相应权限的用户来说，触发自动警告的消息在 Discord 聊天中仍然可见。我不确定为什么警告没有被立即解除，尤其是考虑到当时工作人员清楚地看到了事件的经过。

禁言的截图显示为波兰语：

"Nie można opublikować tego elementu, ponieważ zawiera on treści zablokowane przez ten serwer. Zawartość ta może być wyświetlana również przez właścicieli serwera."

翻译成中文是：

“此项目无法发布，因为它包含此服务器屏蔽的内容。此内容可能仍对服务器所有者可见。”

--------------------------------------------------------------------------------------------------------------

## 6. 隐私与安全影响

### 数据主权担忧

- **持续收集**：无视用户偏好，持续收集遥测数据。
- **精细追踪**：传输详细的系统和使用信息。
- **境外数据处理**：信息被路由到字节跳动（一家中国公司）的基础设施。
- **唯一身份识别**：通过多个持久性标识符实现长期追踪。

### 信任与透明度问题

- **误导性设置**：遥测开关似乎只是摆设，并不起作用。
- **未文档化的行为**：没有明确披露数据收集的做法。
- **压制社区声音**：对技术性批评采取审查而非沟通的方式。

**核心要点：**
- 资源占用是 VSCode 基准的 6 倍（在 2.0.2 版本中与 Cursor 相当）。
- 遥测设置似乎只是表面功夫，而非实际功能。
- 社区反馈机制因内容审查而受到损害。
- 数据收集实践缺乏透明度和用户控制。

---

*本分析于 2025 年 7 月使用 Trae IDE PRE-2.0.2 和 2.0.2 版本进行。网络流量通过标准监控工具捕获，所有发现均可复现。鼓励社区成员自行测试并通过适当渠道分享结果。*

尽管本研究的核心内容为手动撰写，但借助了大型语言模型（LLM）来修正蹩脚的英语、语法错误并提升行文的流畅度。

Discord: cryptux
x (twitter): https://x.com/CookingCodes
