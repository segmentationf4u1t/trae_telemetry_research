# Telemetry and Privacy Analysis of Trae IDE

### A Follow-Up Investigation into ByteDance’s VSCode Fork (MK 2)

This is a refined and updated deep dive into Trae, ByteDance’s customized fork of Visual Studio Code.

The initial investigation, conducted on 2025-07-24, focused on telemetry behaviors and issues with community moderation. This follow-up, carried out on 2025-07-31, revisits earlier claims, verifies developments, and provides further technical insights.

---

## Addressing the “Automated Censorship” Claim

Let’s begin with the elephant in the room: the accusation of automated censorship.

After direct contact with the Trae team, I can now confidently clarify — this was not censorship, but a poorly executed moderation mechanism. When I shared my findings in the official community Discord, my messages were auto-flagged due to a string match on the word “tokens,” which seems to be part of an internal keyword blacklist.

While the incident was unfortunate, it was not the result of targeted censorship. Instead, it reveals a misconfigured moderation system, lacking both adequate human oversight and contextual filtering. The staff response was slow and insufficient, but not malicious.

To their credit, Trae’s team reached out promptly after this occurred. They’ve shown willingness to explain the issue, clarify the system behavior, and discuss telemetry transparency — a level of openness that deserves acknowledgment.

---

## 1. Network Traffic and Telemetry Overview

Being a fork of VSCode, Trae inherits most of its core features — including the telemetry configuration UI.

![telemetry](https://i.imgur.com/6z3sXBZ.png)

*Figure 1: Telemetry explicitly disabled in settings (no effect in practice).*

### Primary Endpoints Observed:

* `http://mon-va.byteoversea.com`
* `http://maliva-mcs.byteoversea.com`
* `https://mon-va.byteoversea.com/monitor_browser/collect/batch/?biz_id=marscode_nativeide_us`

---

### 1.1 Telemetry Configuration Testing

The telemetry toggle in Trae is effectively cosmetic. Disabling it has no practical impact — backend processes continue transmitting data regardless.

**Key findings:**

* Disabling telemetry does **not** reduce outbound network activity.
* Persistent connections remain to key ByteDance endpoints.
* During testing, a single telemetry batch reached **53,606 bytes**.
* Over \~7 minutes of normal IDE use, over **500 calls** were logged, totaling approximately **26 MB** of data.

---

## 2. Data Transmission Analysis

### 2.1 Batch Payloads

Despite telemetry being disabled in settings, the application still sends verbose telemetry packets, including detailed hardware and usage metadata.

Example payload:

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
        "user_id": "redacted :)",
        ...
        "cpu": "Intel",
        "os_version": "Microsoft Windows 11 Pro",
        "app_version": "2.0.2"
      }
    }
  ]
}
```

While the data may be part of an authentication or analytics mechanism, the inclusion of low-level hardware identifiers is excessive.

---

### 2.2 User Interaction Telemetry

Granular user activity data is also streamed to `maliva-mcs.byteoversea.com`.

Sample event payload:

```json
{
  "event": "icube_time_on_ide",
  "params": {
    "duration": 1964,
    "activeEditorLanguage": "markdown",
    "workspaceId": "88eda3be...",
    "isMouseOrKeyboardActive": true,
    ...
  }
}
```

These events track:

* Window focus/blur state
* File extensions in use
* Duration of activity in the editor
* AI-assistance toggle state
* Unique identifiers for workspace, machine, and user

---

## 3. Telemetry Scope Summary

Trae continues to transmit a broad set of data even with telemetry opt-out enabled:

| Category                | Details                                               |
| ----------------------- | ----------------------------------------------------- |
| **System Info**         | OS, CPU, RAM, architecture, screen resolution         |
| **Device ID**           | Multiple unique identifiers: user, machine, session   |
| **Usage Data**          | File types, open files, session duration, focus state |
| **Performance Metrics** | Response time, error status, SDK version              |
| **Location & Locale**   | Region, language, timezone offset                     |
| **Workspace Info**      | Project workspace IDs, recent files (obfuscated)      |

---


ByteDance has made efforts toward transparency, and the recent outreach by the Trae team is a step in the right direction. However, disabling telemetry must be meaningful — not decorative.

All traffic was captured using standard monitoring tools. Findings are reproducible, and further testing is encouraged.

---

Written and researched by hand.
Language cleanup and formatting powered by LLM.

> Discord: **cryptux**
> X (Twitter): [@CookingCodes](https://x.com/CookingCodes)
