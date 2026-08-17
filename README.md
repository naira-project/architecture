# Naira - Architecture

## About


[![Docs](https://img.shields.io/badge/docs-architecture%20%26%20design-blue)](#architecture--design-documentation)  
[![RFC](https://img.shields.io/badge/RFC-request%20for%20comments-orange)](#review-process)  
[![ADR](https://img.shields.io/badge/ADR-architecture%20decision%20records-purple)](#when-to-use-rfc-vs-adr-vs-ddr)  
[![DDR](https://img.shields.io/badge/DDR-design%20decision%20records-green)](#when-to-use-rfc-vs-adr-vs-ddr)

This repository contains the architectural and design documentation for the [Naira Project](https://github.com/naira-project).

We use three complementary document types to capture important proposals and decisions:

- **Requests for Comments (RFCs)** — propose and discuss larger features or cross-cutting changes before implementation begins. RFCs are more comprehensive than ADRs and are used to gather feedback from the broader team. A single RFC can lead to multiple ADRs and/or DDRs as the design is refined into concrete decisions.
- **Architecture Decision Records (ADRs)** — document significant technical and architectural decisions, including their context, alternatives, and consequences. ADRs follow the [MADR](https://github.com/adr/madr) template.
- **Design Decision Records (DDRs)** — document UI/UX and product design decisions, including interaction patterns, layout choices, and user experience tradeoffs.

In short:

- **RFCs** describe and align on **proposed changes**
- **ADRs** describe **how the system is built**
- **DDRs** describe **how the product is experienced**

---

## Repository Structure

```
architecture/
├── rfc/
│   ├── TEMPLATE.md
│   ├── RFC-001.md
│   └── ...
├── adr/
│   ├── TEMPLATE.md
│   ├── ADR-001.md
│   └── ...
└── ddr/
    ├── TEMPLATE.md
    ├── DDR-001.md
```

## Creating a New Record

1. Create a new file in the appropriate directory:
- `rfc/` — Requests for Comments
- `adr/` — Architecture Decision Records
- `ddr/` — Design Decision Records
2. Use the next available number and a short descriptive name in each directory, for example:
   - `RFC-001-<name>.md`
   - `ADR-001-<name>.md`
   - `DDR-001-<naame>.md`
2. Start from the corresponding `TEMPLATE.md`.
3. Open a pull request for review.

## Review Process

### RFCs

RFCs are used to propose and discuss larger changes before implementation begins.  
They are intended to gather feedback from the broader team and help shape the final solution.

### ADRs

ADRs capture important technical decisions and are reviewed by the relevant engineering stakeholders.  
An ADR is considered accepted once the pull request is approved and merged.

### DDRs

DDRs capture design and UX decisions and are reviewed by relevant product, design, and engineering stakeholders.  
They help align teams on user-facing decisions before or during implementation.

---

## Relationship Between RFCs, DDRs, and ADRs

Some initiatives begin with an **RFC** and later result in one or more **ADRs** and/or **DDRs**.

For example:

- An RFC may propose a new settings experience, including user flow and technical constraints.
- A DDR may define the UX and interaction design.
- One or more ADRs may then define the implementation approach required to support it.

This separation helps keep:

- **proposal and discussion** clear in RFCs
- **design intent** clear in DDRs
- **technical implementation** clear in ADRs

---

## Contribution Guidelines

When creating a new record:

- Keep the document focused on a single decision or a closely related set of decisions.
- Include the context, considered alternatives, and rationale.
- Write clearly and concisely.
- Link related RFCs, ADRs, and DDRs when appropriate.

---

## Need Help Deciding?

If you are unsure which record type to use:

- If the change is **large, cross-cutting, or still under discussion**, start with an **RFC**
- If it affects **system architecture, technical implementation, or infrastructure**, use an **ADR**
- If it affects **layout, interaction, usability, or visual design**, use a **DDR**

## Process

RFCs are open for discussion and feedback from all contributors. Use the pull request review process to comment, suggest changes, and iterate on the proposal.

ADRs require approval from the **Technical Steering Committee (TSC)**, who are the [code owners](CODEOWNERS) of this repository. An ADR is considered accepted once the pull request is approved and merged.


<p align="center"><img alt="Bundesministerium für Wirtschaft und Energie (BMWE)-EU funding logo" src="https://apeirora.eu/assets/img/BMWK-EU.png" width="400"/></p>