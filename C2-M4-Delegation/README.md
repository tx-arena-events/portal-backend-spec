# C2-M4 Delegation

**Version:** dropped, as of 2026-10-28
**Capability:** C2 Identity and Access
**Kind:** dropped with OSP 2.0

With OSP 2.0, an OSP and its portal are one. A PSP no longer calls on behalf of a company, so there is no delegation.

**Where the tasks went:** The company reference of the permissions (FRAMEWORK 4.3) takes the place of delegation: permissions with scope `company` apply only to the person's company, permissions with scope `osp` to OSP staff. A change of provider becomes a transfer to another OSP (FRAMEWORK 5.6).
