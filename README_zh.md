<img width="684" height="426" alt="image" src="https://github.com/user-attachments/assets/10235641-331c-41b6-9c8f-bc126e660ccd" />


# Trae IDE 性能与遥测分析：深入探索字节跳动 VSCode 分支

## 概要说明

本分析考察了字节跳动 Visual Studio Code 分支 Trae IDE 中发现的相关性能和隐私问题。
主要发现包括资源消耗过度（33 个进程相较于 VSCode 的 9 个）、无视用户设置持续传输遥测数据，以及令人担忧的社区管理方式。

## 1. 背景与研究方法

在评估个人项目开发环境时，我对比分析了三个流行的 IDE：Visual Studio Code、Cursor 和 Trae（字节跳动的 VSCode 分支）。这项分析揭示了资源使用和网络行为上的显著差异，需要进行更深入的调查。

**测试环境：**
- 操作系统：Microsoft Windows 11 Pro
- CPU: Intel Core™ i7-14700KF
- RAM: 64GB
- 测试项：每个 IDE 中加载相同的代码库
- 监控工具：System Informer, Fiddler Everywhere

## 2. 资源消耗分析

### 进程数量和内存使用

初步测试显示资源消耗存在巨大差异：

| IDE      | 进程数量 | 内存使用 | 性能影响 |
|----------|---------------|--------------|-------------------|
| VS Code  | 9             | ~0.9 GB      | 基准          |
| Cursor   | 11            | ~1.9 GB      | 2.1倍内存       |
| **Trae** | **33**        | **~5.7 GB**  | **6.3倍内存**   |

