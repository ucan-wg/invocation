# UCAN Invocation Specification v1.0.0-rc. 1

FIXME
FIXME show proxy invocation
FIXME show delegated execution (work stealing) example
- move pipelines, await, etc to own spec
- explain in FAQ why we don't need `join` (i.e. use promises, since the model is inverted)
- operation -> command, Instruction -> action
- Be clear that you MUST sign

## Editors

- [Brooklyn Zelenka], [Fission]
- [Irakli Gozalishvili], [DAG House]

## Authors

- [Brooklyn Zelenka], [Fission]
- [Irakli Gozalishvili], [DAG House]
- [Zeeshan Lakhani], [Fission]

## Depends On

- [UCAN Delegation]

# 0 Abstract

UCAN Invocation defines a format for expressing the intention to execute delegated UCAN capabilities, and the attested receipts from an execution.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] when, and only when, they appear in all capitals, as shown here.

# 1 Introduction

> Just because you can doesn't mean that you should
>
> — Anonymous

> When authorizaton is communicated without such context, it's like receiving a key in the mail with no hint about what to do with it [...] After an object receives this message, she can invoke arg if she chooses, but why would she ever choose to do so?
>
> Mark Miller, [E-lang Mailing List, 2000 Oct 18]

UCAN is a chained-capability format. A UCAN contains all of the information that one would need to perform some task, and the provable authority to do so. This begs the question: can UCAN be used directly as an RPC language?

Some teams have had success with UCAN directly for RPC when the intention is clear from context. This can be successful when there is more information on the channel than the UCAN itself (such as an HTTP path that a UCAN is sent to). However, capability invocation contains strictly more information than delegation: all of the authority of UCAN, plus the command to perform the task.

## 1.1 Intuition

## 1.1.1 Car Keys

Consider the following fictitious scenario:

Akiko is going away for the weekend. Her good friend Boris is going to borrow her car while she's away. They meet at a nearby cafe, and Akiko hands Boris her car keys. Boris now has the capability to drive Akiko's car whenever he wants to. Depending on their plans for the rest of the day, Akiko may find Boris quite rude if he immediately leaves the cafe to go for a drive. On the other hand, if Akiko asks Boris to run some last minute pre-vacation errands for that require a car, she may expect Boris to immediately drive off.

## 1.1.2 Lazy vs Eager Evaluation

In a referentially transparent setting, the description of a task is equivalent to having done so: a function and its results are interchangeable. [Programming languages with call-by-need semantics][Haskell] have shown that this can be an elegant programming model, especially for pure functions. However, _when_ something will run can sometimes be unclear.

Most languages use eager evaluation. Eager languages must contend directly with the distinction between a reference to a function and a command to run it. For instance, in JavaScript, adding parentheses to a function will run it. Omitting them lets the program pass around a reference to the function without immediately invoking it.

```js
const message = () => alert("hello world")
message // Nothing happens
message() // A message interrupts the user
```

Delegating a capability is like the statement `message`. Task is akin to `message()`. It's true that sometimes we know to run things from their surrounding context without the parentheses:

```js
[1, 2, 3].map(message) // Message runs 3 times
```

However, there is clearly a distinction between passing a function and invoking it. The same is true for capabilities: delegating the authority to do something is not the same as asking for it to be done immediately, even if sometimes it's clear from context.

## 1.2 Delegation Gossip

UCAN delegation can be gossiped freely between services. This is not true for invocation.

For example, if `alice@example.com` delegates her `web3.storage` storage quota to `bob@example.com`, it may be beneficial for all of the related `web3.storage` services to cache this information. If this were to be understood as an invocation, then gossiping this information would lead to validation failures due to principal misalignment in the certificate chain.

By distinguishing invocation from delegation, agents are able to understand the user intention, and handle such messages accordingly. Receipt of an invocation with misaligned principles will fail, but a delegation may be held in e.g. Bob's proxy inbox to be acted on when he comes online or widely distributed across the `web3.storage` infrastructure.

## 1.3 Separation of Concerns

Information about the scheduling, order, and pipelining of tasks is orthogonal to the flow of authority. An agent collaborating with the original executor does not need to know that their call is 3 invocations deep; they only need to know that they been asked to perform some task by the latest invoker.

