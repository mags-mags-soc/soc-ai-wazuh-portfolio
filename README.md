# SOC Investigation Portfolio

I am preparing for a SOC Analyst / Blue Team role. In my homelab, I use Wazuh and Windows endpoint telemetry, and I run my SOC AI Assistant on an Ubuntu VM to help explain alerts and plan investigations.

[Application source and setup](https://github.com/mags-mags-soc/ai-soc-assistant) · [Case index](cases/README.md) · [Current case](cases/001-temporary-dll-triage.md)

## Two parts of the project

| Repository | What it contains |
| --- | --- |
| [AI SOC Assistant](https://github.com/mags-mags-soc/ai-soc-assistant) | Application code, setup instructions, tests and development documentation |
| This portfolio | Case notes, evidence status, investigation questions and documented decisions |

## Current work

| Case | Focus | Stage | Disposition |
| --- | --- | --- | --- |
| [001 — Temporary DLL creation](cases/001-temporary-dll-triage.md) | Distinguish file creation from loading or transfer for a temp-directory DLL alert | Closed — evidence collected | Benign, with recorded collection gaps |

Case 001 is closed. The original reported record had rotated out of retention, so I verified the installed rule definition, captured a fresh trigger of the same rule with full telemetry, and checked binary provenance and launch chain on the endpoint. The disposition is recorded together with the collection gaps that remain.

## My lab workflow

| Component | Purpose |
| --- | --- |
| Windows / Sysmon | Collect endpoint telemetry |
| Wazuh | Generate alerts with rule and event context |
| SOC AI Assistant on Ubuntu | Produce an initial assessment and Markdown report |
| Case notes | Track observations, hypotheses, evidence gaps and disposition |

## Method and tooling

My SOC AI Assistant generated the initial report. ChatGPT helped prepare and technically review these documents. I distinguish report-derived statements from raw-event verification; the case records which checks still need evidence.

## Reading this portfolio

Each case includes its review date, source status, technical references and the evidence needed for a decision. Review dates describe document work, not the time of an incident. Underlying operational records are excluded from this public edition.

Start with the [application repository](https://github.com/mags-mags-soc/ai-soc-assistant) for implementation details and the [case index](cases/README.md) for investigation progress.
