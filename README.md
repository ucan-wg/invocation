# UCAN Invocation Specification v1.0.0-rc. 1

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

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] when, and only when, they appear in all capitals, as shown here.

# 0 Abstract

UCAN Invocation defines a format for expressing the intention to execute delegated UCAN capabilities, and the attested receipts from an execution.

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

## 1.3 Public Resources

A core part of UCAN's design is interacting with the wider, non-UCAN world. Many resources are open to anyone to access, such as unauthenticated web endpoints. Unlike UCAN-controlled resources, an invocation on public resources is both possible, and a hard required for initiating flow (e.g. signup). These cases typically involve a reference passed out of band (such as a web link). Due to [designation without authorization], knowing the URI of a public resource is often sufficient for interacting with it. In these cases, the Executor MAY accept Invocations without having a "closed-loop" proof chain , but this SHOULD NOT be the default behaviour.

## 1.4 Promise Pipelining

[UCAN Promise] extends UCAN Invocation with [distributed promise pipelines]. Promises are helpful in a wide variety of situations for efficiency and convenience. Implementing [UCAN Promise]s is RECOMMENDED.

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

## 2.2 Lifecycle

At a very high level:

- A [Task] absractly requests some action
- An [Invocation] attaches proven ([delegated][Delegation]) authority to a [Task]
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
                DelegationProofs[Delegation Proofs]
            end
        end

        Inv1Sig[Signature]
    end

    subgraph Receipt
        subgraph ReceiptPayload
            Ran
            Result
            Enqueue
        end

        RecSig[Signature]
    end

    subgraph Invocation2["(Next) Invocation"]
        direction RL

        subgraph InvocationPayload2[Invocation Payload]
            subgraph Task2
                Command2["Command"]
                DelegationProofs2[Delegation Proofs]
            end

            Cause
        end

        Inv2Sig[Signature]
    end

    Cause -->|provenance| Enqueue
    Ran   -->|provenance| Invocation
