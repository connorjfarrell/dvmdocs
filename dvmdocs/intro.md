# Welcome to DVMProject

This is the main website for all documentation related to DVMProject tools and applications.

The DVMProject organization maintains an open-source collection of applications and hardware
designs for creating and interacting with P25, DMR, and NXDN digital radio networks. The
project spans both hardware (modems, hotspots, and V.24 adapters) and software (the `dvmhost`
application suite, firmware, consoles, and supporting tools), which together provide a
full-featured range of digital radio connectivity — from a single conventional hotspot to
multi-site trunked networks linked through fixed network equipment (FNE).

```{warning}
DVMProject software and hardware are provided for personal, non-commercial, hobbyist use only.
**They must never be used in public safety or life safety critical applications.** See the
[Introduction](overview/introduction.md) for the full usage policy.
```

```{note}
**About this documentation.** These pages were drafted with AI assistance, summarizing the
READMEs, example configuration files, and technical notes published in the individual
[DVMProject repositories](https://github.com/DVMProject). They are a convenience layer over
those upstream sources — where this site and a project repository disagree, **the repository
is authoritative**. Every page links its sources; corrections are welcome via
[pull request](https://github.com/DVMProject/dvmdocs) (see [Contributing](contributing.md)).
```

## Start here

::::{grid} 1 1 2 2
:gutter: 3

:::{grid-item-card} 🧭 Introduction
:link: overview/introduction
:link-type: doc

What DVMProject is, the supported digital modes, and the usage policy.
:::

:::{grid-item-card} 🚀 Getting Started
:link: overview/getting-started
:link-type: doc

Decide what to build and which hardware and software you need.
:::

:::{grid-item-card} 🗺️ Example System Configurations
:link: overview/system-configs
:link-type: doc

Conventional, control-channel, trunked, and multi-site setups.
:::

:::{grid-item-card} 📖 Glossary
:link: overview/glossary
:link-type: doc

WACN, NAC, TSCC, DFSI, FNE, and the rest of the alphabet soup.
:::

::::

## Explore

::::{grid} 1 1 3 3
:gutter: 3

:::{grid-item-card} 💾 Software
:link: software/dvmhost/index
:link-type: doc

`dvmhost`, `dvmfne`, `dvmbridge`, `dvmcmd`, firmware, and the console.
:::

:::{grid-item-card} 🔌 Hardware
:link: hardware/index
:link-type: doc

The DVM-V1 duplex modem and the DVM-V24 Motorola V.24 adapter.
:::

:::{grid-item-card} 🛠️ Guides
:link: guides/networking
:link-type: doc

Identity tables, FNE networking, encryption, bridging, migration, and
troubleshooting.
:::

::::

## Getting help

- **Discord:** <https://discord.gg/3pBe8xgrEz>
- **Source & issues:** <https://github.com/DVMProject>

```{tableofcontents}
```
