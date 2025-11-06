# Get-Resource-Groups-Created-


---
title: DevOps Consultation Swimlane
---

flowchart TB
    %% Define lanes
    subgraph Client [Client]
        C1[Share business goals & challenges]
        C2[Describe current tools & processes]
        C3[Explain infra & deployment methods]
        C4[Discuss compliance & security needs]
        C5[Share monitoring & incident practices]
        C6[Explain collaboration & workflows]
        C7[Highlight pain points & blockers]
        C8[Discuss future growth & scaling]
        C9[Confirm next steps]
    end

    subgraph Consultant [Consultant]
        S1[Ask about business context & KPIs]
        S2[Assess CI/CD, cloud & tooling maturity]
        S3[Evaluate infra, IaC, container/serverless strategy]
        S4[Probe into compliance, RBAC & secret management]
        S5[Review monitoring, logging, observability maturity]
        S6[Assess DevOps culture & process alignment]
        S7[Prioritize quick wins vs. long-term roadmap]
        S8[Align DevOps roadmap with scaling & innovation]
        S9[Summarize findings & propose next actions]
    end

    %% Flow connections
    C1 --> S1
    C2 --> S2
    C3 --> S3
    C4 --> S4
    C5 --> S5
    C6 --> S6
    C7 --> S7
    C8 --> S8
    C9 --> S9