![Process Usage Comparison](https://i.imgur.com/jqYEBM7.png)

*图 1：Trae 比 VSCode 多产生 3.7 倍进程，并消耗 6.3 倍内存*

### 社区反馈和部分解决

在 Trae 的 Discord 服务器([参见](https://discord.com/channels/1320998163615846420/1335032920850825391/1397999824389017742))上报告此问题后，开发团队承认了这个问题。版本 2.0.2 解决了一些问题，将进程数量减少了大约 20 个进程。然而，Trae 仍然比同类 IDE 有着显著更高的资源使用率——无论是由于优化不佳还是内存泄漏，我不知道，建议进一步研究。

**更新后指标 (v2.0.2):**
- 从33个减少到约13个进程
- 内存使用降至约2.5GB



## 3. 网络流量和遥测调查

现在，有趣的部分开始了，网络监测发现系统持续向字节跳动的基础设施发出连接请求。

![Network Requests](https://i.imgur.com/crkKdXF.png)

*图 2：Trae 的网络活动显示定期连接到字节跳动的服务器*

 
**已识别的主要端点：**
- `http://mon-va.byteoversea.com`
- `http://maliva-mcs.byteoversea.com`
- `https://mon-va.byteoversea.com/monitor_browser/collect/batch/?biz_id=marscode_nativeide_us`



### 遥测配置测试

#### 禁用遥测

我尝试通过标准设置界面禁用遥测

![Telemetry Settings](https://i.imgur.com/BYjJU0w.png)

*图3：用户设置中禁用遥测*


#### 意外结果

禁用遥测并未减少网络活动。相反，它：
- 维持了与 `mon-va.byteoversea.com` 和 `maliva-mcs.byteoversea.com`的现有连接
- **增加** 了批量收集端点的请求频率

在遥测功能关闭的情况下，这些调用仍然可以快速累积——在测试中，单个批次曾达到 53,606 字节。在积极使用编辑器时，我观察到在约 7 分钟内大约有 500 次调用，在那短暂的时间内总共传输了高达 26 MB 的数据。

![Increase of calls](https://i.imgur.com/BwdiwC4.png)
*图4：请求量增加*


## 4. 数据传输分析

### 批量遥测有效载荷

即使遥测功能已禁用，Trae 仍继续传输详细的使用数据：

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
				"user_id": "redacted :)",
				"device_id": "redacted :)",
				"session_id": "redacted :)",
				"env": "prod",
				"timestamp": 1753636703370,
				"sdk_version": "1.12.7",
				"sdk_name": "SDK_SLARDAR_WEB",
				"pid": "AI Agent",
				"view_id": "AI Agent 1753636703370",
				"context": {
					"tracing_id": "redacted :)",
					"span_id": "59a833c5359d4978",
					"product": "nativeIDE",
					"machine_id": "redacted :)",
					"user_id": "redacted :)",
					"user_name": "Redacted :)",
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
					"biz_user_id": "redacted :)",
					"user_is_login": "true",
					"device_id": "redacted :)",
					"user_unique_id": "redacted :)",
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
虽然这也可能与身份验证有关，但它包含大量硬件规格，为什么？

### 用户活动跟踪

额外的端点 (`maliva-mcs.byteoversea.com`) 接收细粒度的用户交互数据：

```json
[
	{
		"events": [
			{
				"event": "_be_active",
				"params": "{\"start_time\":1753636688373,\"end_time\":1753636688373,\"url\":\"vscode-file://vscode-app/c:/Users/segfault/AppData/Local/Programs/Trae/resources/app/out/vs/code/electron-sandbox/workbench/workbench.html\",\"referrer\":\"\",\"title\":\"read.md - tauri-app - Trae\",\"event_index\":1753636640233}",
				"local_time_ms": 1753636688374,
				"is_bav": 0,
				"session_id": "redacted :)"
			},
			{
				"event": "icube_time_on_ide",
				"params": "{\"entryTime\":1753636462123,\"now\":1753636687360,\"workspaceId\":\"88eda3be5f237760cfadaa7abbd89c24\",\"duration\":1964,\"lastActiveTime\":1753636685396,\"isEditorFocused\":false,\"activeEditorTypeId\":\"workbench.editors.files.fileEditorInput\",\"isMouseOrKeyboardActive\":true,\"isAiActive\":false,\"activeEditorProviderId\":\"\",\"activeEditorLanguage\":\"markdown\",\"activeEditorFileExt\":\".md\",\"isFocus\":\"not_focus\",\"isVisible\":\"hidden\",\"event_index\":1753636640232}",
				"local_time_ms": 1753636688372,
				"is_bav": 0,
				"session_id": "redacted :)"
			}
		],
		"user": {
			"user_unique_id": "redacted :)",
			"user_id": "redacted :)",
			"user_is_login": true,
			"device_id": "redacted :)"
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

遥测系统捕获：
- **系统信息**: 硬件规格、操作系统详情、架构
- **使用模式**: 活跃时间、会话时长、功能使用情况
- **性能指标**: 响应时间、资源消耗
- **唯一标识符**: 机器ID，用户ID，设备指纹
- **工作区详情**: 项目信息，文件路径（已混淆）


👉 [在 Streamable 上观看网络请求](https://streamable.com/e/agr0a2?loop=0)

## 5. 社区管理担忧

### 自动化审查

当我尝试在 Trae 的 Discord 服务器上讨论这些发现时，却被“禁言大锤”伺候了一顿。
![mutedlmao](https://i.imgur.com/wvAOzuG.png)
![notgood](https://i.imgur.com/phpvlVS.png)
https://discord.com/channels/1320998163615846420/1335032920850825391/1398374824987852891

在讨论发生之后添加了审核黑名单，最初禁言是手动操作的。
1. **关键词过滤**: 单词"track"被添加到自动黑名单中
2. **自动处罚**: 提及跟踪问题触发了立即的 7 天禁言
3. **技术讨论的压制**: 正当的安全关切被视为干扰行为


## 6. 隐私和安全影响

### 数据主权关切

- **持久性收集**: 不论用户偏好如何，遥测数据仍会持续收集
- **粒度跟踪**: 传输详细的系统和使用信息
- **外部数据处理**: 信息被路由至字节跳动（中国公司）的基础设施
- **唯一性识别**: 多个持久性标识符实现长期跟踪

### 信任与透明度问题

- **误导性设置**: 遥测开关看似无法使用
- **未记录的行为**: 未明确披露数据收集行为
- **社区压制**: 技术批评遭到审查而非回应



**关键要点：**
- 资源使用量比 VSCode 基准高出 6 倍
- 遥测设置似乎更多是装饰性的而非功能性的
- 社区反馈机制因审查而受损
- 数据收集实践缺乏透明度和用户控制

---

*这项分析于 2025 年 7 月使用 Trae IDE 版本 PRE-2.0.2 和 2.0.2 进行。网络流量通过标准监控工具捕获，所有发现均可重复。鼓励社区成员自行进行测试并通过适当渠道分享结果。*

虽然这项研究的核心部分是手写的，但一个 LLM 已经将其重写以修复错误的英语、语法并增强研究的语言表达方面。

Discord: cryptux
x (twitter): https://x.com/CookingCodes
