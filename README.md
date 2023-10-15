# UCAN Invocation Specification v1.0.0-rc. 1

FIXME show proxy invocation
FIXME show delegated execution (work stealing) example
FIXME move pipelines, await, etc to own spec
FIXME explain in FAQ why we don't need `join` (i.e. use promises, since the model is inverted)
FIXME reverse lookup secion

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

## 1.3 Serialization

Unlike [UCAN Delegation]s, Invocations are point-to-point. This means that — aside from an Instruction ID — the exact format need only be agreed on by the two parties directly communicating. The examples in this document are given as [JWT], but others (such as [`invocation-ipld`]) MAY be used as long as it's accepted by both parties.

## 1.4 Public Resources

A core part of UCAN's design is interacting with the wider, non-UCAN world. Many resources are open to anyone to access, such as unauthenticated web endpoints. Unlike UCAN-controlled resources, an invocation on public resources is both possible, and a hard required for initiating flow (e.g. signup). These cases typically involve a reference passed out of band (such as a web link). Due to [designation without authorization], knowing the URI of a public resource is often sufficient for interacting with it. In these cases, the Executor MAY accept Invocations without having a "closed-loop" proof chain , but this SHOULD NOT be the default behaviour.

# 2 Concepts

## 2.1 Roles

Task adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delegate principals MUST persist to the invocation.

| UCAN Field | Delegation                             | Invocation                      |
| ---------- | -------------------------------------- | ------------------------------- |
| `iss`      | Delegator: transfer authority (active) | Invoker: request task (active)  |
| `aud`      | Delegate: gain authority (passive)     | Executor: perform task (active) |

### 2.1.1 Invoker

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

### 2.1.2 Executor

The executor is directed to perform some task described in the UCAN by the invoker.

The executor MUST be the UCAN delegate. Their DID MUST be set the in `aud` field of the contained UCAN.

## 2.2 Lifecycle

At a very high level:

- A [Task] absractly requests some action
- An [Invocation] attaches proven ([Delegation]) authority to a [Task]
- A [Receipt] MAY request that the Invoker enqueue more [Task]s

``` mermaid
erDiagram
    Delegation }o--|{ Invocation: proves
    Invocation }|--|| Task: requests
    Invocation ||--|| Receipt: returns
    Receipt |o--|{ Task: enqueues
```

## 2.3 Anatomy

| Concept      | Description                                                                 |
|--------------|-----------------------------------------------------------------------------|
| [Command]    | (Deferred) function application; a description of work to be performed      |
| [Task]       | Contextual information for a [Command], such as resource limits             |
| [Invocation] | A request to perform some [Task] based on [delegated][Delegation] authority |
| [Result]     | The success value or error information from the run [Invocation]            |
| [Receipt]    | The return value from an [Invocation], which may [enqueue] more tasks       |

``` mermaid
flowchart RL
    subgraph Invocation
        direction LR
        subgraph InvocationPayload
            subgraph Task
                direction TB

                Command
                DelegationProofs["Delegation Proofs"]
            end
        end

        Inv1Sig
    end

    subgraph Receipt
        subgraph ReceiptPayload
            Ran
            Result
            Enqueue
        end

        RecSig[Signature]
    end

    subgraph Invocation2["(Next Invocation)"]
        direction RL

        subgraph InvocationPayload2["Invocation Payload"]
            subgraph Task2
                Command2["Command"]
                DelegationProofs2["Delegation Proofs"]
            end

            Cause
        end

        Inv2Sig[Signature]
    end

    Cause -->|provenance| Enqueue
    Ran   -->|provenance| Invocation
```

# 3 Request

## 3.1 Command

A Command is the smallest unit of work that can be Invoked. It is akin to a function call or actor message.

The `input` field is free form, and depend on the specific resource and ability being interacted with, and is not described in this specification.

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
  msg.send({
    from: "mailto:alice@example.com",
    to: ["bob@example.com", "carol@example.com"],
    subject: "hello",
    body: "world"
  })
