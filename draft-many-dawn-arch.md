---
title: "Architecture for Discovery of Agents With Names"
abbrev: "DAWN Architecture"
docname: draft-many-dawn-arch-latest
category: info
date:
consensus:
ipr: trust200902
area: "Internet"
workgroup: Discovery of Agents With Names
submissiontype: IETF
keyword: Internet-Draft
pi: [toc, sortrefs, symrefs]

author:

 -
    name: Kehan Yao
    organization: China Mobile
    email: yaokehan@chinamobile.com

normative:
  I-D.farrel-dawn-terminology:


informative:


--- abstract

This document defines a reference architecture and deployment models for the Discovery of Agents With Names (DAWN). While DAWN's primary scope is strictly limited to name-based discovery lookups, it operates within a broader ecosystem. This document decomposes the ecosystem into three planes (Publication, Discovery, and Interaction) and defines an architecture consisting of the Registrar, Organization Catalogue, AI Resource Catalogue, AI Resource Registry, Discovering Entity, and Discovered Entity. It maps these components across three deployment models: local networks, single-domain networks, and between trusted collaborating domains. The architecture aligns with external lifecycle frameworks without redefining entity provisioning or trust models.

--- middle

# Introduction {#intro}

The complete discovery ecosystem encompasses several functions: registration and management, search based on formalized criteria, discovery lookup based on a minimum viable dataset, reading of additional information, and capability exchange. From an implementation perspective, some of these functional components might be included within the same module, and some elements might be achieved using the same protocols.

However, the scope of DAWN covers only the name-based "discovery lookup". A Discovering Entity provides an identifier (e.g., a name), and the system returns the network location and Minimum Discoverable Information (MDI) of the Discovered Entity. To clearly illustrate where DAWN sits within the broader ecosystem, this document provides a reference architecture that decomposes these functions into three planes and distinct roles.

The overall entity lifecycle including provisioning, description, publication, operation, withdrawal, and revocation, is defined elsewhere. This architecture does not define a new lifecycle. Instead, it constrains the behavior of DAWN components at the lifecycle hooks.

# Terminology {#terminology}

This document uses the following terms as defined in {{I-D.farrel-dawn-terminology}}: Discovering Entity, Discovered Entity, Registration, Registrar, Minimum Discoverable Information (MDI), and Selection.

This document additionally defines the following architectural component terms:

**Organization Catalogue**: A component that resolves an organization or well-known label into the location (endpoint URL) of the organization's specific AI Resource Catalogue. It acts as the entry point for discovery.

**AI Resource Catalogue**: A component that performs search based on function or capability and returns a list of candidate names. It holds names, functions, and capability tags, but does not hold network locations or full internal records. Search results from this Catalogue are typically filtered through Selection before entering the Registry.

**AI Resource Registry**: A component that performs discovery lookup based on an exact name and returns the location and MDI. It holds only the name, location, freshness, and minimal dataset, and does not carry full capability descriptions.

**Cache**: Local storage on the Discovering Entity side used to save recently obtained discovery results or Catalogue/Registry locations to reduce repeated queries and discovery latency. Entries in the Cache are constrained by freshness and should be re-queried after expiration.

**Host**: A physical or virtual computing machine that provides the execution environment for DAWN components. A Host may run one or more Discovering Entities, Discovered Entities, or infrastructure components.

**Domain**: An independent administrative and trust boundary governed by a unified security policy.

# Architecture and Components {#arch}

Since DAWN's scope relies on interactions with the broader discovery ecosystem, this section provides a system architecture for the discovery ecosystem.

## Three Planes and Interaction Boundaries {#planes}

The discovery ecosystem is logically decomposed into three distinct planes. This layered design decouples "intent expression", "location resolution", and "actual communication".

### Publication Plane {#publication}

*   **Responsibilities**: Handles entity registration, information storage, policy filtering, and export. It is the source of data and maintains the full internal records of the entity.

*   **Boundary**: It provides only trimmed metadata to the Discovery Plane. The Publication Plane has strict lifecycle hooks. When external lifecycle managers trigger entity retirement or revocation, the Publication Plane must execute the Withdraw action to ensure information synchronization on the Discovery Plane.

