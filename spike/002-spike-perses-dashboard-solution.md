---
creation date: 2026-06-02
contributors: "@kocakkaan"
---

# Perses Evaluation - Recommendation

## Overview

Perses is a GitOps-friendly, Dashboard-as-Code tool for building customized dashboards, deployable via Kubernetes CRDs, Go or CUE SDKs, or as embedded React components. It is primarily designed for observability use cases, covering all observability pillars. This recommendation document is an outcome of the [SPIKE: Perses](https://github.com/naira-project/naira/issues/19) issue.

Perses was evaluated for its potential as a platform-native, GitOps-friendly embedded dashboard solution for visualizing AI and observability metrics from various sources. The groundwork for this evaluation was established in [RFC 005: Perses as Naira's Embedded Dashboard Foundation](https://github.com/naira-project/architecture/pull/8). This document consolidates the results of that RFC and provides a clear recommendation on when Perses should and should not be used.

Overall, Perses is recommended for use cases where continuous observability data can be retrieved from its officially supported datasources such as Prometheus, Loki, Tempo, and Pyroscope.

## What is Perses?

Perses is a CNCF Sandbox project - accepted in August 2024 - focused on observability visualization. It provides a standardized, open dashboard specification with first-class Kubernetes support via CRDs, a Dashboard-as-Code workflow through Go and CUE SDKs, and a React component library designed for embedding dashboards directly into third-party UIs. It is built with embedding and GitOps workflows as primary design goals rather than as afterthoughts.

## Evaluation Summary

**The following aspects of Perses were researched during the evaluation:**

- **Perses CRDs and their ease of integration via the Perses Operator**: Perses CRDs were expected to be highly customizable, and this was validated through a simple CRD flow covering the Perses server, Perses project, and its components (PersesDatasource and PersesDashboard) integrated into the ecosystem.

- **Dashboard-as-Code via the Go SDK**: Given that most architectural decisions in Naira have revolved around Go as reflected in the [Naira Core API Design](https://github.com/naira-project/architecture/pull/7), the Go SDK was thoroughly evaluated for richness, extensibility, and potential bottlenecks. From the experiments, it was decided that the Go SDK enables type-safe and programmatic dashboard composition.

- **Integration with OpenMFP UI**: Since OpenMFP is Naira's chosen micro-frontend platform, Perses' ability to integrate with it was a critical evaluation criterion which was validated throughout the experimentation.

**The following aspects were excluded from this evaluation:**

- **Dashboard-as-Code via the CUE SDK**: Although CUE provides stricter schema validation and full plugin coverage, it is not widely adopted within our community. It was therefore decided to first evaluate the Go SDK's capabilities before exploring CUE further.

## Strengths

- **CRD integration and datasource flexibility**: Through experimentation with a Custom Resource setup provided by the [Perses Operator](https://perses.dev/perses-operator/docs/user-guide/), the claim of high customizability was validated. Perses also performs 1:1 mappings between Kubernetes namespaces and Perses Projects via the operator, enabling clean multi-tenant isolation without additional configuration overhead. In many cases, React embedding is not even required, and where it is, the integration is straightforward.

- **Observability metric visualization**: Through its panel structure and native datasource integrations, Perses enables extensive customization for visualizing continuous data from sources such as Prometheus. These dashboards can be shipped as independently deployable Kubernetes CRs within Naira, keeping them version-controlled and GitOps-managed alongside the rest of the platform.

- **Dashboard-as-Code with Go SDK**: The Go SDK enables type-safe, programmatic dashboard composition with compile-time checks and reusable components, which aligns naturally with Naira's Go-centric development culture and CI/CD workflows.

## Limitations & Concerns

- **Unsuitability for tabular and graph views**: Perses is purpose-built for observability visualization and its components are designed around that paradigm. While Naira's [Core API Design](https://github.com/naira-project/architecture/pull/7) accommodates both graph and tabular views, tabular representations - such as viewing asset details in a structured list - are not well supported by Perses. As outlined in [RFC 005: Perses as Naira's Embedded Dashboard Foundation](https://github.com/naira-project/architecture/pull/8), Perses must not be treated as a general-purpose visualization tool; its use should be confined to the specific observability use cases described in this document.

- **New tool for contributors**: Perses is a new tool on top of the existing stack, and its strengths, limitations as well as an extensive documentation must be developed for future integration so that contributors would not have a hard time integrating Perses into their workflow. 

## Recommendation

> Perses is recommended for **visualizing observability metrics retrieved from its officially supported datasources** (Prometheus, Loki, Tempo, Pyroscope). It should **not** be adopted as a general-purpose visualization solution for tabular data such as asset details, nor as an alternative to graph views outside the observability domain. Those views require their own dedicated API design and frontend components.

### When to use Perses

- Visualizing continuous observability data from officially supported datasources
- Shipping platform dashboards as version-controlled, GitOps-managed Kubernetes CRDs
- Embedding observability panels natively into Naira's OpenMFP UI

### When not to use Perses

- Rendering tabular asset data (e.g. model registry listings, dataset details)
- General-purpose UI visualization outside the observability domain
- As a replacement for graph or detail views that require Naira's own API design

## Work in Progress

1. **Authentication workflow with Perses**: Perses supports integration with OIDC providers, and its compatibility with Keycloak is currently being explored under the [SPIKE Perses repository](https://github.com/naira-project/spike-perses).

2. **Direct integration with OpenMFP**: Ongoing discussions with the OpenMFP community are underway to explore a direct integration of Perses into the OpenMFP ecosystem, which would simplify its adoption within Naira. This direct integration could mitigate the second limitation provided in "Limitations & Concerns".

## Next Steps

1. **API design for observability metrics**: Existing tabular and graph view API design is enabled through the [Naira Core API Design](https://github.com/naira-project/architecture/pull/7). This design must be tested and implemented against datasources from the LGTM stack as part of the [Reference Integration: Observability Stack](https://github.com/naira-project/naira/issues/14) issue.

2. **Refinement of React embedded dashboards**: React components have been implemented in the [SPIKE Perses repository](https://github.com/naira-project/spike-perses). These require further refinement to fully evaluate the flexibility of Perses' embedded dashboard capabilities for React-based integration.

3. **Refinement of [Reference Integration: Observability Stack](https://github.com/naira-project/naira/issues/14) issue**: Apart from the potential extension of API design to cover observability metrics which could be linked with this issue, it has not been updated with Perses dashboard solution yet. Therefore, a refinement concerning both these aspects is necessary for [Reference Integration: Observability Stack](https://github.com/naira-project/naira/issues/14) issue.

## References

- [Perses official documentation](https://perses.dev)
- [SPIKE Perses repository](https://github.com/naira-project/spike-perses)
- [RFC 005: Perses as Naira's Embedded Dashboard Foundation](https://github.com/naira-project/architecture/pull/8)
- [Naira Core API Design](https://github.com/naira-project/architecture/pull/7)
- [Reference Integration: Observability Stack](https://github.com/naira-project/naira/issues/14) 