# UCAN Invocation Specification v1.0.0-rc. 1

## Editors

- [Brooklyn Zelenka], [Fission]
- [Irakli Gozalishvili], [DAG House]

## Authors

- [Brooklyn Zelenka], [Fission]
- [Irakli Gozalishvili], [DAG House]
- [Zeeshan Lakhani], [Fission]

## Depends On

- [IPLD]
- [DID]
- [UCAN Delegation]

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14] when, and only when, they appear in all capitals, as shown here.

# 0 Abstract

UCAN Invocation defines a format for expressing the intention to execute delegated UCAN capabilities, and the attested receipts from an execution.

# 1 Introduction

> Just because you can doesn't mean that you should
>
> — Anonymous

> When authorization is communicated without such context, it's like receiving a key in the mail with no hint about what to do with it [...] After an object receives this message, she can invoke arg if she chooses, but why would she ever choose to do so?
>
> Mark Miller, [E-lang Mailing List, 2000 Oct 18]

UCAN is a chained-capability format. A UCAN contains all of the information that one would need to perform some task, and the provable authority to do so. This begs the question: can UCAN be used directly as an RPC language?

Some teams have had success with UCAN directly for RPC when the intention is clear from context. This can be successful when there is more information on the channel than the UCAN itself (such as an HTTP path that a UCAN is sent to). However, capability invocation contains strictly more information than delegation: all of the authority of UCAN, plus the command to perform the task.

## 1.1 Intuition

## 1.1.1 Car Keys

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

## 1.2 Public Resources

A core part of UCAN's design is interacting with the wider, non-UCAN world. Many resources are open to anyone to access, such as unauthenticated web endpoints. Unlike UCAN-controlled resources, an invocation on public resources is both possible, and a hard requirement for initiating a flow (e.g. sign up). These cases typically involve a reference passed out of band (such as a web link). Due to [designation without authorization], knowing the URI of a public resource is often sufficient for interacting with it. In these cases, the Executor MAY accept Invocations without having a "closed-loop" proof chain, but this SHOULD NOT be the default behavior.

## 1.3 Promise Pipelining

[UCAN Promise] extends UCAN Invocation with [distributed promise pipelines]. Promises are helpful in a wide variety of situations for efficiency and convenience. Implementing UCAN Promises is RECOMMENDED.

## 1.4 Serialization

UCAN Invocations MUST be encoded with some [IPLD] codec. [DAG-CBOR] is RECOMMENDED.

# 2 Concepts

## 2.1 Roles

Task adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delegate principals MUST persist to the invocation.

| UCAN Field | Delegation                             | Invocation                      |
|------------|----------------------------------------|---------------------------------|
| `iss`      | Delegator: transfer authority (active) | Invoker: request task (active)  |
| `aud`      | Delegate: gain authority (passive)     | Executor: perform task (active) |

### 2.1.1 Invoker

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

### 2.1.2 Executor

The executor is directed to perform some task described in the UCAN invocation by the invoker.

## 2.2 Life Cycle

At a very high level:

- A [Task] abstractly describes some Action to be run
- An [Invocation] attaches proven ([delegated][Delegation]) authority to a [Task], and requests it be run by a certain Agent
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
                etc["..."]
            end
        end
    end
