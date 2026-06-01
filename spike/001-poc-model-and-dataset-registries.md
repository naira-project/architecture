---
status: in review
creation date: 2026-06-01
contributors: "@libera13, @Daviidg, @kocakkaan"
---

# **Overview**
This SPIKE document summarizes the work carried out for the issue SPIKE: Research and PoC Model Registries. The primary goal of this SPIKE was to research available model registries suitable for Naira's first PoC. Additionally, other AI assets - such as datasets, agents, and skills - were explored. Based on this research, a PoC was implemented that mirrors Naira's capabilities with respect to AI asset unification.

# **Outcome**

1. **AI asset diversification**: The first PoC iteration focused on two distinct AI asset types: models and datasets. In addition to these, an "application" type was introduced to represent the application as the consumer of models, and models as the consumer of the datasets they are trained on.

2. **Model registries**: MLFlow and LiteLLM were selected as the model registry solutions for the PoC. Since Naira is designed to function as an overarching AI asset unification platform, it must remain agnostic to whichever model registry or provider a given company uses. This flexibility is enabled through the [Naira Core API Design](https://github.com/naira-project/architecture/pull/7). In this PoC, MLFlow is used to pull models registered internally within the company's own infrastructure, while LiteLLM acts as a unified API gateway for externally hosted models from third-party providers such as OpenAI or Anthropic.

3. **Dataset registries**: Datahub and OpenMetadata were selected as dataset registries for the PoC. As Naira's flexibility claim extends to all AI asset types, these registries should be treated as PoC-specific choices rather than final decisions. Due to performance and resource concerns around Datahub, OpenMetadata is preferred for the initial iterations of Naira, before moving toward a fully flexible, provider-agnostic platform for AI asset unification.

4. **PoC**: The PoC focuses on pulling distinct AI assets via "plugins" implemented within the PoC source code, which can also be understood as API calls for asset aggregation. The PoC covers the following technologies: MLFlow and LiteLLM APIs for model registries; Datahub and OpenMetadata APIs for dataset registries; Go for the backend and React for the frontend; and OpenMFP as the micro frontend platform.

# **Work in Progress**

1. **Integration of PoC backend into the Naira repository**: The PoC backend structure has been aligned with the decisions made during [Naira Core API Design](https://github.com/naira-project/architecture/pull/7), which forms the foundation of Naira's flexible architecture. Once the final changes are confirmed, the PoC will be merged into the Naira repository.

2. **UI/UX decisions from the PoC**: Given Naira's broad functionality, a clean user interface with a strong end-user focus is essential. To that end, a [Minimal UI Features RFC](https://github.com/naira-project/architecture/pull/6) has been proposed and is currently under discussion. 

# **Next Steps**

1. **UI/UX for the Naira repository**: While the [Minimal UI Features RFC](https://github.com/naira-project/architecture/pull/6) is still being discussed, the immediate priority is to integrate the PoC backend structure into Naira. Once the RFC matures and is elaborated in one or more ADRs, specific UI features can be implemented in Naira or migrated from the PoC.

2. **Authentication and authorization workflow**: The PoC does not cover Naira's authentication and authorization workflow, which will be provided by Keycloak and OpenFGA. A dedicated [OpenFGA integration](https://github.com/naira-project/naira/issues/20) issue has been created to deliver a minimal OpenFGA setup demonstrating a basic authorization workflow, to be followed by a clean authentication flow integrating Keycloak user retrieval. A potential RFC on this topic is also being considered, building on the groundwork already laid by @rootlsz in his workshops. Beyond Naira's own auth workflow, a further consideration for future iterations is how Naira should integrate with the authentication and authorization systems of the plugins themselves, given that each plugin may interface with an external service that has its own auth requirements.