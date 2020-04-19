---
title: Releases
bigheader: Kubernetes release versions
content_template: templates/concept
class: releases
mermaid: true
---

{{< announcement >}}

{{< deprecationwarning >}}

Demo starter page covering releases...

#### Active Kubernetes releases

Kubernetes follows an N-2 support policy (meaning that the 3 most recent minor versions receive security and bug fixes). This generally results in a particular minor version being supported for ~9 months; as illustrated by the chart below 

{{< mermaid >}}
gantt
    title            Past 5 Releases
    dateFormat       YYYY-MM-DD
    %% 36w = 9 months
    section 1.18.x
    1.18.x           :2020-03-25, 36w
    section 1.17.x
    1.17.x           :2019-12-09, 36w
    section 1.16.x
    1.16.x           :2019-10-22, 36w
    section 1.15.x
    1.15.x           :done, 2019-06-19, 36w
    section 1.14.x
    1.14.x           :done, 2019-03-25, 36w
{{</ mermaid >}}

| Version | Release Date | Notes |
|---------|--------------|-------|
| 1.0     | 2019-11-24   | link  |
| 1.1     | 2018-10-22   | link  |
| 1.2     | 2017-11-12   | link  |

#### todo: include patches or maybe have a separate table for latest?

| Latest Version | Released   | Release Notes | Projected EOL |
|----------------|------------|---------------|---------------|
| 1.16.3         | 2019-11-24 | link          | Dec, 2021     |
| 1.17.4         | 2018-10-22 | link          | Dec, 2021     |
| 1.18.5         | 2017-11-12 | link          | Dec, 2021     |

#### Future

| Version | Released   | Release Notes | Projected EOL |
|---------|------------|---------------|---------------|
| 1.19.0  | 2019-11-24 | link          | Dec, 2021     |

Release notes or something...

#### Release lifecycle

A breif overview, what is alpha / beta / rc.

- helpful link 1
- helpful link 2
- Version skew: https://kubernetes.io/docs/setup/release/version-skew-policy/

#### Check version

```
kuebctl version
```