As we shall see in the [discussion of promise pipelining][Pipeline], asking an agent to perform a sequence of tasks before you know the exact parameters requires delegating capabilities for all possible steps in the pipeline. Pulling pipelining detail out of the core UCAN spec serves two functions: it keeps the UCAN spec focused on the flow of authority, and makes salient the level of de facto authority that the executor has (since they can claim any value as having returned for any step).

``` mermaid
sequenceDiagram
    participant Alice 💾
    participant Bob
    participant Carol 📧
    participant Dan

    autonumber

    Note over Alice 💾, Dan: Delegations
        Alice 💾 -->> Bob:      Delegate<Read from Alice's DB>
        Bob      -->> Carol 📧: Delegate<Read from Alice's DB>
        Carol 📧 -->> Dan:      Delegate<Read from Alice's DB>
        Carol 📧 -->> Dan:      Delegate<Send email as Carol>

    Note over Alice 💾, Dan: Single Invocation
        Dan      ->>  Alice 💾: Read from Alice's DB!
        Alice 💾 -->> Dan:      Result<➎>

    Note over Alice 💾, Dan: Multiple Invocation Flow
        Dan      ->>  Alice 💾: Read from Alice's DB!
        Alice 💾 -->> Dan:      Result<➐>
        Dan      ->>  Carol 📧: Send email containing Result<➐> as Carol!
        Carol 📧 ->>  Carol 📧: Send email!

    Note over Alice 💾, Dan: Promise Pipeline
        Dan      ->>  Alice 💾: Read from Alice's DB!
        Dan      ->>  Carol 📧: Send email containing Result<⓫> as Carol!
        Alice 💾 -->> Carol 📧: Result<⓫>
        Carol 📧 ->>  Carol 📧: Send email containing Result<⓫> as Carol!
```

## 1.4 Serialization

Unlike [UCAN Delegation]s, Invocations are point-to-point. This means that — aside from an Instruction ID — the exact format need only be agreed on by the two parties directly communicating. The examples in this document are given as [JWT], but others (such as [`invocation-ipld`]) MAY be used as long as it's accepted by both parties.

## 1.5 Public Resources

A core part of UCAN's design is interacting with the wider, non-UCAN world. Many resources are open to anyone to access, such as unauthenticated web endpoints. Unlike UCAN-controlled resources, an invocation on public resources is both possible and 

FIXME Open HTTP GET doensn't need a prf, but can still use invocation

# 2 Roles

Task adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delegate principals MUST persist to the invocation.

| UCAN Field | Delegation                             | Invocation                      |
| ---------- | -------------------------------------- | ------------------------------- |
| `iss`      | Delegator: transfer authority (active) | Invoker: request task (active)  |
| `aud`      | Delegate: gain authority (passive)     | Executor: perform task (active) |

### 2.1.1 Invoker

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

### Subject

### 2.1.2 Executor

The executor is directed to perform some task described in the UCAN by the invoker.

The executor MUST be the UCAN delegate. Their DID MUST be set the in `aud` field of the contained UCAN.

## 3 Concepts

### 3.1 Command

A [Command] is like a deferred function application: a request to perform some action on a resource with specific input.

### 3.2 Task

A [Task] wraps a [Command] with runtime configuration, including timeouts, fuel, trace metadata, and so on.

### 3.3 Invocation

An [Invocation] is an [Authorized] [Task].

### 3.4 Result

A [Result] is the output of an [Command].

### 3.5 Receipt

A [Receipt] is a cryptographically signed description of the [Invocation] output and requested [Effect]s.

# 3 Command

FIXME rename?

An Instruction is the smallest unit of work that can be requested from a UCAN. It describes one `(resource, operation, input)` triple. The `input` field is free form, and depend on the specific resource and ability being interacted with, and is not described in this specification.

Using the JavaScript analogy from the introduction, an Instruction is similar to wrapping a call in a closure:

```json
{
  "act": "msg/send",
  "arg": {
    "from": "mailto:alice@example.com",
    "to": [
      "bob@example.com",
      "carol@example.com"
    ],
    "subject": "hello",
    "body": "world"
  }
}
```

```js
// Pseudocode
() =>
  send(alice, msg.send({
    from: "mailto:alice@example.com",
    to: ["bob@example.com", "carol@example.com"],
    subject: "hello",
    body: "world"
  })
```