```

# 3 Request

## 3.1 Action

A Command is the smallest unit of work that can be Invoked. It is akin to a function call or actor message. It MUST conform to the following struct shape:

| Name        | Field | Type             | Required |
|-------------|-------|------------------|----------|
| [Subject]   | `sub` | `DID`            | No       |
| [Command]   | `cmd` | `String`         | Yes      |
| [Arguments] | `arg` | `{String : Any}` | Yes      |
| [Nonce]     | `nnc` | `String`         | Yes      |

The `arg` field MUST be defined by the `cmd` field type. This is similar to how a method or message contain certain data shapes in object oriented or actor model languages respectively.

Using the JavaScript analogy from the introduction, an Action is similar to wrapping a call in a closure:

```js
// Command
{
  "cmd": "msg/send",
  "arg": {
    "from": "mailto:alice@example.com",
    "to": [ "bob@example.com", "carol@example.com" ],
    "subject": "hello",
    "body": "world"
  }
}
```

```js
// Pseudocode JS Analogy
() => msg.send({
  from: "mailto:alice@example.com",
  to: ["bob@example.com", "carol@example.com"],
  subject: "hello",
  body: "world"
})
```

### 3.1.1 Command

The Command (`cmd`) field MUST contain a concrete operation that can be sent to the Resource. This field can be thought of as the message or trait being sent to the resource. Note that _unlike_ a UCAN Delegation [Ability], which includes heirarchy, an Operation MUST be fully concrete.

### 3.1.2 Arguments

The Arguments (`arg`) field, MAY contain any parameters expected by the Resource/Operation pair, which MAY be different between different Resources and Operations, and is thus left to the executor to define the shape of this data. This field MUST be representable as a map or keyword list.

UCAN capabilities provided in proofs MAY impose certain constraint on the type of Arguments allowed.

### 3.1.3 Subject

The OPTIONAL `sub` field is intended for cases where parametrizing a specific agent is important. This is especially critical for two parts of the lifecycle:

1. Specifying a particular `sub` (and thus `aud`) when [enqueuing new Tasks][enqueue] in a Receipt
2. Indexing Receipts for reverse lookup and memoization

### 3.1.4 Nonce

If present, the REQUIRED `nnc` field MUST include a random nonce expressed in ASCII. This field ensures that multiple (non-idempotent) invocations are unique. The nonce SHOULD be `""` for Commands that are idempotent (such as determinstic Wasm modules or standards-abiding HTTP PUT requests).

## 3.2 Task

A Task wraps a [Command] with contextual information. This includes expiration time, delegation chain, and extensible metadata for things like resource limits.

| Field | Type                       | Description                                                                                  | Required |
|-------|----------------------------|----------------------------------------------------------------------------------------------|----------|
| `run` | `&Command`                 | The `run` field MUST contain a link to the [Task] to be run                                  | Yes      |
| `mta` | `{String : Any}`           | Extensible fields, e.g. resource limits, human-readable tags, notes, and so on               | Yes      |
| `prf` | `[&Delegation]`            | Links to any [UCAN Delegation]s that provide the authority to perform the enclosed [Command] | Yes      |
| `exp` | `Integer \| null`[^js-num] | The UTC Unix timestamp at which the Task expires                                             | Yes      |

The CID of a Task is useful for reverse lookups in [Receipt]-sharing networks to check if someone else has run this Task before, and in [UCAN Promise] to connect Tasks together.

## 3.3 Invocation

As [noted in the introduction][lazy-vs-eager], there is a difference between a reference to a function and calling that function. The [Invocation] is an instruction to the [Executor] to perform enclosed [Task]. [Invocation Payload]s are not executable until they have been signed and [Delegation] proofs validated.

### 3.3.1 Invocation Payload

The Invocation Payload attaches sender, receiver, and provenanial information to the [Task].
 
| Field | Type       | Required | Description                                               |
|-------|------------|----------|-----------------------------------------------------------|
| `iss` | `DID`      | Yes      | The DID of the [Invoker]                                  |
| `aud` | `DID`      | Yes      | The DID of the intended [Executor]                        |
| `run` | `&Task`    | Yes      | The enclosed [Task]'s CID                                 |
| `cau` | `&Receipt` | No       | An OPTIONAL CID of the [Receipt] that enqueued the [Task] |

### 3.3.2 Invocation (Envelope)
 
| Field | Type                 | Required | Description                                              |
|-------|----------------------|----------|----------------------------------------------------------|
| `uci` | `&InvocationPayload` | Yes      | The CID of the [Invocation Payload]                      |
| `sig` | `&Signature`         | Yes      | A signature by the Payload's `iss` over the CID in `uci` |

## 3.4 Examples

### 3.4.1 Interacting with an HTTP API

```js
{
  "sig": {"/": {bytes: "7aEDQIscUKVuAIB2Yj6jdX5ru9OcnQLxLutvHPjeMD3pbtHIoErFpo7OoC79Oe2ShgQMLbo2e6dvHh9scqHKEOmieA0"}},
  "inv": {                                                                    //           ┐
    "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",                                //           │
    "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",        //           │
    "run": cid({                                                              //  ┐        │
      "act": cid({                                 // ┐                           │        │
        "nnc": "&NCC-1701-D*",                     // │                           │        │
        "cmd": "crud/create",                      // │                           │        │
        "arg": {                                   // │                           │        │
          "uri": "https://example.com/blog/posts", // │                           │        │
          "headers": {                             // │                           │        │
            "content-type": "application/json"     // │                           │        │
          },                                       // ├── Action                  │        │
          "payload": {                             // │                           │        │
            "title": "UCAN for Fun an Profit",     // │                           │        │
            "body": "UCAN is great!",              // │                           ├── Task ├── Payload
            "topics": ["authz", "journal"],        // │                           │        │
            "draft": true                          // │                           │        │
          }                                        // │                           │        │
        }                                          // │                           │        │
      }),                                          // ┘                           │        │
      "mta": {                                                                //  │        │
        "env": "development",                                                 //  │        │
        "tags": ["blog", "post", "pr#123"]                                    //  │        │
      },                                                                      //  │        │
      "prf": [                                                                //  │        │
        {"/": "bafkr4iblvgvkmqt46imsmwqkjs7p6wmpswak2p5hlpagl2htiox272xyy4"}, //  │        │
        {"/": "bafkr4idnrqfouibxdqpvh2lmkhgsbw5yabvjbiaea3fplrb4vxifaphvgy"}, //  │        │
        {"/": "bafkr4ig4o5mwufavfewt4jurycn7g7dby2tcwg5q2ii2y6idnwguoyeruq"}, //  │        │
      ],                                                                      //  │        │
      "exp": 1697409438                                                       //  │        │
    })                                                                        //  ┘        │
  }                                                                           //           ┘
}
```

### 3.4.2 Sending Email

```js
{
  "sig": {"/": {bytes: "7aEDQIscUKVuAIB2Yj6jdX5ru9OcnQLxLutvHPjeMD3pbtHIoErFpo7OoC79Oe2ShgQMLbo2e6dvHh9scqHKEOmieA0"}},
  "inv": {
    "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
    "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
    "run": {
      "act": {
        "nnc": "email-akiko#1234567890"
        "cmd": "msg/send",
        "arg": {
          "from": "mailto:akiko@example.com",
          "to": [ "boris@example.com", "carol@example.com" ],
          "subject": "Coffee",
          "body": "Let get coffee sometime and talk about UCAN Invocations!"
        }
      },
      "mta": {},
      "prf": [{"/": "bafkr4iblvgvkmqt46imsmwqkjs7p6wmpswak2p5hlpagl2htiox272xyy4"}],
      "exp": 1697409438
    }
  }
}
```

### 3.4.3 Inline WebAssembly

```js
{
  "sig": {"/": {bytes: "7aEDQIscUKVuAIB2Yj6jdX5ru9OcnQLxLutvHPjeMD3pbtHIoErFpo7OoC79Oe2ShgQMLbo2e6dvHh9scqHKEOmieA0"}},
  "inv": {
    "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
    "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
    "run": {
      "act": {
        "nnc": "", // NOTE: as stated above, idempotent Actions should always have the same nonce
        "act": "wasm/run",
        "arg": {
          "mod": "data:application/wasm;base64,AHdhc21lci11bml2ZXJzYWwAAAAAAOAEAAAAAAAAAAD9e7+p/QMAkSAEABH9e8GowANf1uz///8UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP////8AAAAACAAAACoAAAAIAAAABAAAACsAAAAMAAAACAAAANz///8AAAAA1P///wMAAAAlAAAALAAAAAAAAAAUAAAA/Xu/qf0DAJHzDx/44wMBqvMDAqphAkC5YAA/1mACALnzB0H4/XvBqMADX9bU////LAAAAAAAAAAAAAAAAAAAAAAAAAAvVXNlcnMvZXhwZWRlL0Rlc2t0b3AvdGVzdC53YXQAAGFkZF9vbmUHAAAAAAAAAAAAAAAAYWRkX29uZV9mAAAADAAAAAAAAAABAAAAAAAAAAkAAADk////AAAAAPz///8BAAAA9f///wEAAAAAAAAAAQAAAB4AAACM////pP///wAAAACc////AQAAAAAAAAAAAAAAnP///wAAAAAAAAAAlP7//wAAAACM/v//iP///wAAAAABAAAAiP///6D///8BAAAAqP///wEAAACk////AAAAAJz///8AAAAAlP///wAAAACM////AAAAAIT///8AAAAAAAAAAAAAAAAAAAAAAAAAAET+//8BAAAAWP7//wEAAABY/v//AQAAAID+//8BAAAAxP7//wEAAADU/v//AAAAAMz+//8AAAAAxP7//wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAU////pP///wAAAAAAAQEBAQAAAAAAAACQ////AAAAAIj///8AAAAAAAAAAAAAAADQAQAAAAAAAA==",
          "fun": "add_one",
          "params": [42]
        }
      }
    }
  }
}
```

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

A `Receipt` is an attestation of the [Result] and requested [enqueued Tasks][enqueue]. A Receipt MUST be signed by the [Executor] (including by [Execution Proxy]).

**NB: a Receipt does not guarantee correctness of the result!** The statement's veracity MUST be only understood as an attestation from the executor.

Receipts MUST use the same-or-higher version number as the Invocation that they reference.

### 4.1.1 Receipt Payload

| Field | Type                      | Required | Description                                                                        |
|-------|---------------------------|----------|------------------------------------------------------------------------------------|
| `ucr` | `SemVer`                  | Yes      | The version of this spec that the Receipt conforms to                              |
| `iss` | `DID`                     | Yes      | The DID of the Executor                                                            |
| `ran` | `&Invocation`             | Yes      | MUST be a link to the [Invocation] that the Receipt is for                         |
| `prf` | `[&Delegation]`           | Yes      | [Delegation] proof chain if the Executor was not the `aud` of the `ran` Invocation |
| `out` | `Result`                  | Yes      | MUST contain the value output of the invocation in [Result] format                 |
| `enq` | `[&Task]`                 | Yes      | Further [Task]s that the [Invocation] would like to enqueue                        |
| `mta` | `{String : Any}`          | Yes      | Additional data about the receipt                                                  |
| `rec` | `&Receipt`                | No       | Recursive `Signed<Receipt>`s if the Invocation was proxied to another Executor     |
| `iat` | `Integer | null`[^js-num] | No       | The UTC Unix timestand at which the Receipt was issed                              |

A few of these fields warrant further comment below.

### 4.1.1.1 Enqueue

FIXME

The result of an [Invocation] MAY include a request for further actions to be performed via "effects". This enables several things: a clean separation of pure return values from requesting impure tasks to be performed by the runtime, and gives the runtime the control to decide how (or if!) more work should be performed.

Enqueued [Task]s describe requests for future work to be performed. The SHOULD come with [Delegation]s

All [Tasks]s in an [enqueue] array MUST be treated as concurrent, unless explicit data dependencies between them exist via promise [Pipeline]s.

### 4.1.1.2 Proxy Execution

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

## 4.2 Receipt Envelope

| Field | Type              | Required | Description                                                              |
|-------|-------------------|----------|--------------------------------------------------------------------------|
| `ucr` | `&ReceiptPayload` | Yes      | The CID for the [Receipt Payload]                                        |
| `sig` | `Signature`       | Yes      | The [Signature] of the `uci` value, by the [Receipt Payload]'s `iss` DID |

## 4.3 DAG-JSON Examples

``` js
{
  "ucr": {
    "ran": {"/": "bafyreia5tctxekbm5bmuf6tsvragyvjdiiceg5q6wghfjiqczcuevmdqcu"}
    "out": {
      "ok": ["bob@example.com", "alice@example.com"]
    },
    "mta": {
      "retry-count": 2,
      "total-time": [400, "hours"]
    }
  },
  "sig": {"/": {bytes: "7aEDQLYvb3lygk9yvAbk0OZD0q+iF9c3+wpZC4YlFThkiNShcVriobPFr/wl3akjM18VvIv/Zw2LtA4uUmB5m8PWEAU"}}
}
```

# 5 Prior Art

[ucanto RPC] from [DAG House] is a production system that uses UCAN as the basis for an RPC layer.

The [Capability Transport Protocol (CapTP)] is one of the most influential object-capability systems, and forms the basis for much of the rest of the items on this list.

The Object Capability Network ([OCapN]) protocol extends [CapTP] with a generalized networking layer. It has implementations from the [Spritely Institute] and [Agoric]. At time of writing, it is in the process of being standardized.

[Electronic Rights Transfer Protocol (ERTP)] builds on top of [CapTP] concepts for blockchain & digital asset use cases.

[Cap 'n Proto RPC] is an influential RPC framework based on concepts from [CapTP].

# 6 Acknowledgements

Many thanks to [Mark Miller] for his [trail blazing work][erights] on [capability systems].

Many thanks to [Luke Marsen] and [Simon Worthington] for their feedback on invocation model from their work on [Bacalhau] and [IPVM].

Thanks to [Marc-Antoine Parent] for his discussions of the distinction between declarations and directives both in and out of a UCAN context.

Many thanks to [Quinn Wilton] for her discussion of speech acts, the dangers of signing canonicalized data, and ergonomics.

Thanks to [Blaine Cook] for sharing their experiences with OAuth 1, irreversible design decisions, and advocating for keeping the spec simple-but-evolvable.

Thanks to [Philipp Krüger] for the enthusiastic feedback on the overall design and encoding.

Thanks to [Christine Lemmer-Webber] for the many conversations about capability systems and the programming models that they enable.

Thanks to [Rod Vagg] for the clarifications on IPLD Schema implicits and the general IPLD worldview

<!-- Footnotes -->

[^js-num]: JavaScript has a single numeric type ([`Number`][JS Number]) for both integers and floats. This representation is defined as a [IEEE-754] double-precision floating point number, which has a 53-bit significand.
 
<!-- Internal Links -->

[Arguments]: #312-arguments
[Command]: #311-command
[Execution Proxy]: #41112-proxy-execution
[Executor]: #212-executor
[Invocation Payload]: #331-invocation-payload
[Invocation Signature]: #331-invocation-envelope
[Invocation]: #33-invocation
[Invoker]: #211-invoker
[Nonce]: #314-nonce
[Receipt Payload]: #411-receipt-payload
[Receipt]: #42-receipt
[Response]: #4-response
[Result]: #41-result
[Subject]: #313-subject
[Task]: #32-task
[enqueue]: #4111-enqueue
[lazy-vs-eager]: #112-lazy-vs-eager-evaluation

<!-- External Links -->

[Ability]: https://github.com/ucan-wg/delegation#33-abilities
[Agoric]: https://agoric.com/
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Bacalhau]: https://www.bacalhau.org/
[Blaine Cook]: https://github.com/blaine
[Brooklyn Zelenka]: https://github.com/expede/
[CapTP]: https://capnproto.org/rpc.html#specification
[Christine Lemmer-Webber]: https://github.com/cwebber
[DAG House]: https://dag.house
[Delegation]: https://github.com/ucan-wg/delegation
[E-lang Mailing List, 2000 Oct 18]: http://wiki.erights.org/wiki/Capability-based_Active_Invocation_Certificates
[Electronic Rights Transfer Protocol (ERTP)]: https://docs.agoric.com/guides/ertp/
[Fission]: https://fission.codes/
[Haskell]: https://en.wikipedia.org/wiki/Haskell
[IPVM]: https://github.com/ipvm-wg
[Irakli Gozalishvili]: https://github.com/Gozala
[JS Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
[Luke Marsen]: https://github.com/lukemarsden
[Marc-Antoine Parent]: https://github.com/maparent
[Mark Miller]: https://github.com/erights
[OCapN]: https://github.com/ocapn/
[Philipp Krüger]: https://github.com/matheus23/
[Quinn Wilton]: https://github.com/QuinnWilton
[Robust Composition]: http://www.erights.org/talks/thesis/markm-thesis.pdf
[Rod Vagg]: https://github.com/rvagg/
[Simon Worthington]: https://github.com/simonwo
[Spritely Institute]: https://spritely.institute/news/introducing-a-distributed-debugger-for-goblins-with-time-travel.html
[UCAN Ability]: https://github.com/ucan-wg/delegation/#23-ability
[UCAN Delegation]: https://github.com/ucan-wg/delegation/
[UCAN Promise]: https://github.com/ucan-wg/promise/
[URI]: https://en.wikipedia.org/wiki/Uniform_Resource_Identifier
[Zeeshan Lakhani]: https://github.com/zeeshanlakhani
[`data`]: https://en.wikipedia.org/wiki/Data_URI_scheme
[`ipfs`]: https://docs.ipfs.tech/how-to/address-ipfs-on-web/#native-urls
[`magnet`]: https://en.wikipedia.org/wiki/Magnet_URI_scheme
[capability systems]: https://en.wikipedia.org/wiki/Capability-based_security
[distributed promise pipelines]: http://erights.org/elib/distrib/pipeline.html
[eRights]: https:/erights.org
[erights]: https://erights.org
[ucanto RPC]: https://github.com/web3-storage/ucanto
