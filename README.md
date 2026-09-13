# AI-Assisted SOC Triage with Wazuh

A personal SOC homelab portfolio exploring how AI-generated alert analysis can support an evidence-based investigation.

**First case:** reviewing a temporary DLL creation alert.  
**Status:** open — further evidence is required for a final verdict.

[Read the case study](cases/001-temporary-dll-triage.md)

## Lab workflow

| Component | Role |
| --- | --- |
| Windows endpoint / Sysmon | Endpoint event telemetry |
| Wazuh | Detection and alert context |
| SOC AI Assistant on an Ubuntu VM | Initial analysis and report generation |
| Document review | Assess claims, compare hypotheses and define follow-up checks |

The lab environment is described by its owner. This repository documents a review of an exported alert report; the underlying integration and application code have not been independently inspected.

## What the case demonstrates

- Separating an alert's severity from a confirmed incident verdict.
- Comparing potential malicious staging with expected application unpacking.
- Checking whether an ATT&CK mapping is supported by available evidence.
- Treating AI-generated confidence scores as unverified unless validated.
- Planning the evidence collection needed to resolve an alert.
- Publishing a case study with clear limitations and without operational records.

## Current finding

The reviewed export describes a file creation but lacks the evidence needed to confirm compromise or close the alert as benign. The case remains open.

## Repository scope

This is a documentation portfolio. It does not contain an installable release of the SOC AI Assistant, its source code or the underlying incident export.

The original assistant generated the initial analysis. This write-up and its review were prepared with ChatGPT assistance. No live endpoint investigation, malware removal or measured detection improvement is claimed.

## Next steps

- Validate the original event records privately.
- Verify file provenance and correlate related process and network activity.
- Update the case with an evidence-backed disposition when those checks are completed.

Official technical references are linked directly in the [case study](cases/001-temporary-dll-triage.md).
