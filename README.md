# UCAN Invocation Specification v0.1.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)

## Authors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
* [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

## Depends On

* [DAG-CBOR](https://ipld.io/specs/codecs/dag-cbor/spec/)
* [UCAN](https://github.com/ucan-wg/spec/)
* [UCAN-IPLD](https://github.com/ucan-wg/ucan-ipld/)
* [VarSig](https://github.com/ChainAgnostic/varsig/)

# 0 Abstract

UCAN Invocation defines a format for expressing the intention to run delegated capabilities from a UCAN, and how to promise pipeline invocations.

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1 Introduction

> Just because you can doesn't mean that you should
>
> — Anonymous

UCAN is a chained-capability format. A UCAN contains all of the information that one would need to perform some task, and the provable authority to do so. This begs the question: can UCAN be used directly as an RPC language?

Some teams have had success with UCAN directly for RPC when the intention is clear from context. This can be successful when there is more information on the channel than the UCAN itself (such as an HTTP path that a UCAN is sent to). However, capability invocation contains strictly more information than delegation: all of the authority of UCAN, plus the command to perform the task.

## 1.1 Intuition

## 1.1.1 Car Keys

Consider the following fictitious scenario:

Akiko is going away for the weekend. Her good friend Boris is going to borrow her car while she's away. They meet at a nearby cafe, and Akiko hands Boris her car keys. Boris now has the capability to drive Alice's car whenever he wants to. Depending on their plans for the rest of the day, Akiko may find Boris quite rude if he immediately leaves the cafe to go for a drive. On the other hand, if Akiko asks Boris to run some last minute pre-vacation errands for that require a car, she may expect Boris to immediately drive off.

## 1.1.2 Lazy vs Eager Evaluation

In a referentially transparent setting, the description of a task is equivalent to having done so: a function and its results are interchangeable. [Programming languages with call-by-need semantics](https://en.wikipedia.org/wiki/Haskell) have shown that this can be an elegant programming model, especially for pure functions. However, _when_ something will run can sometimes be unclear. 

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

## 1.2 Separation of Concerns

Information about the scheduling, order, and pipelining of tasks is orthogonal to the flow of authority. An agent collaborating with the original executor does not need to know that their call is 3 invocations deep; they only need to know that they been asked to perform some task by the latest invoker.

As we shall see in the [discussion of promise pipelining](#5-promise-pipelining), asking an agent to perform a sequence of tasks before you know the exact parameters requires delegating capabilities for all possible steps in the pipeline. Pulling pipelining detail out of the core UCAN spec serves two functions: it keeps the UCAN spec focused on the flow of authority, and makes salient the level of de facto authority that the executor has (since they can claim any value as having returned for any step).

```
  ────────────────────────────────────────────Time──────────────────────────────────────────────────────►

┌──────────────────────────────────────────Delegation─────────────────────────────────────────────────────┐
│                                                                                                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐         ┌─────────┐                ┌─────────┐                 │
│  │         │   │         │   │         │         │         │                │         │                 │
│  │  Alice  ├──►│   Bob   ├──►│  Carol  ├────────►│   Dan   ├───────────────►│  Erin   │                 │
│  │         │   │         │   │         │         │         │                │         │                 │
│  └─────────┘   └─────────┘   └─────────┘         └─────────┘                └─────────┘                 │
│                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────Invocation─────────────────────────────────────────────────────┐
│                                                                                                         │
│                              ┌─────────┐         ┌─────────┐                                            │
│                              │         │         │         │                                            │
│                              │  Carol  ╞═══All══►│   Dan   │                                            │
│                              │         │         │         │                                            │
│                              └─────────┘         └─────────┘                                            │
│                                                                                                         │
│                                                  ┌─────────┐                              ┌─────────┐   │
│                                                  │         │                              │         │   │
│                                                  │   Dan   ╞═══════════Update DB═════════►│  Erin   │   │
│                                                  │         │                              │         │   │
│                                                  └─────────┘                              └─────────┘   │
│                                                                                                         │
│                                                           ┌─────────┐                ┌─────────┐        │
│                                                           │         │                │         │        │
│                                                           │   Dan   ╞═══Read Email══►│  Erin   │        │
│                                                           │         │           ▲    │         │        │
│                                                           └─────────┘           ┆    └─────────┘        │
│                                                                               With                      │
│                                                                               Result                    │
│                                                                  ┌─────────┐   Of         ┌─────────┐   │
│                                                                  │         │    ┆         │         │   │
│                                                                  │   Dan   ╞════Set DNS══►│  Erin   │   │
│                                                                  │         │              │         │   │
│                                                                  └─────────┘              └─────────┘   │
│                                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## 1.3 A Note On Serialization

The JSON examples below are given in [DAG-JSON](https://ipld.io/docs/codecs/known/dag-json/), but UCAN Invocation is actually defined as IPLD. This makes UCAN Invocation agnostic to encoding. DAG-JSON follows particular conventions around wrapping CIDs and binary data in tags like so:

``` json
// CID
{"/": Qmf412jQZiuVUtdgnB36FXFX7xg5V6KEbSJ4dpQuhkLyfD"}

// Bytes (e.g. a signature)
{"/": {"bytes": "s0m3Byte5"}}
```

This format help disambiguate type information in generic DAG-JSON tooling. However, your presentation need not be in this specific format, as long as it can be converted to and from this cleanly. As it is used for the signature format, DAG-CBOR is RECOMMENDED.

## 1.4 Signatures

All payloads described below MUST be signed with a [VarSig](https://github.com/ChainAgnostic/varsig/).

# 2 Roles

Invocation adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delegate principals MUST persist to the invocation.

| UCAN Field | Delegation                             | Invocation                      |
|------------|----------------------------------------|---------------------------------|
| `iss`      | Delegator: transfer authority (active) | Invoker: request task (active)  |
| `aud`      | Delegate: gain authority (passive)     | Executor: perform task (active) |

## 2.1 Invoker

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

## 2.2 Executor

The executor is directed to perform some task described in the UCAN by the invoker.

The executor MUST be the UCAN delegate. Their DID MUST be set the in `aud` field of the contained UCAN.

# 3 Envelope

The invocation envelope is a thin wrapper around a UCAN that conveys that all of the contained tasks SHOULD be performed.

All of the roles from the referenced UCANs MUST persist to the invocation per the [roles table](#2-roles).

The invocation wrapper MUST be signed by the same principal that issued the UCAN.

## 3.1 Fields

| Field | Type                     | Description                                            | Required | Default |
|-------|--------------------------|--------------------------------------------------------|----------|---------|
| `v`   | `SemVer`                 | SemVer of the UCAN invocation this schema (v0.1.0)     | Yes      |         |
| `prf` | `[&UCAN]`                | UCANs that supply the authority to run this invocation | Yes      |         |
| `run` | `"*" or {String : Task}` | Which UCAN capabilities to run                         | Yes      |         |
| `nnc` | `String`                 | A unique nonce, to distinguish each invocation         | Yes      |         |
| `ext` | `Any`                    | Non-normative extended fields                          | No       | `null`  |
    
### 3.1.1 Version

The `v` field MUST contain the version of the invocation object  schema.

### 3.1.2 Proofs

The `prf` field MUST contain CIDs pointing to the UCANs that provide the authority to run these tasks. The elements of this array MUST be sorted in ascending numeric order. Restricting the outmost UCANs to only the authority required for the current invocation is RECOMMENDED.

### 3.1.3 Run Capabilities

The `run` field MUST reference the tasks contained in the UCAN are to be run during the invocation. To run all tasks in the underlying UCAN, the `"*"` value MUST be used. If only specific tasks (or [pipelines](#5-promise-pipelining)) are intended to be run, then they MUST be prefixed with an arbitrary label and treated as a UCAN attenuation: all tasks MUST be backed by a matching capability of equal or greater authority.

#### 3.1.3.1 Promises

The only difference from general capabilities is that [promises](#5-promise-pipelining) MAY also be used as inputs to attenuated fields.

If a capability input has the key `"_"` and the value is a promise, the input MUST be discarded and only used for determining sequencing tasks.

### 3.1.4 Nonce

The `nnc` field MUST contain a unique nonce. This field exists to enable making the CID of each invocation unique. While this field MAY be an empty string, it is NOT RECOMMENDED.

### 3.1.5 Extended Fields

The OPTIONAL `ext` field MAY contain arbitrary data. If not present, the `ext` field MUST be interpreted as `null`, including for [signature](#315-signature).

## 3.2 IPLD Schema

``` ipldsch
type SignedInvocation struct {
  inv Invocation (rename "ucan/invoke") 
  sig VarSig
}

type Invocation struct {
  v   SemVer  -- Version
  prf [&UCAN] -- Authority to run this invocation
  run Scope   -- Which tasks to invoke
  nnc String  -- Nonce
  ext nullable Any (implicit null) -- Extended fields
}

type Scope union {
  | All "*"
  | {String : Task}
}

type Task struct {
  with   URI 
  do     Ability
  inputs Any
  meta   {String : Any}
} 
```

## 3.3 JSON Examples

### 3.3.1 Run All

 ``` js
{
  "ucan/invoke": {
    "v": "0.1.0",
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": "*"
    "nnc": "abcdef",
    "ext": null
  },
  "sig": {"/": { "bytes:": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA" }}
}
```

### 3.3.2 Promise Pipelines

Promise pipelines are handled in more detail in [section 5](#5-promise-pipelining). In brief, they enable taking the output of a task that has not yet run as the input to another task. Here is a simple example:

``` js
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "02468",
    "ext": null,
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": {
      "notify-bob": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "bob@example.com",
            "subject": "DNSLink for example.com",
            "body": "Hello Bob!"
          }
        ]
      },
      "log-as-done": {
        "with": "https://example.com/report"
        "do": "crud/update"
        "inputs": {
          "from": "mailto://alice@exmaple.com",
          "to": ["bob@exmaple.com"],
          "event": "email-notification",
          "value": {"ucan/promise": ["/", "notify-bob"]} // Pipelined promise
        }
      }
    }
  },
  "sig": {"/": {"bytes:": "5vNn4--uTeGk_vayyPuNTYJ71Yr2nWkc6AkTv1QPWSgetpsu8SHegWoDakPVTdxkWb6nhVKAz6JdpgnjABppC7"}}
}
```

# 4 Receipt

An invocation receipt is an attestation of the output of an invocation. A receipt MUST be signed by the executor (the `aud` of the associated UCAN).

**NB: a Receipt this does not guarantee correctness of the result!** The statement's veracity MUST be only understood as an attestation from the executor.

Receipts don't have their own version field. Receipts MUST use the same version as the invocation that they contain.

## 4.1 Fields

| Field  | Type            | Description                                                           | Required | Default |
|--------|-----------------|-----------------------------------------------------------------------|----------|---------|
| `inv`  | `&Invocation`   | CID of the Invocation that generated this response                    | Yes      |         |
| `out`  | `{String: Any}` | The results of each call, the task's label. MAY contain sub-receipts. | Yes      |         |
| `meta` | `Any`           | Non-normative extended fields                                         | No       | `null`  |

### 4.1.1 Invocation

The `inv` field MUST include a link to the Invocation that the Receipt is for.

### 4.1.2 Output

The `out` field MUST contain the output of steps of the call graph, indexed by the task name inside the invocation. If the invocation is the implicit `"*"`, then the base64 hash of the concatenation of the URI, Ability and extensional fields MUST be used.

The `out` field MAY omit any tasks that have not yet completed, or results which are not public. An `Invocation` may be associated to zero or more `Receipts`.

A `Result` MAY include recursive `Receipt` CIDs in on the `Success` branch. As a Task may require subdelegation, the OPTIONAL `rec` field MAY be used to include recursive `Receipt`s.

### 4.1.3 Metadata Fields

The metadata field MAY be omitted or used to contain additional data about the receipt. This field MAY be used for tags, commentary, trace information, and so on.

## 4.2 IPLD Schema

``` ipldsch
type SignedReceipt struct {
  rec Receipt (rename "ucan/receipt")
  sig VarSig
}

type Receipt struct {
  inv  &SignedInvocation
  out  {String : Result}
  meta optional Any
} 

type Result union {
  | Success
  | Failure
}

type Failure struct {
  err   nullable String
  trace Any
}

type Success struct {
  val Any
  rec optional &SignedReceipt
}
```

## 4.3 JSON Example

``` json
{
  "ucan/receipt": {
    "v": "0.1.0",
    "inv": [{"/": "bafkreifzjut3te2nhyekklss27nh3k72ysco7y32koao5eei66wof36n5e"}],
    "out": {
      "ok": {
        "bafkreiakkqwuffzsxrzseo7fweeicqlnqanhvyffjieh3etsbnnkjxbphi": [
          {
            "from": "alice@example.com",
            "text": "Hello world!"
          }, 
          {
            "from": "bob@example.com",
            "text": "What's up?"
          }
        ],
        "bafkreienxymjwglxlb3rkdyeyjt54ddoe4x4qi7a7hyfce3z56zspmy6pm": {
          "http": { 
            "status": 200,
            "body": "hello world"
          },
          "ms": 476
        },
        "delegated-task": {"/": "bafkreieiupg4smeb5ammpsydbiea4yvwzwne5ly4hiripy4cjocqiat3ce"}
      }
      "meta": {
        "notes": "very receipt. such wow.",
        "tags": ["dag-house", "fission"]
      }
    }
  },
  "sig": {"/": {"bytes": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt_VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"}}
 }
```

# 5 Promise Pipelining

> Machines grow faster and memories grow larger. But the speed of light is constant and New York is not getting any closer to Tokyo. As hardware continues to improve, the latency barrier between distant machines will increasingly dominate the performance of distributed computation. When distributed computational steps require unnecessary round trips, compositions of these steps can cause unnecessary cascading sequences of round trips
>
> — [Mark Miller](https://github.com/erights), [Robust Composition](http://www.erights.org/talks/thesis/markm-thesis.pdf)

At UCAN creation time, the UCAN MAY not yet have all of the information required to construct the next request in a sequence. Waiting for each request to complete before proceeding to the next task has a performance impact due to the amount of latency. [Promise pipelining](http://erights.org/elib/distrib/pipeline.html) is a solution to this problem: by referencing a prior invocation, a pipelined invocation can direct the executor to use the output of earlier invocations into the input of a later one. This liberates the invoker from waiting for each step.

The authority to execute later task often cannot be fully attenuated in advance, since the executor controls the reported output of the prior step in a pipeline. When choosing to use pipelining, the invoker MUST delegate capabilities for any of the possible outputs. If tight control is required to limit authority, pipelining SHOULD NOT be used.

Promises MUST resolve to a [`Result`](#42-ipld-schema). If a promise resolves to the `Success` branch, the value in the `val` MUST be extracted and substituted for the promise. Behavior is left undefined if the promise returns on the `Failure` branch.

Values MUST only be pipelined if they resolve to the `"ok"` branch of the `Result`. In the success case, the value inside the `"ok"` field MUST be extracted and replace the promise.

A promise MAY be placed in any Task field. Substituting into the `with` and `do` fields is NOT RECOMMENDED in fully trustless contexts, as it makes it difficult to understand what is involved in the invocation in advance.

## 5.1 Promises
 
## 5.1.1 Fields

| Field       | Type         | Description                                                                      | Required |
|-------------|--------------|----------------------------------------------------------------------------------|----------|
| `promised`  | `CID or "/"` | The Invocation being referenced                                                  | Yes      |
| `taskLabel` | `String`     | The task's label. If the tasks were implicit (`"run": "*"`), the the CID is used | Yes      |

The above table MUST be serialized as a tuple. In JSON, this SHOULD be represented as an array containing the values (but not keys) sequenced in the order they appear in the table.

## 5.1.2 IPLD Schema

``` ipldsch
type Promise struct {
  promised    Target 
  taskLabel String -- The label inside the invocation
} representation tuple

type Target union {
  | Local "/"
  | &Invocation
}
```

## 5.1.3 JSON Examples

``` json
["/", "some-label"]
[{"/": "bafkreiddwsbb7fasjb6k6jodakprtl4lhw6h3g4k7tew7vwwvd63veviie"}, "some-label"]
[{"/": "bafkreiddwsbb7fasjb6k6jodakprtl4lhw6h3g4k7tew7vwwvd63veviie"}, {"/": {"bytes": "bafkreidlqsd6nh6hdgwhr4machsvusobpvn4qfrxfgl5vowoggzk2xpldq"}}]
```

## 5.2 Pipelining

Pipelining uses promises as inputs to determine the required dataflow graph. The following examples both express the following dataflow graph:

```
              ┌────────────────────────────┐
              │                            │
              │ dns://example.com?TYPE=TXT │
              │        crud/update         │
              │                            │
              └───────┬──────────┬─────────┘
                      │          │
                      │          │
┌─────────────────────▼───┐  ┌───▼────────────────────────┐
│                         │  │                            │
│ mailto:alice@exaple.com │  │ mailto://alice@example.com │
│         msg/send        │  │          msg/send          │
│     bob@example.com     │  │      carol@exmaple.com     │
│                         │  │                            │
└─────────────────────┬───┘  └───┬────────────────────────┘
                      │          │
                      │          │
             ┌────────▼──────────▼────────┐
             │                            │
             │ https://example.com/events │
             │         crud/create        │
             │                            │
             └────────────────────────────┘
```

#### 5.2.1 Batched 

 ``` json
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "abcdef",
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": {
      "update-dns" : {
        "with": "dns://example.com?TYPE=TXT",
        "do": "crud/update",
        "inputs": { 
          "value": "hello world",
          "content-type": "text/plain; charset=utf-8"
        }
      },
      "notify-bob": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "bob@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": ["/", "update-dns"]}
          }
        ]
      },
      "notify-carol": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "carol@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": ["/", "update-dns"]}
          }
        ]
      },
      "log-as-done": {
        "with": "https://example.com/report",
        "do": "crud/update",
        "inputs": [
          {
            "from": "mailto://alice@exmaple.com",
            "to": ["bob@exmaple.com", "carol@example.com"],
            "event": "email-notification",
          },
          {
            "_": {"ucan/promise": ["/", "notify-bob"]}
          },
          {
            "_": {"ucan/promise": ["/", "notify-carol"]}
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"}}
}
```

### 5.2.2 Serial Pipeline

```
                ┌────────────────────────────┐
                │                            │
                │ dns://example.com?TYPE=TXT │
                │        crud/update         │
                │                            │
                └───────┬──────────┬─────────┘
                        │          │
                        │          │
                        │          │
                        │      ┌───▼────────────────────────┐
                        │      │                            │
                        │      │ mailto://alice@example.com │
                        │      │          msg/send          │
                        │      │      carol@exmaple.com     │
                        │      │                            │
                        │      └───┬────────────────────────┘
                        │          │
┌───────────────────────┼──────────┼──────────┐
│                       │          │          │
│ ┌─────────────────────▼───┐      │          │
│ │                         │      │          │
│ │ mailto:alice@exaple.com │      │          │
│ │         msg/send        │      │          │
│ │     bob@example.com     │      │          │
│ │                         │      │          │
│ └─────────────────────┬───┘      │          │
│                       │          │          │
│                       │          │          │
│              ┌────────▼──────────▼────────┐ │
│              │                            │ │
│              │ https://example.com/events │ │
│              │         crud/create        │ │
│              │                            │ │
│              └────────────────────────────┘ │
│                                             │
└─────────────────────────────────────────────┘
```

 ``` json
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "abcdef",
    "prf": [{"/": "bafkreifzjut3te2nhyekklss27nh3k72ysco7y32koao5eei66wof36n5e"}],
    "run": {
      "update-dns": {
        "with": "dns://example.com?TYPE=TXT",
        "do": "crud/update",
        "inputs": [
          { 
            "value": "hello world",
            "content-type": "text/plain; charset=utf-8"
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": kQHtTruysx4S8SrvSjTwr6ttTLzc7dd7atANUYT-SRK6v_SX8bjHegWoDak2x6vTAZ6CcVKBt6JGpgnjABpsoL"}}
}
 
{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "12345",
    "prf": [
      {"/": "bafkreie2cyfsaqv5jjy2gadr7mmupmearkvcg7llybfdd7b6fvzzmhazuy"},
      {"/": "bafkreibbz5pksvfjyima4x4mduqpmvql2l4gh5afaj4ktmw6rwompxynx4"}
    ],
    "run": {
      "notify-carol": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "carol@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": ["bafkreieimb4hvcwizp74vu4xfk34oivbdojzqrbpg2y3vcboqy5hwblmeu", "update-dns"]}
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": "XZRSmp5cHaXX6xWzSTxQqC95kQHtTruysx4S8SrvSjTwr6ttTLzc7dd7atANUQJXoWThUiVuCHWdMnQNQJgiJi"}}
}

{
  "ucan/invoke": {
    "v": "0.1.0",
    "nnc": "02468",
    "ext": null,
    "run": {
      "notify-bob": {
        "with": "mailto://alice@example.com",
        "do": "msg/send",
        "inputs": [
          {
            "to": "bob@example.com",
            "subject": "DNSLink for example.com",
            "body": {"ucan/promise": [{"/": "bafkreieimb4hvcwizp74vu4xfk34oivbdojzqrbpg2y3vcboqy5hwblmeu"}, "update-dns"]}
          }
        ]
      },
      "log-as-done": {
        "with": "https://example.com/report",
        "do": "crud/update",
        "inputs": [
          {
            "from": "mailto://alice@exmaple.com",
            "to": ["bob@exmaple.com", "carol@example.com"],
            "event": "email-notification",
          },
          {
            "_": {"ucan/promise": ["/", "notify-bob"]}
          },
          {
            "_": {"ucan/promise": [{"/": "bafkreidcqdxosqave5u5pml3pyikiglozyscgqikvb6foppobtk3hwkjn4"}, "notify-carol"]}
          }
        ]
      }
    }
  },
  "sig": {"/": {"bytes": "5vNn4--uTeGk_vayyPuNTYJ71Yr2nWkc6AkTv1QPWSgetpsu8SHegWoDakPVTdxkWb6nhVKAz6JdpgnjABppC7"}}
}
```

# 6 Appendix

## 6.1 Support Types

``` ipldsch
type CID String
type URI String
type Ability String
type Path String
type VarSig Bytes

type SemVer struct {
  num NumVer
  tag optional String
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

Many thanks to [Mark Miller](https://github.com/erights) for his [pioneering work](http://erights.org/talks/thesis/markm-thesis.pdf) on [capability systems](http://erights.org/).

Many thanks to [Luke Marsen](https://github.com/lukemarsden) and [Simon Worthington](https://github.com/simonwo) for their feedback on invocation model from their work on [Bacalhau](https://www.bacalhau.org/) and [IPVM](https://github.com/ipvm-wg).

Thanks to [Marc-Antoine Parent](https://github.com/maparent) for his discussions of the distinction between declarations and directives both in and out of a UCAN context.

Many thanks to [Quinn Wilton](https://github.com/QuinnWilton) for her discussion of speech acts, the dangers of signing canonicalized data, and ergonomics.

Thanks to [Blaine Cook](https://github.com/blaine) for sharing their experiences with OAuth 1, irreversible design decisions, and advocating for keeping the spec simple-but-evolvable.

Thanks to [Philipp Krüger](https://github.com/matheus23/) for the enthusiastic feedback on the overall design and encoding.

Thanks to [Christine Lemmer-Webber](https://github.com/cwebber) for the many conversations about capability systems and the programming models that they enable.

Thanks to [Rod Vagg](https://github.com/rvagg/) for the clarifications on IPLD Schema implicits and the general IPLD worldview.
