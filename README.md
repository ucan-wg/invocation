# UCAN Invocation Specification v0.1.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)

## Authors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
* [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

## Depends On

* [UCAN](https://github.com/ucan-wg/spec/)
* [`ucan-ipld`](https://github.com/ucan-wg/ucan-ipld/)

# 0 Abstract

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1 Introduction

> Just because you can doesn't mean that you should
>
> — Anonymous

UCAN is a chained-capability format. A UCAN contains all of the information that one would need to perform some action, and the provable authority to do so. This begs the question: can UCAN be used directly as an RPC language?

Some teams have had success using UCAN directly for RPC when the intention is clear from context. This can be successful when there is more information on the channel than the UCAN itself (such as an HTTP path that a UCAN is sent to). However, capability invocation contains strictly more information than delegation: all of the authority of UCAN, plus the command to perform the action.

## 1.1 Intuition

## 1.1.1 Car Keys

Consider the following fictitious scenario:

Akiko is going away for the weekend. Her good friend Boris is going to borrow her car while she's away. They meet at a nearby cafe, and Akiko hands Boris her car keys. Boris now has the capability to drive Alice's car whenever he wants to. Depending on their plans for the rest of the day, Akiko may find Boris quite rude if he immediately leaves the cafe to go for a drive. On the other hand, if Akiko asks Boris to run some last minute pre-vacation errands for that require a car, she may expect Boris to immediately drive off.

## 1.1.2 Lazy vs Eager Evaluation

In a referentially transparent setting, the description of an action is equivalent to having done so: a function and its results are interchangeable. [Programming languages with call-by-need semantics](https://en.wikipedia.org/wiki/Haskell) have shown that this can be an elegant programming model, especially for pure functions. However, _when_ something will run can sometimes be unclear. 

Most languages use eager evaluation. Eager languages must contend directly with the distinction between a reference to a function and a command to run it. For instance, in JavaScript, adding parentheses to a function will run it. Omitting them lets the program pass around a reference to the function without immediately invoking it.

``` js
const message = () => alert("hello world")
message // Nothing happens
message() // A message interups the user
```

Delegating a capability is like the statement `message`. Invocation is akin to `message()`. It's true that sometimes we know to run things from their surrounding context without the parentheses:

```js
[1,2,3].map(message) // Message runs 3 times 
```

However, there is clearly a distinction between passing a function and invoking it. The same is true for capabilities: delegating the authority to do something is not the same as asking for it to be done immediately, even if sometimes it's clear from context.

## 1.3 Separation of Concerns

Information about the scheduling, order, and pipelining of actions is orthogonal to the flow of authority. An agent collaborating with the original executor does not need to know that their call is 3 invocations deep; they only need to know that they been asked to perform some action by the latest invoker.

As we shall see in the [discussion of promise pipelining](#5-promise-pipelining), asking an agent to perform a sequence of actions before you know the exact parameters requires delegating capabilities for all possible steps in the pipeline. Pulling pipelining detail out of the core UCAN spec serves two functions: it keeps the UCAN spec focused on the flow of authority, and makes salient the level of de facto authority that the executor has (since they can claim any value as having returned for any step).

# 2 Roles

Invocation adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delegate principals MUST persist to the invocation.

| UCAN Field | Delegation                             | Invocation                        |
|------------|----------------------------------------|-----------------------------------|
| `iss`      | Delegator: transfer authority (active) | Invoker: request action (active)  |
| `aud`      | Delegate: gain authority (passive)     | Executor: perform action (active) |

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

## 2.2 Executor

The executor is directed to perform some task described in the UCAN by the invoker.

The executor MUST be the UCAN delegate. Their DID MUST be set the in `aud` field of the contained UCAN.

# 3 Envelope

The invocation envelope is a thin wrapper around a UCAN that conveys that all of the contained UCAN actions SHOULD be performed.

All of the roles from the referenced UCANs MUST persist to the invocation per the [roles table](#2-roles).

The invocation wrapper MUST be signed by the same principal that issued the UCAN.

## 3.1 Fields

| Field         | Type                               | Value     | Description                                      | Required | Default |
|---------------|------------------------------------|-----------|--------------------------------------------------|----------|---------|
| `ucan/invoke` | `CID`                              |           | The CID of the UCAN to invoke                    | Yes      |         |
| `v`           | `SemVer`                           | `"0.1.0"` | SemVer of the UCAN invocation object schema      | Yes      |         |
| `run`         | `"*" or {URI : {Ability : [Any]}}` |           | Which UCAN capabilities to run                   | No       | `"*"`   |
| `nnc`         | `String`                           |           | A unique nonce, to distinguish each invocation   | Yes      |         |
| `ext`         | `Any`                              |           | Non-normative extended fields                    | No       | `null`  |
| `sig`         | `Bytes`                            |           | Signature of the rest of the field canonicalized | Yes      |         |
  
### 3.1.1 Invocation

The `ucan/invoke` field MUST contain CIDs pointing to the UCANs to invoke. The outmost UCAN being invoked SHOULD NOT contain any actions that are not intended to be executed.

### 3.1.2 Version

The `v` field MUST contain the version of the invocation object  schema.

### 3.1.3 Run Capabilities

The OPTIONAL `run` field MUST reference the actions contained in the UCAN are to be run during the invocation. To run all actions in the underlying UCAN, the `"*"` value MUST be used. If only specific actions (or [pipelines](#5-promise-pipelining)) are intended to be run, then they MUST be treated as a UCAN attenuation: all actions MUST be backed by a matching capability of equal or lesser scope.

#### 3.1.3.1 Promises

The only difference from general capabilities is that [promises](#5-promise-pipelining) MAY also be used as inputs to attenuated fields.

If a capability input has the key `"_"` and the value is a promise, the input MUST be discarded and only used for determining sequencing actions.

### 3.1.4 Nonce

The `nnc` field MUS contain a unique nonce. This field makes each invocation unique.

### 3.1.5 Extended Fields

The OPTIONAL `ext` field MAY contain arbitrary data. If not present, the `ext` field MUST me interpreted as `null`, including for [signature](#315-signature).

### 3.1.6 Signature

The `sig` field MUST contain the signature of the other fields. To construct the payload, the other fields MUST first be canonicalized as `dag-cbor`.

If serialized as JSON, the `sig` field MUST be serialized as [unpadded base64url](https://datatracker.ietf.org/doc/html/rfc4648#section-5).

## 3.2 IPLD Schema

``` haskell
type Invocation struct {
  inv &UCAN  -- The UCAN providing authority
  v   SemVer -- Version
  run Scope  -- Which actions to invoke
  nnc String -- Nonce
  ext Any    -- Extended fields
  sig Bytes  -- Signature
}

type Scope enum {
  | All ("*")
  | {URI : {Ability : [Any]}}
}
```

## 3.3 JSON Examples

 ``` json
{
  "ucan/invoke": "QmYW8Z58V1v8R25USVPUuFHtU7nGouApdGTk3vRPXmVHPR",
  "v": "0.1.0",
  "nnc": "abcdef",
  "ext": null,
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
}
```

### 3.3.2 Pipelines

The following examples both express the following dataflow graph:

```
                 ┌────────────────────────────┐
                 │                            │
                 │ dns://example.com?TYPE=TXT │
                 │        crud/update         │
                 │                            │
                 └───────┬──────────┬─────────┘
                         │          │
                         │          │
┌────────────────────────▼───┐  ┌───▼────────────────────────┐
│                            │  │                            │
│ https://example.com/report │  │ mailto://alice@example.com │
│         crud/update        │  │          msg/send          │
│                            │  │                            │
└────────────────────────┬───┘  └───┬────────────────────────┘
                         │          │
                         │          │
                ┌────────▼──────────▼────────┐
                │                            │
                │ https://example.com/events │
                │         crud/create        │
                │                            │
                └────────────────────────────┘
```

#### 3.3.2.1 Batched 

 ``` json
{
  "ucan/invoke": "QmYW8Z58V1v8R25USVPUuFHtU7nGouApdGTk3vRPXmVHPR",
  "v": "0.1.0",
  "nnc": "abcdef",
  "ext": null,
  "run": {
    "dns://example.com?TYPE=TXT": {
      "crud/update": [
        { 
          "value": "hello world",
          "retries": 5
        }
      ]
    },
    "https://example.com/report": {
      "crud/update": [
        {
          "performedBy": "did:key:zAlice",
          "tags": ["hello", "world"],
          "status": {"ucan/promise": ["/", "dns://example.com?TYPE=TXT", "crud/update", "http/statusCode"]},
          "payload": {"ucan/promise": ["/", "dns://example.com?TYPE=TXT", "crud/update", "http/body"]}
        }
      ]
    },
    "mailto://alice@example.com": {
      "msg/send": [
        {
          "to": "bob@example.com",
          "subject": "DNSLink for example.com",
          "body": {"ucan/promise": ["/", "dns://example.com?TYPE=TXT", "crud/update", "http/body"]}
        }
      ]
    },
    "https://example.com/events": {
      "crud/create": [
        { 
          "event": "update-dns",
          "status": "done"
        },
        {
          "_": [
            {"ucan/promise": ["/", "mailto://alice@example.com", "msg/send", ""]}
            {"ucan/promise": ["/", "https://example.com", "crud/update", ""]}
          ]
        }
      ]
    }
  },
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
}
```

### 3.3.3 Serial Pipeline

 ``` json
{
  "ucan/invoke": "Qmd4trNUhgWwsBbSsYBEWJqgiHyLrnhZ8u1DJsWsEKeuuF",
  "v": "0.1.0",
  "nnc": "abcdef",
  "ext": null,
  "run": {
    "dns://example.com?TYPE=TXT": {
      "crud/update": [
        { 
          "value": "hello world",
          "retries": 5
        }
      ]
    }
  },
  "sig": "kQHtTruysx4S8SrvSjTwr6ttTLzc7dd7atANUYT-SRK6v_SX8bjHegWoDak2x6vTAZ6CcVKBt6JGpgnjABpsoL"
}
 
{
  "ucan/invoke": "QmbXdT8QQMJ55Lb6MGJqTmwNzuUnHsE18t7zXGWeq9rQcV",
  "v": "0.1.0",
  "nnc": "12345",
  "ext": null,
  "run": {
    "http://example.com/report": {
      "http/post": [
        {"updateTo": {"ucan/promise": ["QmWqWBitVJX69QrEjzKsVTy3KQRK6snUoHaPSjmQpxvP1f", "dns://example.com?TYPE=TXT", "crud/update", "http/0/statusCode"]}}
      ]
    },
    "mailto://alice@example.com": {
      "msg/send": [
        {
          "to": "bob@example.com",
          "subject": "DNSLink for example.com",
          "body": {"ucan/promise": ["QmWqWBitVJX69QrEjzKsVTy3KQRK6snUoHaPSjmQpxvP1f", "dns://example.com?TYPE=TXT", "crud/update", "http/0/body"]}
        }
      ]
    }
  },
  "sig": "XZRSmp5cHaXX6xWzSTxQqC95kQHtTruysx4S8SrvSjTwr6ttTLzc7dd7atANUQJXoWThUiVuCHWdMnQNQJgiJi"
}

{
  "ucan/invoke": "QmY4jEVE35u8SHegWoDak2x6vTAZ6Cc4cpSD5LAUQ23W7L",
  "v": "0.1.0",
  "nnc": "02468",
  "ext": null,
  "run": {
    "https://example.com/events": {
      "crud/create": [
        { 
          "event": "update-dns",
          "status": "done"
        },
        {
          "_": [
            {"ucan/promise": ["/", "mailto://alice@example.com", "msg/send", ""]}
            {"ucan/promise": ["/", "https://example.com", "crud/update", ""]}
          ]
        }
      ]
    }
  },
  "sig": "5vNn4--uTeGk_vayyPuNTYJ71Yr2nWkc6AkTv1QPWSgetpsu8SHegWoDakPVTdxkWb6nhVKAz6JdpgnjABppC7"
}
```

# 4 Receipt

An invocation receipt is an attestation of the output of an invocation. A receipt MUST be signed by the executor (the `aud` of the associated UCAN).

Note that this does not guarantee correctness of the result! The statement's veracity MUST be only understood as an attestation from the executor.

## 4.1 Fields

| Field          | Type         | Description                                                                                                                        | Required | Default |
|----------------|--------------|------------------------------------------------------------------------------------------------------------------------------------|----------|---------|
| `ucan/receipt` | `CID`        | CID of the Invocation that generated this response                                                                                 | Yes      |         |
| `rlt`          | `{CID: Any}` | The results of each call, indexed by the CID of the [canonicalized capability](https://github.com/ucan-wg/ucan-ipld#22-capability) | Yes      |         |
| `v`            | `SemVer`     | SemVer of the UCAN invocation object schema                                                                                        | Yes      |         |
| `nnc`          | `String`     | A unique nonce, to distinguish each receipt                                                                                        | Yes      |         |
| `ext`          | `Any`        | Non-normative extended fields                                                                                                      | No       | `null`  |
| `sig`          | `Bytes`      | Signature of the rest of the field canonicalized                                                                                   | Yes      |         |

## 4.1 IPLD

``` haskell
type Receipt struct {
  rec &Invocation
  rlt {URI : {Ability : Any}}
  v   SemVer
  nnc String
  ext optional Any
  sig Bytes
} 
```

## 4.2 JSON Example

``` json
{
  "ucan/receipt": "QmWqWBitVJX69QrEjzKsVTy3KQRK6snUoHaPSjmQpxvP1f",
  "v": "0.1.0",
  "rlt": {
    "QmYqbJBxCqqzDouPMbcrbNuB3WHWajSyq4RWin7ufs2ajf": [
      {
        "from": "alice@example.com",
        "text": "Hello world!"
      }, 
      {
        "from": "bob@example.com",
        "text": "What's up?"
      }
    ],
    "QmXrfqKNUpRiyyi8r3hpSZyZuG3S2MY9rTw1p8iEC9FNh5": {
      "http": { 
        "status": 200,
        "body": "hello world"
      },
      "ms": 476
    }
  },
  "nnc": "xyz",
  "ext": {
    "notes": "very receipt. such wow.",
    "tags": ["dag-house", "fission"]
  },
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt_VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
 }
```

# 5 Promise Pipelining

> Machines grow faster and memories grow larger. But the speed of light is constant and New York is not getting any closer to Tokyo. As hardware continues to improve, the latency barrier between distant machines will increasingly dominate the performance of distributed computation. When distributed computational steps require unnecessary round trips, compositions of these steps can cause unnecessary cascading sequences of round trips
>
> Mark Miller, Robust Composition

At UCAN creation time, the UCAN MAY not yet have all of the information required to construct the next request in a sequence. Waiting for each request to complete before proceeding to the next action has a performance impact due to the amount of latency. [Promise pipelining](http://erights.org/elib/distrib/pipeline.html) is a solution to this problem: by referencing a prior invocation, a pipelined invocation can direct the executor to use the output of earlier invocations into the input of a later one. This liberates the invoker from waiting for each step.

The authority to execute later actions often cannot be fully attenuated in advance, since the executor controls the reported output of the prior step in a pipeline. When choosing to use pipelining, the invoker MUST delegate capabilities for any of the possible outputs. If tight control is required to limit authority, pipelining SHOULD NOT be used.

## 5.1 Fields

| Field           | Type         | Description                     | Required |
|-----------------|--------------|---------------------------------|----------|
| `ucan/promised` | `CID or "/"` | The Invocation being referenced | Yes      |
| `using`         | `URI`        | The resource called             | Yes      |
| `called`        | `Ability`    | The ability used                | Yes      |
| `path`          | `Path`       | Path to the specific output     | No       |

The above table MUST be serialized as a tuple. In JSON, this SHOULD be represented as an array containing the values (but not keys) sequenced in the order they appear in the table.

## 5.2 IPLD Schema

``` haskell
type Promise struct {
  pse &Invocation
  usg URI
  cll Ability -- output path
  pth optional Path
} representation tuple
```

## 5.3 JSON Examples

``` js
// All outputs
["QmYW8Z58V1v8R25USVPUuFHtU7nGouApdGTk3vRPXmVHPR", "example.com/foo/bar", "http/get"]

// Only the status code
["QmYW8Z58V1v8R25USVPUuFHtU7nGouApdGTk3vRPXmVHPR", "example.com/foo/bar", "http/put", "statusCode"]
```

# 6 Appendix

## 6.1 Support Types

``` haskell
type CID = String
type URI = String
type Ability = String
type Path = String

type SemVer struct {
  num NumVer
  tag nullable optional String
} representation stringjoin {
  join "+"
}

type NumVer struct {
  ma Integer
  mi Integer
  pa Integer
} representation stringjoin {
  join "."
}
```

# 7 Prior Art

[ucanto RPC](https://github.com/web3-storage/ucanto) from DAG House is a production system that uses UCAN as the basis for an RPC layer.

The [Capability Transport Protocol (CapTP)](http://erights.org/elib/distrib/captp/index.html) is one of the most influential object-capability systems, and forms the basis for much of the rest of the items on this list.

The [Object Capability Network (OCapN)](https://github.com/ocapn/ocapn) protocol extends CapTP with a generalized networking layer. It has implementations from the [Spritely Institute](https://www.spritely.institute/) and [Agoric](https://agoric.com/). At time of writing, it is in the process of being standardized.

[Electronic Rights Transfer Protocol (ERTP)](https://docs.agoric.com/guides/ertp/) builds on top of CapTP for blockchain & digital asset use cases.

[Cap 'n Proto RPC](https://capnproto.org/) is an influential RPC framework [based on concepts from CapTP](https://capnproto.org/rpc.html#specification).

# 8 Acknowledgements

Thanks to the [DAG House](https://dag.house) team for initially suggesting UCAN as a generalized RPC framework.

Many thanks to [Mark Miller](https://github.com/erights) for his [pioneering work](http://erights.org/talks/thesis/markm-thesis.pdf) on [capability systems](http://erights.org/).

Thanks to [Christine Lemmer-Webber](https://github.com/cwebber) for the many conversations about capability systems and the programming models that they enable.

Thanks to [Quinn Wilton](https://github.com/QuinnWilton) and [Marc-Antoine Parent](https://github.com/maparent) for their discussion of the distinction between declarations and directives.
