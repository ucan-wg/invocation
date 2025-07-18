# UCAN Invocation Specification
## Version 1.0.0-rc. 1

## Editors
[Editors]: #editors

- [Brooklyn Zelenka], [Witchcraft Software]
- [Irakli Gozalishvili], [Protocol Labs]

## Authors
[Authors]: #authors

- [Brooklyn Zelenka], [Witchcraft Software]
- [Irakli Gozalishvili], [Protocol Labs]
- [Zeeshan Lakhani], [Oxide Computer]
- [Hugo Dias], [Decentralised Experience]

# Dependencies
[Dependencies]: #dependencies

- [UCAN Delegation]

## Language
[Language]: #language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] when, and only when, they appear in all capitals, as shown here.

# Abstract
[Abstract]: #abstract

UCAN Invocation defines a format for expressing the intention to execute delegated UCAN capabilities, and the attested receipts from an execution.

# Introduction
[Introduction]: #introduction

> Just because you can doesn't mean that you should
>
> — Anonymous

> When authorization is communicated without such context, it's like receiving a key in the mail with no hint about what to do with it [...] After an object receives this message, she can invoke arg if she chooses, but why would she ever choose to do so?
>
> [Mark Miller], [E-lang Mailing List, 2000 Oct 18]

UCAN is a chained-capability format. A UCAN contains all of the information that one would need to perform some task, and the provable authority to do so. This begs the question: can UCAN be used directly as an RPC language?

Some teams have had success with UCAN directly for RPC when the intention is clear from context. This can be successful when there is more information on the channel than the UCAN itself (such as an HTTP path that a UCAN is sent to). However, capability invocation contains strictly more information than delegation: all of the authority of UCAN, plus the command to perform the task.

## Intuition
[Intuition]: #intuition

### Car Keys
[Car Keys]: #car-keys

Consider the following fictitious scenario:

Akiko is going away for the weekend. Her good friend Boris is going to borrow her car while she's away. They meet at a nearby cafe, and Akiko hands Boris her car keys. Boris now has the capability to drive Akiko's car whenever he wants to. Depending on their plans for the rest of the day, Akiko may find Boris quite rude if he immediately leaves the cafe to go for a drive. On the other hand, if Akiko asks Boris to run some last minute pre-vacation errands for that require a car, she may expect Boris to immediately drive off.

To put this in terms closer to a UCAN flow:

``` mermaid
sequenceDiagram
    participant 🚗
    actor Akiko
    actor Boris

    autonumber

    Note over 🚗, Akiko: Akiko buys a car
    🚗 -->> Akiko: Delegate(Drive 🚗)

    Note over Akiko, Boris: Boris offers to run errands for Akiko
    Boris -->> Akiko: Delegate(Boris to run errands)

    Note over Akiko, Boris: Akiko gives Boris access to her car
    Akiko -->> Boris: Delegate(Drive 🚗)

    Note over 🚗, Boris: Akiko asks Boris to use her car to run errands
    Akiko ->> Boris: Invoke!(Boris to run errands, using 🚗 (➌))
    Boris ->> 🚗: Invoke!(Drive 🚗)
```

In the example above, steps ➌ and ➍ are qualitatively different:

- Step ➌ grants authority (to drive the car)
- Step ➍ is a _command_ to do so

## Lazy vs Eager Evaluation
[Lazy vs Eager Evaluation]: #lazy-vs-eager-evaluation

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

## Public Resources
[Public Resources]: #public-resources

A core part of UCAN's design is interacting with the wider, non-UCAN world. Many resources are open to anyone to access, such as unauthenticated web endpoints. Unlike UCAN-controlled resources, an invocation on public resources is both possible, and a hard requirement for initiating a flow (e.g. sign up). These cases typically involve a reference passed out of band (such as a web link). Due to [designation with authorization], knowing the URI of a public resource is often sufficient for interacting with it. In these cases, the Executor MAY accept Invocations without having a "closed-loop" proof chain, but this SHOULD NOT be the default behavior.

## Promise Pipelining
[Promise Pipelining]: #promise-pipelining

