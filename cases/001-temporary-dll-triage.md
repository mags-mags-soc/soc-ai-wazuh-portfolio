# Case 001 — Temporary DLL Creation

**Review date:** 2026-09-13 (UTC; document review date)
**Source event:** 2026-09-13T12:44:35Z
**Stage:** closed — evidence collected
**Disposition:** benign — expected application behavior, with recorded collection gaps
**Detection context:** Wazuh rule `92213` (verified in `0830-sysmon_id_11.xml`), Sysmon Event ID 11 (FileCreate)

## Starting point

My SOC AI Assistant produced a report describing a Windows process writing a DLL to a temporary directory. Its analysis suggested possible malicious staging and discussed Ingress Tool Transfer.

My question for this case was: **does the file creation belong to expected application behavior, or is there evidence of malicious transfer or DLL abuse?**

I could not retrieve the original alert record — the manager's `alerts.json` had rotated and the date in question was no longer retained. Instead I captured a fresh trigger of the same rule with complete telemetry and investigated that. The two are separate events. Everything below describes the event I collected, not the one the assistant originally reported.

## Evidence register

"Verified" below means checked against the original event, the installed rule definition or endpoint state — not repeated from the exported report.

| Item | Finding | Source | Status |
| --- | --- | --- | --- |
| Detection rule | `92213`, level 15, `if_group: sysmon_event_11`, single `targetFilename` PCRE2 condition | Installed ruleset `0830-sysmon_id_11.xml` | Verified |
| Event type | Sysmon Event ID 11 (FileCreate), `RuleName: DLL` | Raw alert | Verified |
| Writing process | Signed vendor desktop application under `C:\Program Files\WindowsApps\` (MSIX package) | Raw alert | Verified |
| Target file | `%LOCALAPPDATA%\Temp\chromium_chrome_Unpacker_BeginUnzipping<pid>_<random>\_platform_specific\win_x64\widevinecdm.dll` | Raw alert | Verified |
| Directory ↔ process correlation | The PID embedded in the unpacker directory name matches the `ProcessId` in the same event | Raw alert | Verified |
| Binary provenance | Authenticode `Valid`, signer `CN="OpenAI OpCo, LLC", O="OpenAI OpCo, LLC", L=San Francisco, S=California, C=US` | `Get-AuthenticodeSignature` on the endpoint | Verified |
| Launch chain | Main process started by `C:\Windows\System32\sihost.exe`, Medium integrity, 3m33s before the file event; child renderer processes at Low integrity | Sysmon EID 1 on the endpoint | Verified |
| DLL hash and signature | File no longer present at review time | Endpoint check | Not collectable |
| DLL load into a process | Sysmon EID 7 is not collected on this host | Endpoint log inventory | Collection gap |
| Temp directory cleanup | Sysmon EID 23 / 26 are not collected on this host | Endpoint log inventory | Collection gap |
| Original reported event | `alerts.json` rotated; record not retained | Manager log inventory | Not retrievable |

## What the evidence shows

The writing process is a Chromium/Electron desktop application distributed as a signed MSIX package. Chromium unpacks bundled components into a per-process temporary directory using the `chromium_chrome_Unpacker_BeginUnzipping<pid>_<random>` naming convention, and `widevinecdm.dll` is the Widevine DRM module shipped inside such bundles.

Three independent observations support the expected-behavior reading:

1. **Provenance.** The parent binary carries a valid Authenticode signature from the vendor, and MSIX installation enforces signature validation at install time.
2. **Internal consistency.** The PID embedded in the unpacker directory name matches the process that wrote the file, in the same event. A file dropped by an unrelated process would not produce this correspondence.
3. **User-initiated launch.** The process tree roots at `sihost.exe`, which starts packaged applications in response to user interaction. There is no scheduled task, service, script interpreter or remote-execution parent in the chain.

The file was written to a directory the application created for its own process, from a bundle it already shipped, in a session the user started.

## What the evidence does not show

I did not establish that the DLL was loaded, because Event ID 7 is not collected on this host. I did not establish that the temporary directory was cleaned up on exit, because Event ID 23/26 are not collected either. The file was gone by review time, but absence at review time does not by itself demonstrate deliberate cleanup — it is consistent with it, not proof of it.

I also did not analyse the DLL itself. No hash or signature could be collected because the file no longer existed. A valid parent signature does not authenticate a file that parent writes.

Enabling this telemetry now would not recover the historical events. These gaps are recorded rather than resolved.

## Disposition

**Benign — expected application behavior.**

The claim is not that the DLL was proven clean. The claim is that a signed vendor application unpacked a component from its own bundle into its own per-process temporary directory during a user-initiated session, that every collected artifact is consistent with that explanation, and that no artifact supports transfer from an external system or abuse of DLL resolution. The unresolved items above are stated rather than assumed away.

## ATT&CK assessment

| Technique | Status | Basis |
| --- | --- | --- |
| [T1105 — Ingress Tool Transfer](https://attack.mitre.org/techniques/T1105/) | Not supported by the evidence | The observed behavior is a local write from a process's own package. No external transfer or C2 artifact was observed. |
| [T1574.001 — Hijack Execution Flow: DLL](https://attack.mitre.org/techniques/T1574/001/) | Not supported by the evidence | Would require evidence of attacker control over which DLL a process loads. Load telemetry is not collected here, so this was neither observed nor excludable. |

**The T1105 mapping does not originate with the AI report.** It is hard-coded in the `<mitre>` block of rule `92213` in the upstream Wazuh ruleset, so every alert this rule produces carries it — tactic *Command and Control*, technique *Ingress Tool Transfer*. The assistant inherited the label from the detection, then reasoned from it.

This matters because the rule's own match condition cannot support that mapping. The rule matches on `targetFilename` alone: any executable-extension file written under `%LOCALAPPDATA%\Temp\`. A local file write is not evidence of tool transfer from an external system. The technique label is attached upstream of any evidence about how the file arrived.

The useful correction to the initial assessment is therefore to separate **creation**, **transfer** and **loading**. Each requires different evidence. This case had creation telemetry only.

## Detection improvement

Three issues, in order of significance:

**1. Mapping defect.** `92213` asserts T1105 on a condition that cannot establish transfer. The mapping should be removed or replaced with one the match condition can support.

**2. No context conditions.** The rule tests a single field. It has no `image`, `parentImage` or command-line condition, so every application that unpacks a component to `%TEMP%` triggers it — a common pattern for Electron, Chromium-based and packer-built software. A narrow, documented `if_sid` override keyed on verified writing processes would reduce this without creating a blanket temp-directory exclusion.

**3. Severity.** Level 15 is near the top of the Wazuh scale. A single-field match with no execution or provenance context does not justify it. This is what drove the high severity in the initial report.

**4. Collection gap.** The rule detects file creation but the host collects no image-load telemetry, so the follow-up question the rule raises cannot be answered. Detection and collection are misaligned.

**5. Regex anchoring.** The extension list is not anchored with `$`, so `.js` also matches paths ending in `.json`. Reported upstream in 2023 ([#18465](https://github.com/wazuh/wazuh/issues/18465)) and still present in v4.14.7.

### Reported upstream

| Finding | Report |
| --- | --- |
| T1105 mapping not supported by the rule's match condition | [wazuh/wazuh#39227](https://github.com/wazuh/wazuh/issues/39227) (opened from this case) |
| Missing `$` anchor in the extension pattern | [wazuh/wazuh#18465](https://github.com/wazuh/wazuh/issues/18465) (existing report; confirmed still present in v4.14.7) |

## Next investigation steps

| Order | Action | Result to record |
| --- | --- | --- |
| 1 | Enable Sysmon EID 7 with a scoped filter, and EID 23/26 for temp paths | Whether load and deletion become answerable for future triggers |
| 2 | Re-trigger the rule with telemetry enabled | A complete creation → load → cleanup sequence for the same behavior |
| 3 | Draft and test a narrow local override for verified writing processes | Override scope, what remains detected, regression evidence |
| 4 | Track the upstream reports | Maintainer response and any ruleset change |

## Decision log

| Review date | Decision | Reason |
| --- | --- | --- |
| 2026-09-13 | Case opened; kept unresolved pending evidence | The exported report alone could not distinguish the competing hypotheses |
| 2026-09-13 | Original record declared not retrievable; investigated a fresh trigger of the same rule instead | `alerts.json` rotation; the substitution is recorded rather than conflated |
| 2026-09-13 | Closed as benign with recorded gaps | Provenance, internal consistency and launch chain verified; no supporting evidence for transfer or DLL abuse |
| 2026-09-13 | Reported the mapping defect upstream | The issue originates in the shipped ruleset, not in this environment, so it belongs upstream rather than in a local override |

## Scope and limitations

Completed work: verified the installed rule definition, captured and analysed a raw alert, checked binary provenance and launch chain on the endpoint, inventoried Sysmon collection coverage, and documented the disposition with its gaps.

The analysed event is not the event the assistant originally reported; that record was no longer retained. Host identifiers and the local user name are normalised in this public edition. Underlying operational records are excluded.

[Case index](README.md) · [Portfolio and method](../README.md#method-and-tooling) · [Application source](https://github.com/mags-mags-soc/ai-soc-assistant)
