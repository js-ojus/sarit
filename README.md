# `sarit`
Original author: Srinivas JONNALAGADDA &lt;js@ojuslabs.com&gt;

## Status

`sarit` is in early development currently, and does **not** yet have usable releases.

## Introduction

`sarit` (_IPA_: sʌrɪt) is a tiny state machine + workflow engine.
It is developed using the [Go](https://go.dev) programming language [^goVer].

`sarit` provides primitives that can be used to define and manage:

* business process flows and
* policy chains.

These can be used in process-driven applications, in which logical documents move through different states in response to user and system actions.

Examples include front-office — back-office document flows, issue ticket kind of flows, and maker — checker flows such as leave approvals, expense approvals or resource requisition approvals.

> ![TIP](icons/doc_tip.png) **TIP**
>
> `sarit` is a Sanskrit word meaning 'stream', 'river' - something that _flows_!

### Principles

The following are the high-level guiding principles of `sarit`.

* `sarit` should be small and easily maintainable.
* `sarit` should have low cognitive load.
* `sarit` should provide flexible primitives that enable a rich variety of use cases.
* `sarit` should minimize external dependencies.

The following are the current dependencies.

* [SQLite3](https://sqlite.org) for persistence [^sqliteVer].
* [mattn's go-sqlite3](https://github.com/mattn/go-sqlite3) as the database driver
* [julienschmidt's httprouter](https://github.com/julienschmidt/httprouter) for efficient routing
* [oklog's ulid](https://github.com/oklog/ulid) for easily sortable database IDs
* [Uber's Zap](https://go.uber.org/zap) for logging

### Non-goals

`sarit` is expressly *not* intended to be an enterprise-grade workflow engine.
The following features are **not** supported.

* Import from, and export to, workflow modelling formats like BPMN and XPDL.
* Executable specifications like BPEL and Wf-XML.
* Hierarchical regimes (only graph-like regimes are supported).
* Choreography (only orchestration is supported).

## (Informal) Specification

An informal specification of `sarit` can be found at [SPEC](SPEC.md).
It is mandatory reading for anyone who intends to either use or understand `sarit`.

[^goVer]: Requires >= v1.19.

[^sqliteVer]: Requires >= v3.37.

____

### Icon credits:

'Warning' by Thengakola from <a href="https://thenounproject.com/browse/icons/term/warning/" target="_blank" title="Warning Icons">Noun Project</a>

'Idea' by Numero Uno from <a href="https://thenounproject.com/browse/icons/term/idea/" target="_blank" title="Idea Icons">Noun Project</a>

'Flag' by Oksana Latysheva from <a href="https://thenounproject.com/browse/icons/term/flag/" target="_blank" title="Flag Icons">Noun Project</a>

'Tips' by Adrien Coquet from <a href="https://thenounproject.com/browse/icons/term/tip/" target="_blank" title="Tip Icons">Noun Project</a>

'Danger' by ircham from <a href="https://thenounproject.com/browse/icons/term/danger/" target="_blank" title="Danger Icons">Noun Project</a>