[UCAN Promise] extends UCAN Invocation with [distributed promise pipelines]. Promises are helpful in a wide variety of situations for efficiency and convenience. Implementations supporting UCAN Promises is RECOMMENDED.

# Concepts
[Concepts]: #concepts

## Roles
[Roles]: #roles

Task adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delegate principals MUST persist to the invocation.

| UCAN Field | Delegation                             | Invocation                      |
|------------|----------------------------------------|---------------------------------|
| `iss`      | Delegator: transfer authority (active) | Invoker: request task (active)  |
| `aud`      | Delegate: gain authority (passive)     | Executor: perform task (active) |

### Invoker
[Invoker]: #invoker

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

### Executor
[Executor]: #executor

The executor is directed to perform some task described in the UCAN invocation by the invoker.

## Life Cycle
[Life Cycle]: #life-cycle

At a very high level:

- A [Task] abstractly describes some Action to be run
- An Invocation attaches proven ([delegated][Delegation]) authority to a [Task], and requests it be run by a certain Agent
- A [Receipt] MAY request that the Invoker enqueue more [Task]s

``` mermaid
erDiagram
    Delegation }o--|{ Invocation: proves
    Invocation }|--|| Task: requests
    Invocation ||--|| Receipt: returns
    Receipt |o--|{ Task: enqueues
```

## Anatomy
[Anatomy]: #anatomy

| Concept      | Description                                                                 |
|--------------|-----------------------------------------------------------------------------|
| [Command]    | Function application; a description of work to be performed                 |
| [Task]       | Contextual information for a [Command], such as resource limits             |
| Invocation   | A request to perform some [Task] based on [delegated][Delegation] authority |

A request for some work to be done (or to "exercise your authority") is an Invocation.

``` mermaid
flowchart TD
    subgraph Invocation
        SignatureBytes["Signature (raw bytes)"]
      
        subgraph SigPayload ["Signature Payload"]
            VarsigHeader["Varsig Header"]

            subgraph InvocationPayload ["Invocation Payload"]
                iss
                sub
                do
                args
                prf
                cause["cause (optional)"]
                etc["..."]
            end
        end
    end
    
    cause -.->|CID| Receipt
```

As [noted in the introduction][Lazy vs Eager Evaluation], there is a difference between a reference to a function and calling that function. The Invocation is a request to the [Executor] to perform the enclosed [Task]. [Invocation Payload]s are not executable until they have been signed and [Delegation] proofs validated.

Note that the Invocation MUST include the Signature envelope. An [Invocation Payload] on its own MUST NOT be considered a valid Invocation.

# [UCAN Envelope] Configuration
[UCAN Envelope Configuration]: #ucan-envelope-configuration

## Type Tag
[Type Tag]: #type-tag
  
The UCAN envelope's [payload tag] MUST be `ucan/inv@1.0.0-rc.1`.

## Invocation Payload
[Invocation Payload]: #invocation-payload

The Invocation Payload attaches sender, receiver, and provenance to the [Task].
 
| Field   | Type                       | Required | Description                                                        |
|---------|----------------------------|----------|--------------------------------------------------------------------|
| `iss`   | `DID`                      | Yes      | The DID of the [Invoker]                                           |
| `sub`   | `DID`                      | Yes      | The [Subject] being invoked                                        |
| `aud`   | `DID`                      | No       | The DID of the intended [Executor] if different from the [Subject] |
| `cmd`   | `String`                   | Yes      | The [Command]                                                      |
| `args`  | `{String : Any}`           | Yes      | The [Command]'s [Arguments]                                        |
| `prf`   | `[&Delegation]`            | Yes      | [Delegation]s that prove the chain of authority                    |
| `meta`  | `{String : Any}`           | No       | Arbitrary [Metadata]                                               |
| `nonce` | `Bytes`                    | Yes       | A unique, random nonce                                             |
| `exp`   | `Integer \| null`[^js-num] | Yes      | The timestamp at which the Invocation becomes invalid              |
| `iat`   | `Integer`[^js-num]         | No       | The timestamp at which the Invocation was created                  |
| `cause` | `&Receipt`                 | No       | An OPTIONAL CID of the [Receipt] that enqueued the [Task]          |
 