```

As [noted in the introduction][lazy-vs-eager], there is a difference between a reference to a function and calling that function. The [Invocation] is a request to the [Executor] to perform the enclosed [Task]. [Invocation Payload]s are not executable until they have been signed and [Delegation] proofs validated.

Note that the Invocation MUST include the Signature envelope. An [Invocation Payload] on its own MUST NOT be considered a valid Invocation.

## 3.1 Invocation (Envelope)
 
| Field | Type               | Required | Description                                           |
|-------|--------------------|----------|-------------------------------------------------------|
| `s`   | `Signature`        | Yes      | A signature by the Payload's `iss` over the `p` field |
| `p`   | `SignaturePayload` | Yes      | The content that was signed                           |

## 3.2 Signature Payload

| Field               | Type                | Required | Description              |
|---------------------|---------------------|----------|--------------------------|
| `h`                 | `VarsigHeader`      | Yes      | The Varsig header        |
| `ucan/i/1.0.0-rc.1` | `InvocationPayload` | Yes      | The [Invocation Payload] |

The Signature Payload MUST contain a [Varsig] header, and the [Invocation Payload]. The Varsig header MUST describe the cryptographic configuration used to format and sign the [Invocaion Payload]. This is important in order to:

1. Commits the Signature to the cryptographic algorithms used
2. Describes how the paylaod was serialized before signing

## 3.3 Invocation Payload

The Invocation Payload attaches sender, receiver, and provenance to the [Task].
 
| Field       | Type    | Required           | Description |                                                                    |
|-------------|---------|--------------------|-------------|--------------------------------------------------------------------|
| [Issuer]    | `iss`   | `DID`              | Yes         | The DID of the [Invoker]                                           |
| [Subject]   | `sub`   | `DID`              | Yes         | The [Subject] being invoked                                        |
| [Audience]  | `aud`   | `DID`              | No          | The DID of the intended [Executor] if different from the [Subject] |
| [Command]   | `do`    | `String`           | Yes         | The [Command]                                                      |
| [Arguments] | `args`  | `{String : Any}`   | Yes         | The [Command]'s [Arguments]                                        |
| [Proofs]    | `prf`   | `[&Delegation]`    | Yes         | [Delegation]s that prove the chain of authority                    |
| [Metadata]  | `meta`  | `{String : Any}`   | No          | Arbitrary [Metadata]                                               |
| [Nonce]     | `nonce` | `Bytes`            | No          | A unique, random nonce                                             |
| [Expiry]    | `exp`   | `Integer`[^js-num] | No          | The timestamp at which the Invocation becomes invalid              |
| [Issued At] | `iat`   | `Integer`[^js-num] | No          | The timestamp at which the Invocation was created                  |
| [Cause]     | `cause` | `&Receipt`         | No          | An OPTIONAL CID of the [Receipt] that enqueued the [Task]          |

The shape of the `args` MUST be defined by the `do` field type. This is similar to how a method or message contain certain data shapes in object oriented or actor model languages respectively. Using the JavaScript analogy from the introduction, an Action is similar to wrapping a call in a closure:

```js
// Command
{
  "do": "msg/send",
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

### 3.1.1 Command

The REQUIRED Command (`do`) field MUST contain a concrete, dispatchable message that can be sent to the Executor. The Command MUST define the shape of the data in the [Arguments].

### 3.1.2 Arguments

The REQUIRED Arguments (`args`) field, MAY contain any parameters expected by the Command. The Subject MUST be considered the authority on the shape of this data. This field MUST be representable as a map or keyword list.

UCAN capabilities provided in proofs MAY impose constraints on the type of Arguments allowed.

### 3.1.3 Subject

The REQUIRED `sub` field both parameterizes over a specific agent, and acts as a namespace for how to interpret the [Command]. This is especially critical for two parts of the life cycle:

1. Specifying a particular `sub` (and thus `aud`) when [enqueuing new Tasks][enqueue] in a Receipt
2. Indexing Receipts for reverse lookup and memoization

### 3.1.4 Nonce

The REQUIRED `nonce` field MUST include a random nonce. This field ensures that multiple (non-idempotent) invocations are unique. The nonce SHOULD be empty (`0x`) for Commands that are idempotent (such as deterministic Wasm modules or standards-abiding HTTP PUT requests).

### 3.1.5 Metadata

The `meta` field MAY include arbitrary metadata or extensable fields. For example, Wasm fuel, an internal job ID, references to GitHub Issues, and so on. This data MAY be used by the Executor.

### 3.1.6 Proofs

The `prf` field defines all [Delegation]s required to prove that this Invocation has an unbroken authorization chain.

### 3.1.7 Expiry

The OPTIONAL field `exp` defines when the Invocation SHOULD time out. This is both expressive (defines a timeout, which is a best practice), and prevents replays.

### 3.1.8 Issued At

The `iat` field MAY contain an issuance timestamp. This time SHOULD NOT be trusted; it is only a claim by the Invoker of their system time. System clocks often have clock skew, or a Byzantine Invoker could claim an arbitrary time.

## 3.2 Task

A Task is the subset of fields that uniquely determine the work to be performed[^subtype]. A Task MUST be unqiuely defined by the following fields:

- [Subject]
- [Command]
- [Arguments]
- [Nonce]

The nonce is important for distinguishing between non-idempotent executions of a Task by making the group together unique.

[^subtype]: An Invocation is thus a subtype of Task: $\textsf{Invocation} <: \textsf{Task}$

## 3.3 Attestation

An Invocation MAY be used to attest to some information. This is in effect a statement to the Issuer (without Audience) that never expires.

## 3.4 Proof Chains

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

### 3.4.1 Proxied Proof Paths with `ucan/proxy`

Beyond [attenuation], [`ucan/proxy`] MAY be used to connect otherwise disjoint parts of an authorization network. The motivation is to express the intention of automatically re-delegating (or "forwarding") authority to another agent if you are offline, while retaining the ability to [revoke] that link. The clear use case is linking user devices, but also has applications for PoLA "cold" root/admin keys for servers.

The `ucan/proxy` Command MAY be used to substitute into any delegation chain. It "forwards" whatever is later in the chain, in effect swapping out the `iss` field. `ucan/proxy` MUST NOT change the Ability it re-delegates. It MAY be scoped to a particular scheme or attach additional caveats.

``` js
// Anything
{
  "sub": "did:web:example.com",
  "can": "ucan/proxy",
  "cond": [],
  // ...
}
```

``` mermaid
sequenceDiagram
    actor Alice
    actor Bob
    actor Carol
    actor Dan
    
    autonumber

    Note over Alice, Dan: Delegation Setup
      Bob -->> Carol: Delegate(ucan/proxy)
      Alice -->> Bob: Delegate(crud/create, dns:example.com)
      Carol -->> Dan: Delegate(crud/create, dns:example.com)

    Note over Alice, Dan: Invoke
      Dan ->> Alice: Invoke(crud/create, dns:example.com, txt="hi", proof: [➋,➊,➌])

    Note over Alice, Dan: Delegation path in ➍
      autonumber 2
      Alice -->> Bob: Delegate(crud/create, dns:example.com)

      autonumber 1
      rect rgb(127, 127, 127)
          Bob -->> Carol: Delegate(ucan/proxy)
      end

      autonumber 3
      Carol -->> Dan: Delegate(crud/create, dns:example.com)
```

```js
// Only DNS resources
{
  "sub": "did:web:example.com",
  "can": "ucan/proxy",
  "args": {
    "scheme": "dns"
  }
  "if": [],
  // ...
}
```

## 3.5 Examples

### 3.5.1 Interacting with an HTTP API

```js
// DAG-JSON
{
  "s": {"/": {bytes: "7aEDQIscUKVuAIB2Yj6jdX5ru9OcnQLxLutvHPjeMD3pbtHIoErFpo7OoC79Oe2ShgQMLbo2e6dvHh9scqHKEOmieA0"}},
  "p": {
    "h": {"/": {"bytes": "NBIFEgEAcQ"}},
    "ucan/i/1.0.0-rc.1": {
      "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
      "sub": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "do": "crud/create",
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
        {"/": "bafkr4iblvgvkmqt46imsmwqkjs7p6wmpswak2p5hlpagl2htiox272xyy4"},
        {"/": "bafkr4idnrqfouibxdqpvh2lmkhgsbw5yabvjbiaea3fplrb4vxifaphvgy"},
        {"/": "bafkr4ig4o5mwufavfewt4jurycn7g7dby2tcwg5q2ii2y6idnwguoyeruq"}
      ]
    }
    
  }
}
```

### 3.5.2 Sending Email

```js
// DAG-JSON
{
  "s": {"/": {bytes: "7aEDQIscUKVuAIB2Yj6jdX5ru9OcnQLxLutvHPjeMD3pbtHIoErFpo7OoC79Oe2ShgQMLbo2e6dvHh9scqHKEOmieA0"}},
  "p": {
    "h": {"/": {"bytes": "NBIFEgEAcQ"}},
    "ucan/i/1.0.0-rc.1": {
      "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
      "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "sub": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "do": "msg/send",
      "args": {
        "from": "mailto:akiko@example.com",
        "to": [ "boris@example.com", "carol@example.com" ],
        "subject": "Coffee",
        "body": "Let get coffee sometime and talk about UCAN Invocations!"
      },
      "nonce": {"/": {"bytes": "TWFueSBopZ2h0IHdvcs"}},
      "mta": {},
      "prf": [{"/": "bafkr4iblvgvkmqt46imsmwqkjs7p6wmpswak2p5hlpagl2htiox272xyy4"}],
      "exp": 1697409438
    }
  }
}
```

### 3.5.3 Inline WebAssembly

```js
{
  "s": {"/": {bytes: "7aEDQIscUKVuAIB2Yj6jdX5ru9OcnQLxLutvHPjeMD3pbtHIoErFpo7OoC79Oe2ShgQMLbo2e6dvHh9scqHKEOmieA0"}},
  "p": {
    "h": {"/": {"bytes": "NBIFEgEAcQ"}},
    "ucan/i/1.0.0-rc.1": {
      "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
      "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "sub": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "meta": {"fuel": 999999},
      "nonce": {"/": {"bytes": ""}}, // NOTE: as stated above, idempotent Actions should always have the same nonce
      "do": "wasm/run",
      "args": {
        "mod": "data:application/wasm;base64,AHdhc21lci11bml2ZXJzYWwAAAAAAOAEAAAAAAAAAAD9e7+p/QMAkSAEABH9e8GowANf1uz///8UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAP////8AAAAACAAAACoAAAAIAAAABAAAACsAAAAMAAAACAAAANz///8AAAAA1P///wMAAAAlAAAALAAAAAAAAAAUAAAA/Xu/qf0DAJHzDx/44wMBqvMDAqphAkC5YAA/1mACALnzB0H4/XvBqMADX9bU////LAAAAAAAAAAAAAAAAAAAAAAAAAAvVXNlcnMvZXhwZWRlL0Rlc2t0b3AvdGVzdC53YXQAAGFkZF9vbmUHAAAAAAAAAAAAAAAAYWRkX29uZV9mAAAADAAAAAAAAAABAAAAAAAAAAkAAADk////AAAAAPz///8BAAAA9f///wEAAAAAAAAAAQAAAB4AAACM////pP///wAAAACc////AQAAAAAAAAAAAAAAnP///wAAAAAAAAAAlP7//wAAAACM/v//iP///wAAAAABAAAAiP///6D///8BAAAAqP///wEAAACk////AAAAAJz///8AAAAAlP///wAAAACM////AAAAAIT///8AAAAAAAAAAAAAAAAAAAAAAAAAAET+//8BAAAAWP7//wEAAABY/v//AQAAAID+//8BAAAAxP7//wEAAADU/v//AAAAAMz+//8AAAAAxP7//wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAU////pP///wAAAAAAAQEBAQAAAAAAAACQ////AAAAAIj///8AAAAAAAAAAAAAAADQAQAAAAAAAA==",
        "fun": "add_one",
        "params": [42]
      }
    }
  }
}
```

# 4 Response

A `Receipt` is a kind of Invocation used to attest to the result of another Invocation. A Receipt MUST be issued by the [Executor] (including by [Execution Proxy]).

**NB: a Receipt does not guarantee correctness of the result!** The statement's veracity MUST be only understood as an attestation from the executor.

Receipts MUST use the same-or-higher version number as the [Invocation] that they reference.

| Field | Type        | Required | Description                                                                                    |
|-------|-------------|----------|------------------------------------------------------------------------------------------------|
| `s`   | `Signature` | Yes      | Signature (bytes or struct) of the `pld` field, which MUST be interpreted as the `pld.h` field |
| `p`   | `&Payload`  | Yes      | The data being signed over                                                                     |

## 3.2 Signature Payload

| Field               | Type             | Required | Description                                          |
|---------------------|------------------|----------|------------------------------------------------------|
| `h`                 | `VarsigHeader`   | Yes      | [Varsig] header that describes the outer `sig` field |
| `ucan/r/1.0.0-rc.1` | `ReceiptPayload` | Yes      | Fields unique to the Receipt                         |

## 4.2 Receipt Payload

Receipt Payloads MUST conform to the following shape:

| Field  | Type               | Required | Description                                                                        |
|--------|--------------------|----------|------------------------------------------------------------------------------------|
| `iss`  | `DID`              | Yes      | The DID of the Executor                                                            |
| `ran`  | `&Invocation`      | Yes      | A link to the [Invocation] that the Receipt is for                                 |
| `out`  | `Result`           | Yes      | The value output of the invocation in [Result] format                              |
| `prf`  | `[&Delegation]`    | Yes      | [Delegation] proof chain if the Executor was not the `aud` of the `ran` Invocation |
| `next` | `[&Task]`          | Yes      | Further [Task]s that the [Invocation] would like to enqueue                        |
| `meta` | `{String : Any}`   | Yes      | Additional data about the receipt                                                  |
| `iat`  | `Integer`[^js-num] | No       | The UTC Unix timestamp at which the Receipt was issued                             |

A few of these fields warrant further comment below.

### 4.1 Result

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

## 4.2 Next Task(s)

The result of an [Invocation] MAY include a request for further actions to be performed. This is a process of requesting that the invoker "enqueue" a new Task. This enables several things: a clean separation of pure return values from requesting impure tasks to be performed by the runtime, and gives the runtime the control to decide how (or if!) more work should be performed.

Enqueued [Task]s describe requests for future work to be performed. They SHOULD come with [Delegation]s, but MAY be a simple request back to the Invoker.

All [Task]s in an [enqueue] array MUST be treated as concurrent, unless explicit data dependencies between them exist via [UCAN Promise]s.

## 4.3 Examples

``` js
// DAG-JSON
{
  "s": {"/": {"bytes": "7aEDQLYvb3lygk9yvAbk0OZD0q+iF9c3+wpZC4YlFThkiNShcVriobPFr/wl3akjM18VvIv/Zw2LtA4uUmB5m8PWEAU"}},
  "p": {
    "h": {"/": {"bytes": "NBIFEgEAcQ"}},
    "ucan/r/1.0.0-rc.1": {
      "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
      "ran": {"/": "bafkreictzcfwelyww7zmjkl5nptyot24oilky2bppw42nui2acozhfmzqa"},
      "out": {"ok": 42},
      "iat": 1702907627,
      "prf": [],
    }
  }
}
```

# 5 Proxy Execution

If the Receipt Issuer is not identical to the `aud` field of Invocation referenced in the `ran` field, a [Delegation] proof chain SHOULD be included. If a chain is present, it MUST show that such a proxy execution was authorized by the original listed `aud` Agent.

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
  aud: "did:web:worker.not-example.net",
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

## 5.1 Examples

``` js
// DAG-JSON
{
  "s": {"/": {"bytes": "7aEDQLYvb3lygk9yvAbk0OZD0q+iF9c3+wpZC4YlFThkiNShcVriobPFr/wl3akjM18VvIv/Zw2LtA4uUmB5m8PWEAU"}},
  "p": {
    "h": {"/": {"bytes": "NBIFEgEAcQ"}},
    "ucan/i/1.0.0-rc.1": {
      "iss": "did:plc:ewvi7nxzyoun6zhxrhs64oiz",
      "aud": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "sub": "did:key:z6MkrZ1r5XBFZjBU34qyD8fueMbMRkKw17BZaq2ivKFjnz2z",
      "do": "ucan/receipt",
      "args": {
        "ran": {"/": "bafyreia5tctxekbm5bmuf6tsvragyvjdiiceg5q6wghfjiqczcuevmdqcu"},
        "out": {"ok": ["bob@example.com", "alice@example.com"]}
      },
      "meta": {
        "retry-count": 2,
        "total-time": [400, "hours"]
      },
      "prf": []
    }
  }
}
```

# 6 Prior Art

[ucanto RPC] from [DAG House] is a production system that uses UCAN as the basis for an RPC layer.

The Capability Transport Protocol ([CapTP]) is one of the most influential object-capability systems, and forms the basis for much of the rest of the items on this list.

The Object Capability Network ([OCapN]) protocol extends [CapTP] with a generalized networking layer. It has implementations from the [Spritely Institute] and [Agoric]. At time of writing, it is in the process of being standardized.

[Electronic Rights Transfer Protocol (ERTP)] builds on top of [CapTP] concepts for blockchain & digital asset use cases.

[Cap 'n Proto RPC] is an influential RPC framework based on concepts from [CapTP].

# 7 Acknowledgements

Many thanks to [Mark Miller] for his [trail blazing work][eRights] on [capability systems].

Many thanks to [Luke Marsen] and [Simon Worthington] for their feedback on invocation model from their work on [Bacalhau] and [IPVM].

Thanks to [Marc-Antoine Parent] for his discussions of the distinction between declarations and directives both in and out of a UCAN context.

Many thanks to [Quinn Wilton] for her discussion of speech acts, the dangers of signing canonicalized data, and ergonomics.

Thanks to [Blaine Cook] for sharing their experiences with [OAuth 1], irreversible design decisions, and advocating for keeping the spec simple-but-evolvable.

Thanks to [Philipp Krüger] for the enthusiastic feedback on the overall design and encoding.

Thanks to [Christine Lemmer-Webber] for the many conversations about capability systems and the programming models that they enable.

Thanks to [Rod Vagg] for the clarifications on IPLD Schema implicits and the general IPLD worldview

<!-- Footnotes -->

[^js-num]: JavaScript has a single numeric type ([`Number`][JS Number]) for both integers and floats. This representation is defined as a [IEEE-754] double-precision floating point number, which has a 53-bit significand.
 
<!-- Internal Links -->

[Action]: #31-action
[Arguments]: #312-arguments
[Command]: #311-command
[Execution Proxy]: #423-proxy-execution
[Executor]: #212-executor
[Expiry]: #317-expiry
[Invocation Payload]: #331-invocation-payload
[Invocation Signature]: #331-invocation-envelope
[Invocation]: #33-invocation
[Invoker]: #211-invoker
[Issued At]: #318-issued-at
[Metadata]: #315-metadata 
[Nonce]: #314-nonce
[Proofs]: #316-proofs
[Receipt Payload]: #42-receipt-payload
[Receipt]: #41-receipt-envelope
[Response]: #4-response
[Result]: #421-result
[Subject]: #313-subject
[Task]: #32-task
[enqueue]: #422-enqueue
[lazy-vs-eager]: #112-lazy-vs-eager-evaluation

<!-- External Links -->

[Agoric]: https://agoric.com/
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[Bacalhau]: https://www.bacalhau.org/
[Blaine Cook]: https://github.com/blaine
[Brooklyn Zelenka]: https://github.com/expede/
[Cap 'n Proto RPC]: https://capnproto.org/
[CapTP]: http://erights.org/elib/distrib/captp/index.html
[Christine Lemmer-Webber]: https://github.com/cwebber
[DAG House]: https://dag.house
[DAG-CBOR]: https://ipld.io/specs/codecs/dag-cbor/spec/
[DID]: https://www.w3.org/TR/did-core/
[Delegation]: https://github.com/ucan-wg/delegation
[E-lang Mailing List, 2000 Oct 18]: http://wiki.erights.org/wiki/Capability-based_Active_Invocation_Certificates
[Electronic Rights Transfer Protocol (ERTP)]: https://docs.agoric.com/guides/ertp/
[Fission]: https://fission.codes/
[Haskell]: https://en.wikipedia.org/wiki/Haskell
[IEEE-754]: https://ieeexplore.ieee.org/document/8766229
[IPLD]: https://ipld.io/
[IPVM]: https://github.com/ipvm-wg
[Irakli Gozalishvili]: https://github.com/Gozala
[JS Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number
[Luke Marsen]: https://github.com/lukemarsden
[Marc-Antoine Parent]: https://github.com/maparent
[Mark Miller]: https://github.com/erights
[OAuth 1]: https://oauth.net/1/
[OCapN]: https://github.com/ocapn/
[Philipp Krüger]: https://github.com/matheus23/
[Quinn Wilton]: https://github.com/QuinnWilton
[Robust Composition]: http://www.erights.org/talks/thesis/markm-thesis.pdf
[Rod Vagg]: https://github.com/rvagg/
[Simon Worthington]: https://github.com/simonwo
[Spritely Institute]: https://spritely.institute/news/introducing-a-distributed-debugger-for-goblins-with-time-travel.html
[UCAN Delegation]: https://github.com/ucan-wg/delegation/
[UCAN Promise]: https://github.com/ucan-wg/promise/
[URI]: https://en.wikipedia.org/wiki/Uniform_Resource_Identifier
[Zeeshan Lakhani]: https://github.com/zeeshanlakhani
[`data`]: https://en.wikipedia.org/wiki/Data_URI_scheme
[`ipfs`]: https://docs.ipfs.tech/how-to/address-ipfs-on-web/#native-urls
[`magnet`]: https://en.wikipedia.org/wiki/Magnet_URI_scheme
[capability systems]: https://en.wikipedia.org/wiki/Capability-based_security
[designation without authorization]: https://srl.cs.jhu.edu/pubs/SRL2003-02.pdf
[distributed promise pipelines]: http://erights.org/elib/distrib/pipeline.html
[eRights]: https://erights.org
[ucanto RPC]: https://github.com/web3-storage/ucanto