```

### 3.1.1 Fields

| Name        | Field | Type                | Required |
|-------------|-------|---------------------|----------|
| [Subject]   | `sub` | DID                 | No       |
| [Command]   | `cmd` | String              | Yes      |
| [Arguments] | `arg` | Map `{String: Any}` | Yes      |
| [Nonce]     | `nnc` | String              | Yes      |

#### 3.1.1.1 Command

The Command (`cmd`) field MUST contain a concrete operation that can be sent to the Resource. This field can be thought of as the message or trait being sent to the resource. Note that _unlike_ a [UCAN Ability], which includes heirarchy, an Operation MUST be fully concrete.

#### 3.1.1.2 Arguments

The Arguments (`arg`) field, MAY contain any parameters expected by the Resource/Operation pair, which MAY be different between different Resources and Operations, and is thus left to the executor to define the shape of this data. This field MUST be representable as a map or keyword list.

UCAN capabilities provided in [Proofs] MAY impose certain constraint on the type of `input`s allowed.

If the `input` field is not present, it is implicitly a `unit` represented as empty map.

#### 3.1.1.3 Subject

The OPTIONAL `sub` field is intended for cases where parametrizing a specific agent is important. This is especially critical for two parts of the lifecycle:

1. Specifying a particular `sub` (and thus `aud`) when [enqueuing new Tasks][enqueue] in a Receipt
2. Indexing Receipts for reverse lookup and memoization

#### 3.1.1.4 Nonce

If present, the REQUIRED `nnc` field MUST include a random nonce expressed in ASCII. This field ensures that multiple (non-idempotent) invocations are unique. The nonce SHOULD be `""` for Commands that are idempotent.

### 3.1.2 Exmaples

Interacting with an HTTP API

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

Sending Email:

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

Running WebAssembly. In this case, the Wasm module is inlined.

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

## 3.2 Task

As [noted in the introduction][lazy-vs-eager], there is a difference between a reference to a function and calling that function. The [Invocation] is an instruction to the [Executor] to perform enclosed [Task]. [Invocation]s are not executable until they have been provided provable authority (in form of UCANs in the `prf` field) and an [Authorization] (in the `auth` field) from the [Invoker].

The `auth` field MUST be contain an [Authorization] which signs over the `&Task` in `run`.

Concretely, this means that the `&Task` MUST be present in the associated `auth`'s `scope` field. A `Receipt` where the associated [Authorization] does not include the [Task] in the `scope` MUST be considered invalid.

Tasks wrap a [Command] with contextual information. This includes expiration time, delegation chain, and extensible metadata for things like resource limits.

| Field | Type                      | Description                                                                                  | Required |
|-------|---------------------------|----------------------------------------------------------------------------------------------|----------|
| `run` | `&Command`                | The `run` field MUST contain a link to the [Task] to be run                                  | Yes      |
| `mta` | `{String : Any}`          | Extensible fields, e.g. resource limits, human-readable tags, notes, and so on               | Yes      |
| `prf` | `[&Delegation]`           | Links to any [UCAN Delegation]s that provide the authority to perform the enclosed [Command] | Yes      |
| `exp` | `Integer | null`[^js-num] | The UTC Unix timestamp at which the Task expires                                             | Yes      |

## 3.3 Invocation

The Invocation attaches the authentication information required to authorize the [Task]. It consists of an InvocationPayload, and a signture envelope.

### 3.3.1 InvocationPayload

### 3.3.2 Invocation (Envelope)
 
| Field | Type | Required | Description |
|------|-------|--------|-------------|
| `uci` | `InvocationPayload` | Yes | 

# 4 Response

## 4.1 Result

A Result records the output of the [Task], as well as its success or failure state. A Result MUST be formatted as map with a single `tag` field.

| Tag     | Value            | Description                                     |
|---------|------------------|-------------------------------------------------|
| `ok`    | `Any`            | The successfully returned data from a [Command] |
| `error` | `{String : Any}` | Error information from a failed [Command]       |

```js
// Success
{ "ok": 42 }

