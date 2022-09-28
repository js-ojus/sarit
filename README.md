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

## This Document

This document is intended to serve as an informal specification of `sarit`.
The implementation shall follow this informal specification.
Each variance between them will be a bug in either or both.

## API

`sarit` publishes a REST API that is accessible over HTTPS.
Specify the paths to the certificate file and the key file in the configuration, for use by TLS.

> ![TIP](icons/doc_tip.png) **TIP**
>
> Depending on the network configuration and your specific use case, using a self-signed certificate may suffice.

Authentication is through a client ID and a client secret, both of which have to be set in the request's HTTP header.
The header keys are `X-Sarit-Client-ID` and `X-Sarit-Client-Secret`, respectively.
An initial client ID and its client secret are generated for *admin* during setup.
These can be found in the setup log, which is located in `log` directory in the specified database path.
Once `sarit` service is started, this key pair can be used to connect to the service and perform all other desired actions.

An example header is shown below.

```
X-Sarit-Client-Id: 4AXHLL4KCQPERDXR
X-Sarit-Client-Secret: Z6EKGC2CDOVZTGPQOSVL6KNS64MWZZV2
```

Payloads are in JSON format, unless expressly stated otherwise.
Please refer to the documentation for further details.

An example request to list all registered users is shown below.

```sh
curl -H 'X-Sarit-Client-Id: 4AXHLL4KCQPERDXR' \
     -H 'X-Sarit-Client-Secret: Z6EKGC2CDOVZTGPQOSVL6KNS64MWZZV2' \
     -X GET \
     https://172.16.3.150:8080/users
```

> ![WARNING](icons/doc_warning.png) **WARNING**
>
> `sarit` is intended to be run inside your private network. Do **not** expose it directly to the public Internet.

## Concepts

Let us familiarize ourselves with the important concepts and moving parts of `sarit`.

Even as you read the following, it is highly recommended that you read the database table definitions in `cmd/setup`.
That should help you in forming a mental model of `sarit` faster.

### Users

`sarit` assumes that:

* user management and
* user authentication,

are performed by an appropriate external service.

