---
status: accepted
date: 2026-05-06
written-by: Mateusz Czapliński
---

# ADR-001 - Imperfect, heuristic detection of App-Model relations by scanning Deployments in k8s

## Context and Problem Statement

<!-- Describe the context and problem statement, e.g., in free form using two to three sentences
or in the form of an illustrative story. You may want to articulate the problem in form of a question.
Consider adding links to collaboration boards or issue management systems.
Make the scope of the decision explicit, for instance,
by calling out or pointing at structural architecture elements (components, connectors, ...). -->

Based on internal team discussions with colleagues experienced in deploying k8s at Customers sites,
and in consulting for Customers on their challenges with those, it sounds that it's a common problem
for Customers to not understand very well what systems/apps they are running on their clusters, and how those are related.
This matches my (MC's) personal experience at a previous employer,
where it was clear that middle management considered it risky for their careers to disable/sunset older features or systems,
due to not knowing if any important company-internal app is relying on those.
This led to a noticeable waste of effort and resources on trying to keep legacy systems up and running
for no known practical reason, hindered by lack of understanding whether they are even functioning properly at all.

From a technical point of view, this seems to match cursory research on the Internet.
There seem to be many open-source apps doing trivial detection of basic, builtin, explicit
"Kubernetes hierarchy/tree of resources", such as Deployment<->Pods.
There are also apps doing similarly trivial detection of other intrinsically managed relations for some specific CRDs
(for example, Crossview specifically focuses on showing Crossplane CRDs and their builtin, explicit relationships hierarchies).
However, I didn't find any open-source apps that would detect "client-server" (caller-callee) relationships
between Deployments and Services on k8s.
Note: in context of AI, many different types of assets, such as AI gateways (e.g. LiteLLM) or AI runtimes (e.g. vLLM, ollama),
are technically all only visible as generic Services in k8s.

In this context, it seems a reasonable educated bet, that implementing even a basic, simple
autodetection mechanism between any k8s Deployments and some Services
(focusing on a few known AI-related Services, like LiteLLM, MLFlow, vLLM, etc.),
has a good chance of providing real, useful, and attractive value to Customers.
It can help them improve their understanding of the systems they're operating and managing, and visibility into them.

In Naira as an AIDP, we currently expect to implement both discovery (“reading”), and mutation (“writing”), of cluster state.
Discovery of information from an existing cluster is something we consider as useful to implement first, before mutation.
Discovery leading to visibility into existing state would help Customer make an informed decision what to then mutate,
and after mutation visibility would again help them understand what changed.
Visibility can be also valuable to Customers on its own, even without mutation.

## Considered Options

1. **CHOSEN:** Implementing basic scanner of k8s Deployments (Envs, Secrets)
   - Good, because simple and cheap to implement, easy to understand.
   - Good, because arguably the pattern of keeping hostnames and API keys in Env variables is very common and the bet is that following it should address the "Pareto principle" of covering 80% cases at very low cost.
   - Bad, because will not detect all hostnames (e.g. hardcoded; in config files; in Service Mesh setups) - "false negatives".
   - Bad, because will accidentally detect nonexistent relationships in case when hostnames match common words (e.g. "portal") - "false positives".
     - Can be later mitigated by adding extra custom filtering rules.
   - Bad, because may require extensive API permissions to read Secrets.
     - Can be mitigated by selectively making sub-scanners opt-in for fine-grained choice.
     - Can be mitigated by only running it manually at explicitly controlled times.

2. Crossview ([https://github.com/crossplane-contrib/crossview](https://github.com/crossplane-contrib/crossview))
   - Good, because existing, open-source app.
   - Good, because supports autodetection of basic Crossplane CRDs relationships.
   - Bad, because only handles builtin k8s resources and Crossplane CRDs.
   - Bad, because isn't aware of AI assets.
   - Bad, because only handles trivial, intrinsic relationships; doesn't autodetect Deployment↔Service relationships.

3. Open Resource Discovery (ORD; [https://open-resource-discovery.org](https://open-resource-discovery.org/))
   - Good, because open-source protocol specification and standard.
   - Good, because SAP authorship and partially deployed at SAP (see: [video](https://www.youtube.com/watch?v=IwyaX_XoSuo)).
   - Good, because specification appears extendable (allows adding custom data/fields).
   - Bad, because deployment at SAP is not complete (see: [video](https://www.youtube.com/watch?v=IwyaX_XoSuo)).
   - Bad, because no known users other than SAP.
   - Bad, because requires development effort at each service/app at a Customer, in order to expose ORD manifests.
   - Bad, because no ready-made open-source "ORD Aggregators" tools available.
   - Bad, because covers only Services (server part), does not cover users (client part) - does not help in detecting relationships (see the first Q&A in the [video](https://www.youtube.com/watch?v=IwyaX_XoSuo)).

4. Cartography ([https://cartography-cncf.github.io/cartography](https://cartography-cncf.github.io/cartography))
   - Good, because open-source, existing app.
   - Good, because CNCF.
   - Good, because comprehensive set of plugins for scanning various entities and storing them in a Neo4j graph database.
   - Bad, because doesn't appear to detect Deployment↔Service or Deployment↔AI-Asset relationships in k8s (see: [schema docs](https://cartography-cncf.github.io/cartography/modules/kubernetes/schema.html)).
   - *NOTE: may be worth further investigation, maybe some parts of it can prove useful to us.*

5. Network traffic scanning and monitoring to detect relationships — "packet sniffer"
   - Good, because detection of real traffic.
   - Bad, because complex to implement; may be worth revisiting in future.
   - Bad, because not guaranteed to detect rarely used connections.


## Decision Outcome

We'll implement a basic scanner of k8s resources of Deployment kind, using a simple, imperfect heuristic. The scanner will go over Env fields (including Secrets, and ConfigSets), searching for:

- known DNS hostnames, especially of known AI-related assets and resources
  (such as LiteLLM, vLLM, etc. Services in the same or sibling k8s clusters) -
  those will be treated as a "client-server" relationship between Deployment and AI Services;
- API keys matching known formats (such as LiteLLM) -
  those will help discover relationships between Deployments and specific AI models.

The scanner will save discovered entities and relationships (as described above) into Naira's graph database.

## Related Links

- https://open-resource-discovery.org
- https://www.youtube.com/watch?v=IwyaX_XoSuo
- https://cartography-cncf.github.io/cartography

## Notes & Review Comments

- Future directions:
   - support for KCP
   - not every resource may be on k8s - e.g. may be on Google Cloud, AWS cloud, etc.
   - deep dive on Cartography and explore how/if it could be useful to us (see: https://github.com/naira-project/product/issues/23)
   - research also scanning k8s Network Policies, HTTPRoutes, and other routing-related resources
   - consider researching & adding support for scanning information from ORD manifests, Service Mesh setups, and other sources

