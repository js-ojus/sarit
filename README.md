<!--
   Copyright 2020 Srinivas JONNALAGADDA <js@ojuslabs.com>

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

## Status
`sarit` is in design phase currently, and is *not* yet usable.

## `sarit`
`sarit` (IPA: sɐrɪt) is a tiny workflow engine that is developed using Go, and currently requires MariaDB (or MySQL).

It provides primitives that can be used to define and manage:
- business workflows and
- privileges.

`sarit` _is a Sanskrit word meaning 'stream', 'river' - something that flows!_.

### Express Non-goals

`sarit` is intended to be small!  It is expressly **not** intended to be an enterprise-grade workflow engine.  Accordingly, import from - and export to - workflow modelling formats like BPMN/XPDL are not supported.  Similarly, executable specifications like BPEL and Wf-XML are not supported.  True enterprise-grade engines already exist for addressing such complex workflows and interoperability requirements.

## Concepts

Let us familiarise ourselves with the most important concepts and moving parts of `sarit`.  Even as you read the following, it is highly recommended that you read the database table definitions in `sql` directory, as well as the corresponding object definitions in their respective `*.go` files.  That should help you in forming a mental model of `sarit` faster.

### Users
`sarit` assumes that:
- user management and
- user authentication
are performed by a different service.

Users are represented in `sarit` only by their unique (numeric) IDs.  In addition, `sarit` stores an application-defined (external) string ID for each user.  These string IDs must be unique too.  This is intended to make it easier to cross-link `sarit` data with the application's users.

In addition, the status of each user has to be posted to `sarit` by the said service, as and when such status changes.

### Groups
Groups are collections of users.  Groups are recursive, in that a group can contain other groups.  In practice, this presents some interesting resolution and performance challenges.

Groups are not directly collections of users, though.  A group's membership is defined using a relation (see below).  All users (or groups) that satisfy the specified relation, are included in the group.  Thus, in general, the relationship between users and groups is *M:N*.

Group names must satisfy the following regular expression.

```
[A-Za-z][A-Za-z0-9_]{2,100}
```