```json
{
  iss: "did:example:bob",
  aud: "did:example:alice",
  act: "msg/send",
  nnc: "",
  arg: {
    from: "mailto:alice@example.com",
    to: [
      "bob@example.com",
      "carol@example.com"
    ],
    subject: "hello",
    body: "world"
  }
  meta: {
    fuel: 100,
    disk: "100GB"
  },
  prf: [bafy1, bafy2]
}







``` js
{
  iss: "did:example:bob",
  aud: "did:example:alice", // Origoinally removed these because it's duplicated from prf, but important for e.g. CRDT
  run: {
    sub: "did:example:alice", // <- where, IFF the subject is relevant... only really useful for 
    cmd: "counter/inc",
    arg: {by: 4}
  },
  meta: {},
  cause: {"/": "bafy...123"},
  prf: [bafy1, bafy3],
  exp: 999999 // Doubles as a handy timeout
}
```

``` js
{
  iss: "did:example:bob",
  aud: "did:example:alice", // NOTE: can be ANYONE in the delegation chain for proxying if you don't have a direct path?
  exp: 999999,
  run: {
    act: "wasm/run",
    arg: {
      mod: "ipfs://...",
      fun: "add_one",
      arg: [42]
    }
  },
  prf: [bafy1, bafy3]
}
```


NOTE TO SELF: keep aud field as optional. i.e. make it salient for ergonomic reasons / push it into the task writer's face. also aud & sub MAY diverge in the future.

## 3.1 Fields

| Name        | Field | Type                | Required | Notes        |
|-------------|-------|---------------------|----------|--------------|
| [Subject]   | `sub` | DID                 | No       |              |
| [Command]   | `cmd` | String              | Yes      |              |
| [Arguments] | `arg` | Map `{String: Any}` | Yes      |              |
| [Nonce]     | `nnc` | String              | Yes      | May be empty |

### 3.1.1 Subject

The OPTIONAL `sub` field is intended for cases where parametrizing a specific agent is importnat

<!-- The Resource (`uri`) field MUST contain the [URI] of the resource being accessed. If the resource being accessed is some static data, it is RECOMMENDED to reference it by the [`data`], [`ipfs`], or [`magnet`] URI schemes. -->

### 3.1.3 Command

The Command (`cmd`) field MUST contain a concrete operation that can be sent to the Resource. This field can be thought of as the message or trait being sent to the resource. Note that _unlike_ a [UCAN Ability], which includes heirarchy, an Operation MUST be fully concrete.

### 3.1.4 Arguments

The Arguments (`arg`) field, MAY contain any parameters expected by the Resource/Operation pair, which MAY be different between different Resources and Operations, and is thus left to the executor to define the shape of this data. This field MUST be representable as a map or keyword list.

UCAN capabilities provided in [Proofs] MAY impose certain constraint on the type of `input`s allowed.

If the `input` field is not present, it is implicitly a `unit` represented as empty map.

### 3.1.6 Nonce

If present, the OPTIONAL `nnc` field MUST include a random nonce expressed in ASCII. This field ensures that multiple invocations are unique. The nonce MUST be left blank for commands that are expected to be idempotent.

FIXME better wording

### 3.2.1 Interacting with an HTTP API

```json
{
  // ...
  run: {
    "act": "crud/create",
    "arg": {
      "uri" "https://example.com/blog/posts",
      "headers": {
        "content-type": "application/json"
      },
      "payload": {
        "title": "How UCAN Tasks Changed My Life",
        "body": "This is the story of how one spec changed everything...",
        "topics": [
          "authz",
          "journal"
        ],
        "draft": true
      }
    },
    "nnc": "&NCC-1701-D*"
  }
}
```

### 3.2.2 Sending Email

```json
{
  "act": "msg/send",
  "arg": {
    "from" "mailto:akiko@example.com",
    "to": [
      "boris@example.com",
      "carol@example.com"
    ],
    "subject": "Coffee",
    "body": "Hey you two, I'd love to get coffee sometime and talk about UCAN Invocations!"
  },
  "nnc": "1234567890"
}
```

### 3.2.3 Running WebAssembly

```json
{
  "nnc": "", // FIXME infers idempotence
  "act": "wasm/run",
  "arg": {
    "mod": "data:application/wasm;base64,AHdhc21lci11bml2ZXJzYWwAAAAAAOAEAAAAAAAAAAD9e7+p/QMAkSAEABH9e8GowANf1uz///8UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP////8AAAAACAAAACoAAAAIAAAABAAAACsAAAAMAAAACAAAANz///8AAAAA1P///wMAAAAlAAAALAAAAAAAAAAUAAAA/Xu/qf0DAJHzDx/44wMBqvMDAqphAkC5YAA/1mACALnzB0H4/XvBqMADX9bU////LAAAAAAAAAAAAAAAAAAAAAAAAAAvVXNlcnMvZXhwZWRlL0Rlc2t0b3AvdGVzdC53YXQAAGFkZF9vbmUHAAAAAAAAAAAAAAAAYWRkX29uZV9mAAAADAAAAAAAAAABAAAAAAAAAAkAAADk////AAAAAPz///8BAAAA9f///wEAAAAAAAAAAQAAAB4AAACM////pP///wAAAACc////AQAAAAAAAAAAAAAAnP///wAAAAAAAAAAlP7//wAAAACM/v//iP///wAAAAABAAAAiP///6D///8BAAAAqP///wEAAACk////AAAAAJz///8AAAAAlP///wAAAACM////AAAAAIT///8AAAAAAAAAAAAAAAAAAAAAAAAAAET+//8BAAAAWP7//wEAAABY/v//AQAAAID+//8BAAAAxP7//wEAAADU/v//AAAAAMz+//8AAAAAxP7//wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAU////pP///wAAAAAAAQEBAQAAAAAAAACQ////AAAAAIj///8AAAAAAAAAAAAAAADQAQAAAAAAAA==",
    "fun": "add_one",
    "params": [42]
  }
}
```

# 4 Task

As [noted in the introduction][lazy-vs-eager], there is a difference between a reference to a function and calling that function. The [Invocation] is an instruction to the [Executor] to perform enclosed [Task]. [Invocation]s are not executable until they have been provided provable authority (in form of UCANs in the `prf` field) and an [Authorization] (in the `auth` field) from the [Invoker].

The `auth` field MUST be contain an [Authorization] which signs over the `&Task` in `run`.

Concretely, this means that the `&Task` MUST be present in the associated `auth`'s `scope` field. A `Receipt` where the associated [Authorization] does not include the [Task] in the `scope` MUST be considered invalid.

## 4.1 Task

```
type Task struct {
  run   &Command
  meta  {String : Any}
  cause optional &Receipt
}
```

### 5.1.1 Task

The `run` field MUST contain a link to the [Task] to be run.

### 5.1.2 Metadata

The OPTIONAL `meta` field MAY be used to include human-readable descriptions, tags, execution hints, resource limits, and so on. If present, the `meta` field MUST contain a map with string keys. The contents of the map are left undefined to encourage extensible use.

If `meta` field is not present, it is implicitly a `unit` represented as an empty map.

### 5.1.3 Proofs

The `prf` field MUST contain links to any UCANs that provide the authority to perform this task. All of the outermost proofs MUST have `aud` field set to the [Executor]'s DID. All of the outmost proofs MUST have `iss` field set to the [Invoker]'s DID.







FIXME expand to move proofs here








### 5.1.4 Optional Cause

[Task]s MAY be invoked as an effect caused by a prior [Invocation]. Such [Invocation]s SHOULD have a `cause` field set to the [Receipt] link of the [Invocation] that caused it. The linked [Receipt] MUST have an `Effect` (the `fx` field) containing invoked [Task] in the `run` field.

### 5.2.1 Task

The [Task] containing the [Instruction] and any configuration.

# 6 Result

A Result records the output of the [Task], as well as its success or failure state.

## 6.1 Schema

```
type Result union {
  | any "ok"
  | any "error"
} representation keyed
```

## 6.2 Variants

## 6.2.1 Success

The success branch MUST contain the value returned from a successful [Task] wrapped in the `"ok"` tag. The exact shape of the returned data is left undefined to allow for flexibility in various Task types.

```json
{ "ok": 42 }
```

## 6.2.2 Failure

The failure branch MAY contain detail about why execution failed wrapped in the "error" tag. It is left undefined in this specification to allow for [Task] types to standardize the data that makes sense in their contexts.

If no information is available, this field SHOULD be set to `{}`.

```json
{
  "error": {
    "dev/reason": "unauthorized",
    "http/status": 401
  }
}
```

## 7 Effects

The result of an [Invocation] MAY include a request for further actions to be performed via "effects". This enables several things: a clean separation of pure return values from requesting impure tasks to be performed by the runtime, and gives the runtime the control to decide how (or if!) more work should be performed.

Effects describe requests for future work to be performed. All [Invocation]s in an [Effect] block MUST be treated as concurrent, unless explicit data dependencies between them exist via promise [Pipeline]s. The `fx` block contains two fields: `fork` and `join`.

[Task]s listed in the `fork` field are first-class and only ordered by promises; they otherwise SHOULD be considered independent and equal. As such, atomic guarantees such as failure of one effect implying failure of other effects if left undefined.

The `join` field describes an OPTIONAL "special" [Invocation] which instruct the [Executor] that the [Task] [Invocation] is a continuation of the previous Invocation. This roughly emulates a virtual thread which terminates in an Invocation that produces Effect without a `join` field.

Tasks in the `fork` field MAY be related to the Task in the `join` field if there exists a Promise referencing either Task. If such a promise does not exist, then they SHOULD be treated as entirely separate and MAY be scheduled, deferred, fail, retry, and so on entirely separately.

## 7.1 Schema

```
# Represents a request to invoke enclosed set of tasks concurrently
type Effects {
  fx [&Invocation]
}
```

## 7.2 Fields


# 8 Receipt

A `Receipt` is an attestation of the [Result] and requested [Effect]s by a [Task] [Invocation]. A Receipt MUST be signed by the [Executor] or it's delegate. If signed by the delegate, the proof of delegation from the [Executor] to the delegate (the `iss` of the receipt) MUST be provided in `prf`.

**NB: a Receipt does not guarantee correctness of the result!** The statement's veracity MUST be only understood as an attestation from the executor.

Receipts MUST use the same version as the invocation that they contain.

## 8.1 Receipt

| Field | Type             | Required | Description                                                                        |
|-------|------------------|----------|------------------------------------------------------------------------------------|
| `iss` | `DID`            | Yes      | The DID of the Executor                                                            |
| `prf` | `[&Delegation]`  | Yes      | [Delegation] proof chain if the Executor was not the `aud` of the `ran` Invocation |
| `ran` | `&Invocation`    | Yes      | MUST be a link to the [Invocation] that the Receipt is for                         |
| `out` | `Result`         | Yes      | MUST contain the value output of the invocation in [Result] format                 |
| `enq` | `[&Task]`        | Yes      | Further [Task]s that the [Invocation] would like to enqueue                        |
| `mta` | `{String : Any}` | Yes      | Additional data about the receipt                                                  |
| `rec` | `&Receipt`       | No       | Recursive `Signed<Receipt>`s if the Invocation was proxied to another Executor     |

## 8.3 DAG-JSON Examples

### 8.3.1 Issued by Executor

``` js
{
  "ran": {"/": "bafyreia5tctxekbm5bmuf6tsvragyvjdiiceg5q6wghfjiqczcuevmdqcu"}
  "out": {
    "ok": {
      "members": [
        "bob@example.com",
        "alice@web.mail"
      ]
    }
  },
  "mta": {
    "retries": 2,
    "time": [
      400,
      "hours"
    ]
  },
  "sig": {
    "/": {
      "bytes": "7aEDQLYvb3lygk9yvAbk0OZD0q+iF9c3+wpZC4YlFThkiNShcVriobPFr/wl3akjM18VvIv/Zw2LtA4uUmB5m8PWEAU"
    }
  }
}
```

```json
{
  "ran": { "/": "bafyreia5tctxekbm5bmuf6tsvragyvjdiiceg5q6wghfjiqczcuevmdqcu" },
  "out": {
    "ok": {
      "members": [ "bob@example.com", "alice@web.mail" ]
    }
  },
  "meta": {
    "retries": 2,
    "time": [ 400, "hours" ]
  },
  "sig": { "/": { "bytes": "7aEDQLYvb3lygk9yvAbk0OZD0q+iF9c3+wpZC4YlFThkiNShcVriobPFr/wl3akjM18VvIv/Zw2LtA4uUmB5m8PWEAU" } }
}
```

### 8.3.2 Issued by Delegate

```json
{
  "iss": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
  "prf": [
    { "/": "bafyreihfgvlol74ugosa5gkzvbsghmq7wiqn4xvgack4uwn4qagrml6p74" },
    { "/": "SOME OTHER CID FIXME" }
  ],
  "ran": { "/": "bafyreia5tctxekbm5bmuf6tsvragyvjdiiceg5q6wghfjiqczcuevmdqcu" },
  "out": {
    "ok": {
      "members": [ "bob@example.com", "alice@web.mail" ]
    }
  },
  "mta": {
    "retries": 2,
    "time": [ 400, "hours" ]
  },
  "sig": { "/": { "bytes": "7aEDQKxIrga+88HNDd69Ho4Ggz8zkf+GxWC6dAGYua6l85YgiL3NqGxyGAygiSZtWrWUo6SokgOys2wYE7N+novtcwo" } }
}
```

### 8.3.3 Receipt with Enqueued Tasks

```json
{
  "ran": { "/": "bafyreig3qnao4suz3lchh4joof7fhlobmgxhaal3vw4vtcghtlgtp7u4xy" },
  "out": { "ok": { "status": 200 } },
  "fx": [
      { "/": "bafyreievhy7rnzot7mnzbnqtiajhxx7fyn7y2wkjtuzwtmnflty3767dny" },
      { "/": "bafyreigmmdzix2vxboojvv6j6h7sgvxnrecdxtglwtqpxw7hybebzlsax4" },
      { "/": "bafyreif6gfpzgxnii4ys6a4bjenefg737fb5bgam3onrbmhnoa4llk244q" }
  ],
  "sig": { "/": { "bytes": "7aEDQAHWabtCE+QikM3Np94TrA5T8n2yXqy8Uf35hgw0fe5c2Xi1O0h/JgrFmGl2Gsbhfm05zpdQmwfK2f/Sbe00YQE" } }
}
```

























# 10 Prior Art

[ucanto RPC] from [DAG House] is a production system that uses UCAN as the basis for an RPC layer.

The [Capability Transport Protocol (CapTP)] is one of the most influential object-capability systems, and forms the basis for much of the rest of the items on this list.

The [Object Capability Network (OCapN)] protocol extends CapTP with a generalized networking layer. It has implementations from the [Spritely Institute] and [Agoric]. At time of writing, it is in the process of being standardized.

[Electronic Rights Transfer Protocol (ERTP)] builds on top of CapTP for blockchain & digital asset use cases.

[Cap 'n Proto RPC] is an influential RPC framework [based on concepts from CapTP].

# 11 Acknowledgements

Many thanks to [Mark Miller] for his [pioneering work] on [capability systems].

Many thanks to [Luke Marsen] and [Simon Worthington] for their feedback on invocation model from their work on [Bacalhau] and [IPVM].

Thanks to [Marc-Antoine Parent] for his discussions of the distinction between declarations and directives both in and out of a UCAN context.

Many thanks to [Quinn Wilton] for her discussion of speech acts, the dangers of signing canonicalized data, and ergonomics.

Thanks to [Blaine Cook] for sharing their experiences with OAuth 1, irreversible design decisions, and advocating for keeping the spec simple-but-evolvable.

Thanks to [Philipp Krüger] for the enthusiastic feedback on the overall design and encoding.

Thanks to [Christine Lemmer-Webber] for the many conversations about capability systems and the programming models that they enable.

<!-- Internal Links -->

<!-- External Links -->

[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Bacalhau]: https://www.bacalhau.org/
[Brooklyn Zelenka]: https://github.com/expede/
[DAG House]: https://dag.house
[E-lang Mailing List, 2000 Oct 18]: http://wiki.erights.org/wiki/Capability-based_Active_Invocation_Certificates
[Fission]: https://fission.codes/
[Haskell]: https://en.wikipedia.org/wiki/Haskell
[IPVM]: https://github.com/ipvm-wg
[Irakli Gozalishvili]: https://github.com/Gozala
[Luke Marsen]: https://github.com/lukemarsden
[Mark Miller]: https://github.com/erights
[Philipp Krüger]: https://github.com/matheus23/
[Robust Composition]: http://www.erights.org/talks/thesis/markm-thesis.pdf
[Simon Worthington]: https://github.com/simonwo
[UCAN Ability]: https://github.com/ucan-wg/delegation/#23-ability
[UCAN Delegation]: https://github.com/ucan-wg/delegation/
[URI]: https://en.wikipedia.org/wiki/Uniform_Resource_Identifier
[Zeeshan Lakhani]: https://github.com/zeeshanlakhani
[`data`]: https://en.wikipedia.org/wiki/Data_URI_scheme
[`ipfs`]: https://docs.ipfs.tech/how-to/address-ipfs-on-web/#native-urls
[`magnet`]: https://en.wikipedia.org/wiki/Magnet_URI_scheme
[eRights]: https:/erights.org
[promise pipelining]: http://erights.org/elib/distrib/pipeline.html
[ucanto RPC]: https://github.com/web3-storage/ucanto