Users (and other entities) are represented in `sarit` by their globally-unique [ULIDs](https://github.com/ulid/spec).
In addition, `sarit` stores an application-defined string ID for each user.
These string IDs - referred to as *name* - must be unique too.
This is intended to facilitate cross-linking `sarit` data with the application's user data.

Usernames must have a size in the closed interval `[6,128]`. They must satisfy the following regexp.

```regex
[A-Za-z0-9][A-Za-z0-9\-_.@]{4,126}[A-Za-z0-9]
```

Email addresses are usually good usernames.

The special user `administrator` is predefined by `sarit`. This user has privileges granted on all legal actions in the system.

Each user must be registered with `sarit` before they can participate in process flows.
The user's ULID is generated during registration.

Users cannot be deleted.

> ![WARNING](icons/doc_warning.png) **WARNING**
>
> The current user, on whose behalf an action is requested, is assumed by `sarit` to be an active user of the system.
> Thus, the requesting application **must** ensure that the said user is actually a valid and an active user.

Services are represented as users, as well.
Accordingly, privileges given to services follow the same rules and mechanisms as those governing users.

### Groups

Groups are collections of users and other groups.
In general, the relationship between users and groups is **M:N**.

Groups are represented in `sarit` by their globally-unique ULIDs.
Group names must be unique and have a size in the closed interval `[6,128]`.
They must satisfy the following regexp.

```regex
[A-Za-z0-9][A-Za-z0-9\-_:/]{4,126}[A-Za-z0-9]
```

The special group `admin` is predefined by `sarit`.
Members of this group have privileges granted on all legal actions in the system.
The special user `administrator` is a member of this group.

When naming groups, it is common to express hierarchies using a combination of colon and forward slash.
For instance, `sales:in/south` could represent all sales people of the organization who operate in the southern region of India.
Similarly, `fin:us/west` could represent all finance personnel operating along the west coast of the USA.

`sarit` provides the following predefined states for groups:

* `A` (active),
* `I` (inactive),

with self-evident meanings.

Groups cannot be deleted.
However, they can be inactivated at any time.
New tasks cannot be delivered to an inactive group's **Inbox**.

### Namespaces

A namespace literally defines a namespace in which process flow instances reside.
Each application (or service) that uses `sarit` must define its own distinct namespace.

Namespace names must be unique and have a size in the closed interval `[2,50]`.
They must satisfy the following regexp.

```regex
[A-Za-z0-9][A-Za-z0-9\-_:/]{0,48}[A-Za-z0-9]
```

A process flow created by an application resides in that application's namespace.
In addition, privileges granted (or revoked) to (or from) users and groups can be scoped by namespaces.

`sarit` provides the following predefined states for namespaces:

* `A` (active),
* `I` (inactive),

with self-evident meanings.

Namespaces cannot be deleted.
However, they can be inactivated at any time.
New workflow instances cannot start in inactivated namespaces.

> ![NOTE](icons/doc_note.png) **NOTE**
>
> `sarit` has a reserved namespace called `sys`, in which system metadata is maintained.

### Process Flows

A process flow defines a directed graph of the exhaustive possibilities of flow states and the transitions between them, in a particular business process.

> ![TIP](icons/doc_tip.png) **TIP**
>
> We use "process flow" and "workflow" interchangeably.

Workflow names must be unique within their respective namespaces, and have a size in the closed interval `[2,50]`.
They must satisfy the following regexp.

```regex
[A-Za-z0-9][A-Za-z0-9\-_:/]{0,48}[A-Za-z0-9]
```

`sarit` provides the following predefined states for namespaces:

* `D` (draft),
* `A` (active),
* `I` (inactive),

with self-evident meanings.

Workflows cannot be deleted.
However, they can be inactivated at any time.
New instances of inactivated workflows cannot be started.

Flows can - and often do - have cycles.

Each initiation of a defined process flow creates and starts an associated flow instance in the system.
It is valid for a workflow instance to be long-lived: it can 'get stuck' for an extended duration without terminating.

### States

Each process flow must have:

- a start state in which the process flow begins,
- one or more intermediate states, and
- an end state which terminates the process flow.

All defined process flow states must be reachable from the start state.

State names must be unique within their respective workflows, and have a size in the closed interval `[2,50]`.
They must satisfy the following regexp.

```regex
[A-Za-z0-9][A-Za-z0-9\-_]{0,48}[A-Za-z0-9]
```

Process states cannot be deleted.

The workflow behaviour of each state is determined by the types of its incoming and outgoing connections, as defined by the possible number of users who can operate on the flow into and out of that state.
`sarit` provides the following types of connections.

**Incoming**

* None (_for start states_)
* Exactly one (_for linear flows_)
* At least `n` (_for quorum flows_)
* All of `n` (_for unanimous approvals, etc._)

**Outgoing**

* None (_for end states_)
* Exactly one (_for linear flows_)
* All of `n` (_for branching_)

### Events

Flows undergo state transitions in response to user and system actions. In `sarit`, such actions are represented by *events*.

Event names must be unique within their respective workflows, and have a size in the closed interval `[2,50]`.
They must satisfy the following regexp.

```regex
[A-Za-z0-9][A-Za-z0-9\-_]{0,48}[A-Za-z0-9]
```

Process events cannot be deleted.

A flow object (instance) that is in a given state, upon receiving an applicable event, either transitions to another unique state, or may trigger multiple sub-flows.
Such sub-flows, when defined as above, run in parallel.
It is _not_ necessary for a unique downstream rendezvous point to merge all such sub-flows.

> ![EXPLANATION](icons/doc_explanation.png) **State Transitions vs. Connections**
>
> The topology of the state transitions graph is orthogonal to that of workflow connections.
> Thus, each process flow has two conceptual graphs that together fully define its functionality.
> Though both are driven by events, the underlying mechanisms of fulfilment are different for each.

### Tasks

A **task** is a process flow object that is _not_ in an end state.

Each user and group in `sarit` has an **Inbox** of tasks, and an **Outbox** of tasks or flow objects in their end states.

Inbox is a destination for tasks that are assigned to a user or a group. Each task is marked with a priority level.
`sarit` provides the following levels of priorities:

- High,
- Medium (_default_) and
- Low.

Tasks can be **claim**ed from a group Inbox by a user of that group.
Such tasks get moved into the Inbox of that particular user, and become unavailable to the other users of that group to claim.

Each group in `sarit` has one or more *group owner*s, who have the authority to either manually assign or reassign tasks in that group's Inbox to users of that group.

Each process flow in `sarit` has one or more *process owner*s, who have the authority to either manually assign or reassign tasks of that flow to various users and groups.

In addition, the `sarit` system has one or more *admin*s, who can perform various housekeeping and administrative activities of the system.
Such admins can also either manually assign or reassign tasks of a flow to various users and groups.

A user can **reject** a task that is either claimed by them or assigned to them.
A rejected task goes back to the group Inbox from where it was claimed or assigned.
If it was directly assigned to the user, then it gets redirected to the Inbox of the user's manager.

> ![NOTE](icons/doc_note.png) **NOTE**
>
> Movement of a task from the group Inbox to a user's Inbox does _not_ alter its priority.

Once a user performs an action on a task, should the action result in a state transition of that object, it moves to the user's Outbox.

> ![EXPLANATION](icons/doc_explanation.png) **Task Exclusivity**
>
> An object can never simultaneously be in a given user's Inbox and Outbox.
> As a corollary, should a user's action on a task result in a transition of that object into a state that assigns the object as the next task to the same user, it returns to the Inbox of that user.

Inactive groups cannot be the recipients of tasks.
Those tasks that are already in such a group's Inbox at the time of it getting inactivated, must be:

* claimed by users of that group,
* assigned to users of that or a different group, or
* reassigned fully to another group, whose users can take them up.

### Privileges

> ![NOTE](icons/doc_note.png) **NOTE**
>
> `sarit` is **not** a general purpose authorization engine.
> The privileges in `sarit` are used only for workflow management within `sarit`.

`sarit` defines a set of privileges that can be given to users.
By standardizing these privileges, `sarit` enables a quick determination of who can perform which operations on a given process flow object (set) and in which states of that flow.

In the application of privileges, `sarit` determines them according to the following rules.

* If an object is in a group's Inbox, and a user is a part of that group, they get **view** privilege on that object.

* If an object is in a user's Inbox, they get **edit** privilege on that object.

* If an object is in a user's Outbox, they get **view** privilege on that object.

* If an object is elsewhere, they get no access to that object.

The above and related rules are summarized in the following tables.

**View / Edit Privileges**

| | User | Group Owner | Process Owner | Admin |
| --- | :---: | :---: | :---: | :---: |
| *Group Inbox* | View | View | View | View |
| *User's Own Inbox* | Edit | View | View | View |
| *User's Own Outbox* | View | View | View | View |
| *Elsewhere Within Group* | — | View | View | View |
| *Elsewhere Outside Group But Within Process* | — | — | View | View |
| *Elsewhere* | — | — | — | View |

**Assignment Privileges**

| | User | Group Owner | Process Owner | Admin |
| --- | :---: | :---: | :---: | :---: |
| *Group Inbox* | Claim | Assign | Assign | Assign |
| *User's Own Inbox* | Reject | Retract (to Group Inbox), Reassign Within Group | Reassign Within Process | Reassign Within Process |

**Task Claim and Rejection by User**

| |Initial Location |Current Location |New Location |
| --- | :---: | :---: | :---: |
| *Claim* | Group Inbox | Group Inbox | User Inbox |
| *Reject* | Group Inbox | User Inbox | Group Inbox |
| *Reject* | Other | User Inbox | User's Manager's Inbox |

## TODO

VALIDATION OF STATES

VALIDATION OF TRANSITIONS: from ≠ to

QUORUM FOR TRANSITIONS

* Special meaning of `0`.

QUORUM FOR JOIN STATES

* At least N
* All: dynamic determination of N
** Discourage its use

TASKS

* Difference between assignment to a user's Inbox and a group's.
* Special handling for recursive groups.
* Lazy validation of access and availability to a user.

[^goVer]: Requires >= v1.19.

[^sqliteVer]: Requires >= v3.37.

____

### Icon credits:

'Warning' by Thengakola from <a href="https://thenounproject.com/browse/icons/term/warning/" target="_blank" title="Warning Icons">Noun Project</a>

'Tips' by Adrien Coquet from <a href="https://thenounproject.com/browse/icons/term/tips/" target="_blank" title="Tips Icons">Noun Project</a>

'Flag' by Oksana Latysheva from <a href="https://thenounproject.com/browse/icons/term/flag/" target="_blank" title="Flag Icons">Noun Project</a>

'Bulb' by 1art from <a href="https://thenounproject.com/browse/icons/term/bulb/" target="_blank" title="bulb Icons">Noun Project</a>