[^js-num]: JavaScript has a single numeric type ([`Number`][JS Number]) for both integers and floats. This representation is defined as a [IEEE-754] double-precision floating point number, which has a 53-bit significand.

The shape of the `args` MUST be defined by the `cmd` field type. This is similar to how a method or message contain certain data shapes in object oriented or actor model languages respectively. Using the JavaScript analogy from the introduction, an Action is similar to wrapping a call in a closure:

```js
// Command
{
  "cmd": "/msg/send",
  "args": {
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

### Agents
[Agents]: #agents

#### Issuer
[Issuer]: #issuer

The `iss` field MUST include the Issuer of the Invocation. This DID URL MUST dereference to the public key which proves the signature over the payload included in the envelope.

#### Subject
[Subject]: #subject

The REQUIRED `sub` field both parameterizes over a specific agent, and acts as a namespace for how to interpret the [Command]. This is especially critical for two parts of the life cycle:

1. Specifying a particular `sub` (and thus `aud`) when [enqueuing new Tasks][enqueue] in a Receipt
2. Indexing Receipts for reverse lookup and memoization

#### Audience
[Audience]: #audience

The OPTIONAL `aud` field specified the intended recipient of Invocation, otherwise the Audience MUST be assumed to the [Subject]. This is useful for message routing, command brokers, proxy execution, gateways, replicated state machines, and so on.

### Task
[Task]: #task

A Task is the subset of Invocation fields that uniquely determine the work to be performed. The nonce is important for distinguishing between non-idempotent executions of a Task by making the group together unique.
  
A Task MUST be uniquely defined by a Task ID that is the CID of the following fields as a keyed map:

- [Subject]
- [Command]
- [Arguments]
- [Nonce]

Tasks that describe pure functions — or other strategies like fan-out racing — SHOULD have the same Task ID by using the same nonce.

#### Command
[Command]: #command

The REQUIRED Command (`cmd`) field MUST contain a concrete, dispatchable message that can be sent to the Executor. The Command MUST define the shape of the data in the [Arguments].

### Arguments
[Arguments]: #arguments

The REQUIRED Arguments (`args`) field, MAY contain any parameters expected by the Command. The Subject MUST be considered the authority on the shape of this data. This field MUST be representable as a map or keyword list.

The Arguments MUST pass validation of the Policies on all of the [UCAN Delegations][UCAN Delegation] in the [Proofs] field. If any [Policy] reports failure against the Invocation's Arguments, the Invocation MUST be rejected.

#### Nonce
[Nonce]: #nonce

The REQUIRED `nonce` field MUST include a random nonce. This field ensures that multiple (non-idempotent) invocations are unique. The nonce SHOULD be empty (`0x`) for Commands that are idempotent (such as deterministic Wasm modules or standards-abiding HTTP PUT requests).

### Proofs
[Proofs]: #proofs

The `prf` field lists the path of authority from the [Subject] to the [Invoker]. This MUST be an array of CIDs pointing [Delegations][Delegation] starting from the root Delegation (issued by the Subject), in strict sequence where the `aud` of the previous Delegation matches the `iss` of the next Delegation.

See [Proof Chains] for more detail

#### Cause
[Cause]: #cause

The OPTIONAL `cause` field is a provenance claim describing which [Receipt] requested it. This is helpful for tracking chains of Invocations.

#### Expiration
[Expiration]: #expiration

The REQUIRED nullable field `exp` defines when the Invocation SHOULD time out. Setting a timeout within a a few minutes is RECOMMENDED as it accounts for clock skew but limits the ability of an attacker to take advantage of an intercepted Invocation. In general, the smaller the time window the better. This is both expressive (defines a timeout, which is a best practice), and prevents replays.

#### Issued At
[Issued At]: #issued-at

The OPTIONAL `iat` field MAY contain an issuance timestamp. This time SHOULD NOT be trusted; it is only a claim by the Invoker of their system time. System clocks often have clock skew, or a Byzantine Invoker could claim an arbitrary time.

#### Metadata
[Metadata]: #metadata

The OPTIONAL `meta` field MAY include arbitrary metadata or extensible fields. For example, Wasm fuel, an internal job ID, references to GitHub Issues, and so on. This data MAY be used by the Executor.

## Attestation
[Attestation]: #attestation

An Invocation MAY be used to attest to some information. This is in effect a statement to the Issuer (without Audience) that never expires.

## Proof Chains
[Proof Chains]: #proof-chains

A Task MUST include the entire [UCAN Delegation] proof chain in the `prf` field. The chain MUST form a direct line of authority, starting with the delegation with an `aud` that matches the Invoker, and ending with a delegation where the `iss` matches the `sub`. The `sub` throughout MUST match the `aud` of the Invocation.

``` mermaid
flowchart RL
    invoker((&nbsp&nbsp&nbsp&nbspDan&nbsp&nbsp&nbsp&nbsp))
    subject((&nbsp&nbsp&nbsp&nbspAlice&nbsp&nbsp&nbsp&nbsp))

    subject -- controls --> resource[(Storage)]
    rootCap -- references --> resource

    subgraph Delegations
        subgraph root [Root UCAN]
            subgraph rooting [Root Issuer]
                rootIss(iss: Alice)
                rootSub(sub: Alice)
            end

            rootCap("(Storage, crud/*)")
            rootAud(aud: Bob)
        end

        subgraph del1 [Delegated UCAN]
            del1Iss(iss: Bob) --> rootAud
            del1Sub(sub: Alice)
            del1Aud(aud: Carol)
            del1Cap("(Storage, crud/*)") --> rootCap

            del1Sub --> rootSub
        end

        subgraph del2 [Delegated UCAN]
            del2Iss(iss: Carol) --> del1Aud
            del2Sub(sub: Alice)
            del2Aud(aud: Dan)
            del2Cap("(Storage, crud/*)") --> del1Cap

            del2Sub --> del1Sub
        end
    end

     subgraph inv [Invocation]
        invIss(iss: Dan)
        args("args: [Storage, crud/update, (key, value)]")
        invSub(aud: Alice)
        prf("proofs")
    end

    invIss --> del2Aud
    invoker --> invIss
    args --> del2Cap
    invSub --> del2Sub
    rootIss --> subject
    rootSub --> subject
    prf --> Delegations