### Discovery Plane {#discovery-plane}

*   **Responsibilities**: Handles query requests from Discovering Entities. It consists of the Organization Catalogue, AI Resource Catalogue, AI Resource Registry, and the Cache/Selection logic on the Discovering Entity side.

*   **Boundary**: This is where core DAWN's protocol work resides. It follows strict privacy boundaries: queries cannot expose too much of the Discovering Entity's intent, nor can information beyond the MDI be pulled from the Registry.

### Interaction Plane {#interaction-plane}

*   **Responsibilities**: Establishes actual network communication after discovery is complete.

*   **Boundary**: Completely out of DAWN's scope. This plane includes Bootstrap (authenticate, authorize, negotiate capabilities) and Communicate over agent communication protocols. The `{location, MDI}` provided by the Discovery Plane is merely an input to the Bootstrap step, not the decision to establish a connection itself.

{{fig-arch}} shows the architecture of the discovery ecosystem, DAWN sits within this architecture.

~~~
+++++++++++++++++++++++++++++++++++++  ++++++++++++++++++++++++++++++++++
+  Discovery Plane                  +  + Publication Plane             +
+                                   +  +                               +
+ +-------------+   +------------+  +  +               +-------------+ +
+ | Discovering +-->|Organization|  +  +               | Discovered  | +
+ |   Entity    <---+  Catalogue |<------------+       |   Entity    | +
+ |             |   +------------+  +  +       |       |             | +
+ |             |                   +  +       |       |             | +
+ |   *--*--*   |   +------------+  +  +  +----+----+  |             | +
+ |   |cache|   +-->| AI Resource|<-------+Registrar|<-+             | +
+ |   *--*--*   <---+  Catalogue |  +  +  +----+----+  |             | +
+ |             |   +------------+  +  +       |       |             | +
+ | *----*----* |                   +  +       |       |             | +
+ | |selection| |   +------------+  +  +       |       |             | +
+ | *----*----* +-->| AI Resource|<------------+       |             | +
+ |             <---+  Registry  |  +  +               |             | +
+ |             |   +------------+  +  +               |             | +
+ +------+------+                   +  +               +-------+-----+ +
+++++++++|+++++++++++++++++++++++++++  ++++++++++++++++++++++++|++++++++
+++++++++|+++++++++++++++++++++++++++++++++++++++++++++++++++++|++++++++
+        |  Interaction Plane                                  |       +
+        |                       Bootstrap                     |       +
+        +---------------------------------------------------->|       +
+        |<----------------------------------------------------+       +
+        |                   Agent Communication               |       +
+        +<--------------------------------------------------->|       +
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