// Failure
{
  "error": {
    "dev/reason": "unauthorized",
    "http/status": 401
  }
}
```

# 4.2 Receipt

A `Receipt` is an attestation of the [Result] and requested [Effect]s by a [Task] [Invocation]. A Receipt MUST be signed by the [Executor] or it's delegate. If signed by the delegate, the proof of delegation from the [Executor] to the delegate (the `iss` of the receipt) MUST be provided in `prf`.

**NB: a Receipt does not guarantee correctness of the result!** The statement's veracity MUST be only understood as an attestation from the executor.

Receipts MUST use the same version as the invocation that they contain.

## 8.1 Receipt

### 8.1.1 Receipt Payload

| Field | Type               | Required | Description                                                                        |
|-------|--------------------|----------|------------------------------------------------------------------------------------|
| `iss` | `DID`              | Yes      | The DID of the Executor                                                            |
| `ran` | `&Invocation`      | Yes      | MUST be a link to the [Invocation] that the Receipt is for                         |
| `prf` | `[&Delegation]`    | Yes      | [Delegation] proof chain if the Executor was not the `aud` of the `ran` Invocation |
| `out` | `Result`           | Yes      | MUST contain the value output of the invocation in [Result] format                 |
| `enq` | `[&Task]`          | Yes      | Further [Task]s that the [Invocation] would like to enqueue                        |
| `mta` | `{String : Any}`   | Yes      | Additional data about the receipt                                                  |
| `rec` | `&Receipt`         | No       | Recursive `Signed<Receipt>`s if the Invocation was proxied to another Executor     |
| `iat` | `Integer`[^js-num] | No       | The UTC Unix timestand at which the Receipt was issed                              |

A couple of these fields warrant further comment below.

#### 8.1.1.1 Proof

If the Receipt Issuer is not identical to the `aud` field of Invocation referenced in the `ran` field, a [Delegation] proof chain SHOULD be included. If a chain is present, it MUST show that 

``` js
// Pseudocode

const delegation = {
  iss: "did:web:example.com",
  aud: "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
  can: "crud/update",
  // ...
}

const workerDelegation = {
  iss: "did:web:example.com",
  aud: "did:web:worker.not-example.net"
  sub: "did:web:example.com",
  can: "ucan/execute",
  // ...
}

const invocation = {
  iss: "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
  aud: "did:web:example.com",
  prf: [ cid(delegation) ],
  // ...
}

const receipt = {
  iss: "did:web:worker.not-example.net",
  ran: cid(invocation)
  prf: [ cid(workerDelegation) ],
  // ...
}
```

For u

#### 8.1.1.2 Enqueue

The result of an [Invocation] MAY include a request for further actions to be performed via "effects". This enables several things: a clean separation of pure return values from requesting impure tasks to be performed by the runtime, and gives the runtime the control to decide how (or if!) more work should be performed.

Enqueued [Task]s describe requests for future work to be performed. The SHOULD come with [Delegation]s

All [Invocation]s in an [Effect] block MUST be treated as concurrent, unless explicit data dependencies between them exist via promise [Pipeline]s. The `fx` block contains two fields: `fork` and `join`.

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

### 8.1.2 Receipt Envelope

| Field | Type              | Required | Description                                                              |
|-------|-------------------|----------|--------------------------------------------------------------------------|
| `uci` | `&ReceiptPayload` | Yes      | The CID for the [Receipt Payload]                                        |
| `sig` | `Signature`       | Yes      | The [Signature] of the `uci` value, by the [Receipt Payload]'s `iss` DID |

### 8.1.3 DAG-JSON Examples

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





NOTE: can make great use of batrch signatures
NOTE TO SELF: batch signatures need trees, too?

``` mermaid
sequenceDiagram
    actor User

    participant Service

    participant WorkQueue
    participant Worker1
    participant Worker2

    autonumber

    Note over Service, Worker2: Service Setup
        WorkQueue -->> Service: delegate(WorkQueue, queue/push)
        Service   -->> WorkQueue: delegate(Service, ucan/proxy/sign)

        WorkQueue -->> Worker1: delegate(WorkQueue, queue/pop)
        WorkQueue -->> Worker1: delegate(Service, ucan/proxy/sign)

        WorkQueue -->> Worker2: delegate(WorkQueue, queue/pop)
        WorkQueue -->> Worker2: delegate(Service, ucan/proxy/sign)

    Note over User, Service: Delegates to User
        Service -->> User: delegate(Service, crud/update)

    Note over User, Worker2: Invocation with Proxy Execution
        User ->> Service: invoke(Service, [crud/update, "foo", 42], prf: [➐])
        Service -) WorkQueue: invoke(WorkQueue, queue/push, [➑], prf: [➊])

        Note over WorkQueue, Worker2: Work Stealing
        Worker2 ->>+ WorkQueue: invoke(WorkQueue, queue/pop, prf: [➎])
        WorkQueue ->>- Worker2: receipt(inv: ➓, out: ➒)
        Worker2 ->> Worker2: Execute!(➒)
        Worker2 -) User: receipt(out: ok, inv: ➒, prf: [➏,➋])
```
















<!--
FOR PROMISE SPEC
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
-->