```

## Examples
[Examples]: #examples

### Interacting with an HTTP API

```js
// DAG-JSON
[
  {"/": {"bytes": "bdNVZn+uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT+SRK6v/SX8bjt+VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"}},
  {
    "h": {"/": {"bytes": "NAHtAe0BE3E"}},
    "ucan/i/1.0.0-rc.1": {
      "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
      "sub": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "cmd": "/crud/create",
      "args": {
        "uri": "https://example.com/blog/posts",
        "headers": {
          "content-type": "application/json"
        },
        "payload": {
          "title": "UCAN for Fun an Profit",
          "body": "UCAN is great!",
          "topics": ["authz", "journal"],
          "draft": true
        }
      },
      "nonce": {"/": {"bytes": "TWFueSBopvcs"}},
      "meta": {
        "env": "development",
        "tags": ["blog", "post", "pr#123"]
      },
      "exp": 1697409438
      "prf": [
        {"/": "zdpuAzx4sBrBCabrZZqXgvK3NDzh7Mf5mKbG11aBkkMCdLtCp"},
        {"/": "zdpuApTCXfoKh2sB1KaUaVSGofCBNPUnXoBb6WiCeitXEibZy"},
        {"/": "zdpuAoFdXRPw4n6TLcncoDhq1Mr6FGbpjAiEtqSBrTSaYMKkf"}
      ]
    }
    
  }
]
```

### Sending Email

```js
// DAG-JSON
[
  {"/": {"bytes": "bdNVZn+uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT+SRK6v/SX8bjt+VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"}},
  {
    "h": {"/": {"bytes": "NAHtAe0BE3E"}},
    "ucan/i/1.0.0-rc.1": {
      "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
      "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "sub": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "cmd": "/msg/send",
      "args": {
        "from": "mailto:akiko@example.com",
        "to": [ "boris@example.com", "carol@example.com" ],
        "subject": "Coffee",
        "body": "Let get coffee sometime and talk about UCAN Invocations!"
      },
      "nonce": {"/": {"bytes": "TWFueSBopZ2h0IHdvcs"}},
      "prf": [{"/": "zdpuAzx4sBrBCabrZZqXgvK3NDzh7Mf5mKbG11aBkkMCdLtCp"}],
      "exp": 1697409438
    }
  }
]
```

### Inline WebAssembly

```js
[
  {"/": {"bytes": "bdNVZn+uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT+SRK6v/SX8bjt+VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"}},
  {
    "h": {"/": {"bytes": "NAHtAe0BE3E"}},
    "ucan/i/1.0.0-rc.1": {
      "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
      "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "sub": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "meta": {"fuel": 999999},
      "nonce": {"/": {"bytes": ""}}, // NOTE: as stated above, idempotent Actions should always have the same nonce
      "cmd": "/wasm/run",
      "args": {
        "mod": "data:application/wasm;base64,AHdhc21lci11bml2ZXJzYWwAAAAAAOAEAAAAAAAAAAD9e7+p/QMAkSAEABH9e8GowANf1uz///8UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP////8AAAAACAAAACoAAAAIAAAABAAAACsAAAAMAAAACAAAANz///8AAAAA1P///wMAAAAlAAAALAAAAAAAAAAUAAAA/Xu/qf0DAJHzDx/44wMBqvMDAqphAkC5YAA/1mACALnzB0H4/XvBqMADX9bU////LAAAAAAAAAAAAAAAAAAAAAAAAAAvVXNlcnMvZXhwZWRlL0Rlc2t0b3AvdGVzdC53YXQAAGFkZF9vbmUHAAAAAAAAAAAAAAAAYWRkX29uZV9mAAAADAAAAAAAAAABAAAAAAAAAAkAAADk////AAAAAPz///8BAAAA9f///wEAAAAAAAAAAQAAAB4AAACM////pP///wAAAACc////AQAAAAAAAAAAAAAAnP///wAAAAAAAAAAlP7//wAAAACM/v//iP///wAAAAABAAAAiP///6D///8BAAAAqP///wEAAACk////AAAAAJz///8AAAAAlP///wAAAACM////AAAAAIT///8AAAAAAAAAAAAAAAAAAAAAAAAAAET+//8BAAAAWP7//wEAAABY/v//AQAAAID+//8BAAAAxP7//wEAAADU/v//AAAAAMz+//8AAAAAxP7//wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAU////pP///wAAAAAAAQEBAQAAAAAAAACQ////AAAAAIj///8AAAAAAAAAAAAAAADQAQAAAAAAAA==",
        "fun": "add_one",
        "params": [42]
      }
    }
  }
]
```

# Prior Art
[Prior Art]: #prior-art

[ucanto RPC] from [Storacha] is a production system that uses UCAN as the basis for an RPC layer.

The Capability Transport Protocol ([CapTP]) is one of the most influential object-capability systems, and forms the basis for much of the rest of the items on this list.

The Object Capability Network ([OCapN]) protocol extends [CapTP] with a generalized networking layer. It has implementations from the [Spritely Institute] and [Agoric]. At time of writing, it is in the process of being standardized.

[Electronic Rights Transfer Protocol (ERTP)] builds on top of [CapTP] concepts for blockchain & digital asset use cases.

[Cap 'n Proto RPC] is an influential RPC framework based on concepts from [CapTP].

# 7 Acknowledgements
[Acknowledgements]: #acknowledgements

Many thanks to [Mark Miller] for his [trail blazing work][eRights] on [capability systems].

Many thanks to [Luke Marsen] and [Simon Worthington] for their feedback on invocation model from their work on [Bacalhau] and [IPVM].

Thanks to [Marc-Antoine Parent] for his discussions of the distinction between declarations and directives both in and out of a UCAN context.

Many thanks to [Quinn Wilton] for her discussion of speech acts, the dangers of signing canonicalized data, and ergonomics.

Thanks to [Blaine Cook] for sharing their experiences with [OAuth 1], irreversible design decisions, and advocating for keeping the spec simple-but-evolvable.

Thanks to [Philipp Krüger] for the enthusiastic feedback on the overall design and encoding.

Thanks to [Christine Lemmer-Webber] for the many conversations about capability systems and the programming models that they enable.

Thanks to [Rod Vagg] for the clarifications on IPLD Schema implicits and the general IPLD worldview

Many thanks to [Juan Caballero] for his detailed questions and comments to help polish the spec.
  
<!-- External Links -->

[Agoric]: https://agoric.com/
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Bacalhau]: https://www.bacalhau.org/
[Blaine Cook]: https://github.com/blaine
[Brooklyn Zelenka]: https://github.com/expede/
[Cap 'n Proto RPC]: https://capnproto.org/
[CapTP]: http://erights.org/elib/distrib/captp/index.html
[Christine Lemmer-Webber]: https://github.com/cwebber
[Storacha]: https://github.com/storacha/ucanto
[DAG-CBOR]: https://ipld.io/specs/codecs/dag-cbor/spec/
[DID]: https://www.w3.org/TR/did-core/
[Delegation]: https://github.com/ucan-wg/delegation
[Electronic Rights Transfer Protocol (ERTP)]: https://docs.agoric.com/guides/ertp/
[Fission]: https://fission.codes/
[Haskell]: https://en.wikipedia.org/wiki/Haskell
[IEEE-754]: https://ieeexplore.ieee.org/document/8766229
[IPLD]: https://ipld.io/
[IPVM]: https://github.com/ipvm-wg
[Irakli Gozalishvili]: https://github.com/Gozala
[JS Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
[Juan Caballero]: https://github.com/bumblefudge
[Luke Marsen]: https://github.com/lukemarsden
[Marc-Antoine Parent]: https://github.com/maparent
[Mark Miller]: https://github.com/erights
[OAuth 1]: https://oauth.net/1/
[OCapN]: https://github.com/ocapn/
[Oxide Computer]: https://oxide.computer/ 
[Philipp Krüger]: https://github.com/matheus23/
[Policy]: https://github.com/ucan-wg/delegation#policy
[Protocol Labs]: https://protocol.ai
[Quinn Wilton]: https://github.com/QuinnWilton
[Receipt]: https://github.com/ucan-wg/receipt
[Robust Composition]: http://www.erights.org/talks/thesis/markm-thesis.pdf
[Rod Vagg]: https://github.com/rvagg/
[Simon Worthington]: https://github.com/simonwo
[Spritely Institute]: https://spritely.institute/news/introducing-a-distributed-debugger-for-goblins-with-time-travel.html
[UCAN Delegation]: https://github.com/ucan-wg/delegation/
[UCAN Envelope]: https://github.com/ucan-wg/spec/blob/main/README.md#envelope
[UCAN Promise]: https://github.com/ucan-wg/promise/
[URI]: https://en.wikipedia.org/wiki/Uniform_Resource_Identifier
[Witchcraft Software]: https://github.com/expede
[Zeeshan Lakhani]: https://github.com/zeeshanlakhani
[`data`]: https://en.wikipedia.org/wiki/Data_URI_scheme
[`ipfs`]: https://docs.ipfs.tech/how-to/address-ipfs-on-web/#native-urls
[`magnet`]: https://en.wikipedia.org/wiki/Magnet_URI_scheme
[capability systems]: https://en.wikipedia.org/wiki/Capability-based_security
[designation with authorization]: https://srl.cs.jhu.edu/pubs/SRL2003-02.pdf
[distributed promise pipelines]: http://erights.org/elib/distrib/pipeline.html
[eRights]: https://erights.org
[payload tag]: https://github.com/ucan-wg/spec/blob/main/README.md#envelope
[principle of least authority]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[ucanto RPC]: https://github.com/web3-storage/ucanto
[Decentralised Experience]: https://decentralised.dev