~~~
{: #fig-arch title="DAWN Architecture"}

## Components and Roles {#components}

*   **Discovered Entity**: The named entity that initiates the registration action. It provides its full record to the Registrar and triggers the withdrawal action at the end of its lifecycle.

*   **Registrar**: The hub and policy enforcement point of the Publication Plane. It holds the entity's full record. Based on predefined export policies, it granularly filters and pushes metadata to the appropriate discovery components.

*   **Organization Catalogue**: The first resolution point on the Discovery Plane. It resolves an organization identifier or well-known label into the specific endpoint URL of the target **AI Resource Catalogue**. It acts as a routing guide, which is particularly essential in cross-domain deployments.

*   **AI Resource Catalogue**: The "intent matcher" on the Discovery Plane. It receives capability or function-based search queries and returns a list of matching candidate names. It holds semantic metadata but deliberately does not hold network locations or full internal records. It acts as a buffer between fuzzy search and exact resolution.

*   **AI Resource Registry**: The "authoritative resolver" on the Discovery Plane. It performs exact name lookups and returns the network reachability (`{location}`) and Minimum Discoverable Information (`{MDI}`). It is the direct interaction target for the core DAWN discovery lookup protocol.

*   **Discovering Entity**: The entity that initiates the query. To optimize performance and reduce network overhead, it has a built-in **Cache** to store recently resolved locations and a built-in **Selection** algorithm to filter the candidate list returned by the AI Resource Catalogue into the final names to be resolved by the Registry.

# Overall Workflow {#workflow}

While the specific sequence of interactions varies across deployment topologies (see Section {{deployment}}), the logical lifecycle of an entity within the discovery ecosystem follows a common pattern divided into four abstract states:

**1. Publication and Readiness**: The lifecycle begins when a Discovered Entity registers with a Registrar. The Registrar holds the full record and applies export policies. It then distributes the sanitized metadata to the discovery infrastructure: capability descriptions are pushed to the AI Resource Catalogues, and reachability data `{location, MDI}` is pushed to the AI Resource Registry. Once published, the entity transitions to a "discoverable" state.

**2. Decoupled Discovery Resolution**: When a Discovering Entity seeks a target, it transitions through a three-step logical resolution process. First, when necessary (e.g., in cross-domain deployments), it queries the **Organization Catalogue** to obtain the exact endpoint URL of the target **AI Resource Catalogue**. Next, it performs an **Intent Search** by sending a capability or function query to the **AI Resource Catalogue** to identify potential candidates, applying its local **Selection** logic to pick the desired target name. Finally, it performs a **Location Lookup** by querying the **AI Resource Registry** with the selected name to retrieve the authoritative `{location, MDI}`. These results are cached locally based on freshness rules to optimize subsequent queries.

**3. Handoff to Interaction**: DAWN's role ends once the `{location, MDI}` is retrieved and validated against the cache. The Discovering Entity passes this information as input to the Interaction Plane. The subsequent bootstrap and communication are executed point-to-point between the two entities and are outside the scope of DAWN.

**4. Withdrawal and Invalidation**: The lifecycle concludes when the entity is retired or revoked. The Registrar executes a Withdraw operation, removing or invalidating the entity's records in the Catalogues and Registry. The Discovering Entity's local Cache naturally expires based on the freshness indicators, ensuring that subsequent resolutions no longer return the retired entity.

# Deployment Models {#deployment}

The DAWN architecture is designed to operate across different trust and management scopes. This section maps the architecture to three operational scopes: Local network, single-domain network, and between trusted collaborating organizations.

## Local Network Deployment {#local-deployment}

**Scenario**: A single link or local area network (e.g., smart home, single-machine multi-agent environment, branch office) with pre-established physical or Layer 2 trust.

### Architectural Mapping {#local-mapping}

As shown in {{fig-local}}, to accommodate the constrained and zero-configuration nature of local networks, the logical architectural components are instantiated in a distributed and lightweight manner.

- **Discovered Entity**: The local AI resource running on a host (Host B).

- **Registrar**: Rather than a centralized server, the Registrar operates as a local discovery service/daemon co-located on the discovered entity's host. It handles the publication and withdrawal of the discovered entity's service instance.

- **AI Resource Catalogue**: In a local environment, the Catalogue function is realized through a local broadcast search. The Discovering Entity queries the network to enumerate available entity capabilities or service types.

- **AI Resource Registry**: The Registry function is realized through a targeted local lookup. Once a candidate is selected, the discovering entity queries the local network to resolve the specific instance into a concrete network 'location' and 'MDI'.

- **Discovering Entity**: Runs on a client host (Host A), utilizing its local **Cache** and **Selection** logic to minimize network overhead.

~~~
+-----------------------------------------------------------------------------------+
| LOCAL NETWORK LINK                                                                |
|                                                                                   |
| +-----------------------+                           +---------------------------+ |
| | Host A                |                           | Host B                    | |
| | +-------------------+ |                           | +-----------------------+ | |
| | | Discovering Entity| |                           | | Discovered Entity     | | |
| | | [ Cache ]         | |                           | +-----------+-----------+ | |
| | | [ Selection ]     | |                           |             |             | |
| | +---------+---------+ |                           |             v             | |
| |           |           |                           | +-----------------------+ | |
| |           |           |                           | | Registrar             | | |
| |           |           |                           | | (Local Responder)     | | |
| |           |           |                           | +-----------+-----------+ | |
| +-----------------------+                           +-------------|-------------+ |
|             |                  +-----------------+                |               |
|             +----------------->|                 |<---------------+               |
|             |  Step 1 Search   |   Local Link    |                |               |
|             |<-----------------|                 |                |               |
|             |----------------->|                 |                |               |
|             |  Step 2 Lookup   |                 |                |               |
|             |<-----------------|                 |                |               |
|             |                  +-----------------+                |               |
|             |                                                     |               |
|             |            Step 3: Bootstrap & Communication        |               |
|             +<===================================================>+               |
+-----------------------------------------------------------------------------------+
~~~
{: #fig-local title="Local Network Deployment Architecture"}

### Workflow Description {#local-workflow}

Prerequisite: The Registrar on Host B publishes its presence and basic service metadata to the Local Link, making the Discovered Entity discoverable.

**Step 1:** The Discovering Entity on Host A sends a broadcast/multicast query to the Local Link, searching for available entities by capability or generic service type.

**Step 2:** After the Discovering Entity selects a candidate via its selection logic and checks its local cache, it sends a targeted query to the Local Link for the specific instance. The Registrar on Host B responds with the precise `{Location, MDI}`.

**Step 3:** Using the obtained `{Location, MDI}`, the Discovering Entity bypasses the Local Link broadcast and establishes a direct unicast connection (Host A to Host B) to the Discovered Entity to perform the protocol-specific bootstrap and communication.

### Key Characteristics {#local-char}

- **Distributed Architecture**: The Catalogue and Registry functions are realized through local network queries rather than centralized infrastructure.

- **Component Co-location**: The Registrar operates as a local daemon on the Discovered Entity's host, eliminating the need for central servers.

- **Cache Efficiency**: The Discovering Entity relies heavily on its local cache to minimize broadcast traffic and network overhead.

- **Dynamic Lifecycle**: When a Discovered Entity goes offline, its Registrar withdraws the advertisement, immediately invalidating cached entries on other hosts.

## Single Domain Deployment {#single-domain}

Scenario: Inside a single organization or administrative domain (e.g., enterprise intranet, single-tenant VPC in the cloud), requiring discovery across subnets or departmental boundaries under a unified organizational security policy.

### Architectural Mapping {#single-mapping}

Unlike the distributed nature of local networks, a single domain deployment utilizes centralized or organization-level infrastructure to manage discovery across a wider network scope.

- **Discovered Entity** and **Discovering Entity** remain the same mapping rules as defined in {{local-mapping}}.

- **Registrar**: Deployed as a **centralized organizational service** (e.g., an enterprise control plane or API gateway). It acts as a policy enforcement point (PEP), receiving registrations from agents, holding their full records, and applying strict export policies before publishing data to the catalogues and registry.

- **AI Resource Catalogue**: Deployed as an independent centralized directory service within the domain. It allows entities to search by capability or function across the entire organization, returning a list of candidate names.

- **AI Resource Registry**: Deployed as a centralized location resolver. It holds only the authoritative network locations and MDIs for the entities within the domain.

{{fig-single-domain}} shows the architectural deployment in single domain boundary.

~~~
+-------------------------------------------------------------------------------+
| SINGLE DOMAIN BOUNDARY (e.g., Enterprise Intranet)                            |
|                                                                               |
| +-----------------------+                       +---------------------------+ |
| | Host A                |                       | Host B                    | |
| | +-------------------+ |                       | +-----------------------+ | |
| | | Discovering Entity| |                       | | Discovered Entity     | | |
| | | [ Cache ]         | |                       | |                       | | |
| | | [ Selection ]     | |                       | +-----------+----------++ | |
| | +---------+---------+ |                       +-------------|----------|--+ |
| +-----------|-----------+                                     |          |    |
|             |           +-------------------------------------|--------+ |    |
|             |           | Centralized Organizational Services |        | |    |
|             |           |                                     v        | |    |
|             |           |   +---------------------------------------+  | |    |
|             |           |   |                 Registrar             |  | |    |
|             |           |   +-------------------+-------------------+  | |    |
|             |           |                       |                      | |    |
|             |           |         +-------------+------------+         | |    |
|             |           |         |                          |         | |    |
|             |           |         v                          v         | |    |
|             |  Step 1:  | +---------------+          +---------------+ | |    |
|             |  Search   | | AI Resource   |          | AI Resource   | | |    |
|             +------------>| Catalogue     |          | Registry      | | |    |
|             |           | +---------------+          +---------------+ | |    |
|             |           |                                    ^         | |    |
|             |           +------------------------------------|---------+ |    |
|             |          Step 2: Lookup                        |           |    |
|             +----------------------------------------------->|           |    |
|             |                                                            |    |
|             |              Step 3: Bootstrap & Communication             |    |
|             +<==========================================================>+    |
|                                                                               |
+-------------------------------------------------------------------------------+
~~~
{: #fig-single-domain title="Single Domain Deployment Architecture"}


### Workflow Description {#single-workflow}

Prerequisite: The Discovered Entity on Host B registers with the centralized Registrar. The Registrar holds the full record and applies organizational export policies, pushing filtered capability metadata to the AI Resource Catalogue and location/MDI data to the AI Resource Registry.

**Step 1:** The Discovering Entity on Host A queries the AI Resource Catalogue (e.g., searching by function or capability). The Catalogue returns a list of candidate names, and Host A uses its local selection logic to choose a target.

**Step 2:** The Discovering Entity queries the centralized AI Resource Registry for the chosen name. The Registry resolves the name and returns the precise `{Location, MDI}` to Host A.

**Step 3:** Using the retrieved `{Location, MDI}`, the Discovering Entity on Host A establishes a direct unicast connection to the Discovered Entity on Host B to perform protocol-specific bootstrapping, authentication, and subsequent agent communication.

### Key Characteristics {#single-char}

- **Centralized Policy Control**: The Registrar acts as a strict gatekeeper, ensuring sensitive internal details are filtered based on organizational export policies before reaching the discovery infrastructure.

- **Service Decoupling**: The Catalogue (for semantic search) and Registry (for exact location resolution) are deployed as independent centralized services, each optimized for its specific query type.

- **Cross-Subnet Resolution**: The architecture seamlessly spans multiple network segments, relying on internal routing and centralized resolution rather than local broadcasts.

## Trusted Collaborating Domains Deployment {#cross-domain}

Scenario: Multiple independent administrative domains (e.g., Enterprise A, Enterprise B) establish a federation based on pre-established trust relationship, allowing cross-domain discovery of entities.

### Architectural Mapping {#cross-mapping}

- **Discovered Entity** and **Discovering Entity** remain the same mapping rules as defined in {{local-mapping}}. The only difference is that **Discovered Entity** is deployed in Domain B and **Discovering Entity** is deployed in Domain A.

- **Registrar in Domain B**: Implements strict cross-domain export policies. It receives registrations and exports authorized public metadata to both the AI Resource Catalogue and the AI Resource Registry in Domain B.

- **Organization Catalogue in Domain A**: The local entry point for Domain A. It resolves organizational identities and returns the endpoint URL of the target Domain's (Domain B) public AI Resource Catalogue.

- **AI Resource Catalogue Domain B**: A centralized service containing public-facing capability metadata. It returns candidate names based on semantic search.

- **AI Resource Registry Domain B**: A centralized service containing the authoritative `{Location, MDI}` for federated entities.

{{fig-cross-domain}} shows the architectural deployment in trusted collaborating organizations.

~~~
+-----------------------------------------+     +-----------------------------------------+
| DOMAIN A                                |     | DOMAIN B                                |
|                                         |     |                                         |
| +-----------------------+               |     |               +-----------------------+ |
| | Host A                |               |     |               | Host B                | |
| | +-------------------+ |               |     |               | +-------------------+ | |
| | | Discovering Entity| |               |     |               | | Discovered Entity | | |
| | | [ Cache ]         | |               |     |               | +---------+---------+ | |
| | | [ Selection ]     | |               |     |               |           |           | |
| | +---+-----+---------+ |               |     |               +-----------|--------+--+ |
| +-----+-----|-----------+               |     |                           v        |    |
|       |     |                           |     |                 +--------------+   |    |
|       |     |         Step 1            |     |                 |              |   |    |
|       |     |   Resolve Domain B's AI   |     |                 | Registrar B  |   |    |
|       |     |Resource Catalogue Endpoint|     |                 |              |   |    |
|       |     v                           |     |                 +---------+----+   |    |
|       |   +-------------------+         |     |                           |        |    |
|       |   | Organization      |         |     |                           |        |    |
|       |   | Catalogue    A    |         |     |                           |        |    |
|       |   +-------------------+         |     | +-------------------+     |        |    |
|       |  Step 2 Search                  |     | | AI Resource       |<----+        |    |
|       +---------------------------------------->| Catalogue B       |     |        |    |
|       v                                 |     | +-------------------+     |        |    |
|       |                                 |     |                           |        |    |
|       |  Step 3 Lookup                  |     | +-------------------+     |        |    |
|       +---------------------------------------->| AI Resource       |<----+        |    |
|       |                                 |     | | Registry B        |              |    |
|       |                                 |     | +-------------------+              |    |
|       | Step 4 Bootstrap & Communication|     |                                    |    |
|       +--------------------------------------------------------------------------->|    |
+-----------------------------------------+     +-----------------------------------------+
~~~
{: #fig-cross-domain title="Trusted Collaborating Domains Deployment Architecture"}


### Workflow Description {#cross-workflow}

Prerequisite: The **Discovered Entity** on Host B registers with **Registrar B**. The Registrar applies cross-domain export policies and exports the authorized metadata to Domain B's **AI Resource Catalogue** and **AI Resource Registry**.

**Step 1: Organization Catalogue Lookup**: The Discovering Entity on Host A queries its local Organization Catalogue in domain A to resolve the target domain. It receives the endpoint URL for Domain B's AI Resource Catalogue.

**Step 2: Catalogue Search**: Using the obtained endpoint, the Discovering Entity crosses the trust boundary to query Domain B's AI Resource Catalogue by capability or function. The Catalogue returns candidate names, and Host A's Selection logic chooses a target.

**Step 3: Registry Lookup**: The Discovering Entity queries Domain B's AI Resource Registry for the selected name. The Registry returns the public `{Location, MDI}`.

**Step 4: Bootstrap & Communication**: Using the retrieved `{Location, MDI}`, the Discovering Entity on Host A establishes a direct unicast connection across the public internet to the Discovered Entity on Host B for secure communication.

### Key Characteristics {#cross-char}

- **Asymmetric Exposure**: Domain B only exposes its Catalogue and Registry to authorized cross-domain queries. Internal topology and non-public entities remain entirely hidden.

# Operational Considerations {#ops-considerations}

Information Granularity and Consistency: The separation of the Catalogue and Registry reduces the risk of privacy leakage, but it introduces the possibility of information inconsistency. The Registrar must ensure synchronization between the capability tags in the Catalogue and the MDI in the Registry. If the Catalogue finds an entity but it has been withdrawn from the Registry, it results in a dangling query.

Cache Management: The Cache on the Discovering Entity side cannot be indefinite. It must adhere to the TTL or freshness indications returned by the Registry to prevent directing interactions to locations that have changed or been maliciously hijacked.

# Security Considerations {#sec-considerations}

*   **Integrity of Discovery Metadata**: The `{location, MDI}` returned by the Registry must be integrity-protected to prevent attackers from redirecting a Discovering Entity to a malicious endpoint.

*   **Privacy via Decoupling**: Separating the AI Resource Catalogue from the Registry prevents the Registry from learning the semantic intent behind queries. In federated environments, this prevents a peer domain from profiling internal topology.

*   **Lifecycle Security**: Withdrawal requests must be strictly authenticated to prevent Denial of Service or unauthorized persistence. The Registrar acts as the trust anchor for these hooks.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgments
{: numbered="false"}