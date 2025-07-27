
# Performance and Telemetry Analysis of Trae IDE: A Deep Dive into ByteDance's VSCode Fork

## Executive Summary

This analysis examines concerning performance and privacy issues discovered in Trae IDE, ByteDance's fork of Visual Studio Code. Key findings include excessive resource consumption (33 processes vs 9 in VSCode), persistent telemetry transmission despite user settings, and concerning community management practices.

## 1. Background and Methodology

During evaluation of development environments for a personal project, I conducted a comparative analysis of three popular IDEs: Visual Studio Code, Cursor, and Trae (ByteDance's VSCode fork). This analysis revealed significant discrepancies in resource usage and network behavior that warranted deeper investigation.

**Testing Environment:**
- OS: Microsoft Windows 11 Pro
- CPU: Intel Core™ i7-14700KF
- RAM: 64GB
- Test Project: Identical codebase loaded in each IDE
- Monitoring Tools: System Informer, Fiddler Everywhere

## 2. Resource Consumption Analysis

### Process Count and Memory Usage

Initial testing revealed dramatic differences in resource consumption:

| IDE      | Process Count | Memory Usage | Performance Impact |
|----------|---------------|--------------|-------------------|
| VS Code  | 9             | ~0.9 GB      | Baseline          |
| Cursor   | 11            | ~1.9 GB      | 2.1x memory       |
| **Trae** | **33**        | **~5.7 GB**  | **6.3x memory**   |

![Process Usage Comparison](https://i.imgur.com/jqYEBM7.png)
*Figure 1: Trae spawns 3.7x more processes than VSCode and consumes 6.3x more memory*

### Community Feedback and Partial Resolution

After reporting this issue on Trae's Discord server ([reference](https://discord.com/channels/1320998163615846420/1335032920850825391/1397999824389017742)), the development team acknowledged the problem. Version 2.0.2 addressed some concerns, reducing the process count by approximately 20 processes. However, Trae still maintains significantly higher resource usage than comparable IDEs.

**Post-Update Metrics (v2.0.2):**
- Reduced from 33 to ~13 processes
- Memory usage down to  ~2.5 GB

## 3. Network Traffic and Telemetry Investigation

### Initial Discovery

Network monitoring revealed persistent outbound connections to ByteDance infrastructure:

![Network Requests](https://i.imgur.com/crkKdXF.png)
*Figure 2: Trae's network activity showing regular connections to ByteDance servers*

**Primary Endpoints Identified:**
- `http://mon-va.byteoversea.com`
- `http://maliva-mcs.byteoversea.com`
- `https://mon-va.byteoversea.com/monitor_browser/collect/batch/?biz_id=marscode_nativeide_us`

### Telemetry Configuration Testing

#### Disabling Telemetry

I attempted to disable telemetry through the standard settings interface:

![Telemetry Settings](https://i.imgur.com/BYjJU0w.png)
*Figure 3: Telemetry disabled in user settings*


#### Unexpected Results

**Critical Finding:** Disabling telemetry did not reduce network activity. Instead, it:
- Maintained existing connections to `mon-va.byteoversea.com` and `maliva-mcs.byteoversea.com`
- **Increased** request frequency to the batch collection endpoint

![Increase of calls](https://i.imgur.com/BwdiwC4.png)
*Figure 4: Increase of calls*

## 4. Data Transmission Analysis

### Batch Telemetry Payload

Even with telemetry disabled, Trae continues transmitting detailed usage data:

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

### User Activity Tracking

Additional endpoint (`maliva-mcs.byteoversea.com`) receives granular user interaction data:

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

### Data Collection Scope

The telemetry system captures:
- **System Information**: Hardware specs, OS details, architecture
- **Usage Patterns**: Active time, session duration, feature usage
- **Performance Metrics**: Response times, resource consumption
- **Unique Identifiers**: Machine ID, user ID, device fingerprints
- **Workspace Details**: Project information, file paths (obfuscated)

<iframe allow="fullscreen" allowfullscreen height="500" src="https://streamable.com/e/agr0a2?loop=0" width="500" style="border:none;"></iframe>
## 5. Community Management Concerns

### Automated Censorship

When I attempted to discuss these findings on Trae's Discord server, i got spanked with gag-hammer
![mutedlmao](https://i.imgur.com/wvAOzuG.png)
https://discord.com/channels/1320998163615846420/1335032920850825391/1398374824987852891

1. **Keyword Filtering**: The word "track" was added to an automated blacklist
2. **Automatic Punishment**: Mentioning tracking issues triggered an instant 7-day mute
3. **Suppression of Technical Discussion**: Legitimate security concerns were treated as disruptive behavior



## 6. Privacy and Security Implications

### Data Sovereignty Concerns

- **Persistent Collection**: Telemetry continues despite user preferences
- **Granular Tracking**: Detailed system and usage information transmitted
- **Foreign Data Processing**: Information routed to ByteDance (Chinese company) infrastructure
- **Unique Identification**: Multiple persistent identifiers enable long-term tracking

### Trust and Transparency Issues

- **Misleading Settings**: Telemetry toggle appears non-functional
- **Undocumented Behavior**: No clear disclosure of data collection practices
- **Community Suppression**: Technical criticism met with censorship rather than engagement

## 7. Recommendations

### For Users
1. **Immediate Action**: Consider alternative IDEs if privacy is a concern
2. **Network Monitoring**: Use firewall rules to block ByteDance domains if continuing use
3. **Regular Review**: Monitor process usage and network activity
4. **Data Awareness**: Understand that usage data may be collected regardless of settings

### For ByteDance/Trae Team
1. **Honor User Preferences**: Implement genuine telemetry opt-out functionality
2. **Transparency**: Provide clear documentation of data collection practices
3. **Performance Optimization**: Address excessive resource consumption
4. **Community Engagement**: Replace censorship with constructive dialogue

## 8. Conclusion

This analysis reveals significant concerns with Trae IDE's approach to user privacy and system resources. The combination of excessive resource usage, persistent telemetry despite user settings, and defensive community management creates a troubling pattern that users should carefully consider.

While Trae offers some interesting features as a VSCode fork, the current implementation raises questions about ByteDance's commitment to user privacy and transparent development practices. Users requiring strict data privacy or optimal system performance may want to consider established alternatives like standard VSCode or other privacy-focused development environments.

**Key Takeaways:**
- Resource usage is 6x higher than VSCode baseline
- Telemetry settings appear to be cosmetic rather than functional
- Community feedback mechanisms are compromised by censorship
- Data collection practices lack transparency and user control

---

*This analysis was conducted in July 2025 using Trae IDE version  PRE-2.0.2 and 2.0.2. Network traffic was captured using standard monitoring tools, and all findings are reproducible. Community members are encouraged to conduct their own testing and share results through appropriate channels.*
