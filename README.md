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

## 1.1 Motivation

Why separate the UCAN from the invocation format? Surely the UCAN itself already contains all the infomation required.

A UCAN contains everythiung needed to execute a request

"intention"

Dleegation and execution are opposite pointing arrows

## 1.2 Separation

I would argue that this is not the case for a few reasons.

1. Authority is not intent to execute. Authority transfer is first-class in UCAN, other actions are not
2. Mixing the two levels means that invocation info will live with the token (as a proof) on subdelegations
3. Other detail, such as scheduling

## 1.3 Authority Is Not Intention

"Just because you can, doens't mean you should". Granting a UCAN to a peer means that they are allowed to perform some actions for a period of time. This is a feature not a bug, but also says nothing about the intention of when it should be run. I may grant a collaborative process the ability to perform actions on an ongoing basis (hmm, but vioating POLA)

I could preload the service with `n` UCANs with narrow time windows and `nbf`s in the future. That would certainly be very secure, but it would be less convenient since I need to come online to issue more or to 

# 2 Roles

Invocation adds two new roles to UCAN: invoker and executor. The existing UCAN delegator and delgate pricipals MUST persist to the invocation.

| UCAN Field | Delegation                  | Invocation              |
|------------|-----------------------------|-------------------------|
| `iss`      | Transfer authority (active) | Request action (active) |
| `aud`      | Receive authority (passive) | Perform action (active) |

## 2.1 Invoker

The invoker signals to the executor that a task associated with a UCAN SHOULD be performed.

The invoker MUST be the UCAN delegator. Their DID MUST be authenticated in the `iss` field of the contained UCAN.

## 2.2 Executor

The executor is directed to perform some task described in the UCAN by the invoker.

The executor MUST be the UCAN delegate. Their DID MUST be set the in `aud` field of the contained UCAN.

# 3 Envelope

<!-- 

!!! Irakli thinks this envelope is not required -- FIXME either clearly justify it or cut it !!!
Part of this may be lazy vs eager semantics.

In a referentially transparent lazy langauge (Haskell), you can always substitute the value of the output for the expression, so there's no distinction between "running a function" and a value.

In an eager langauge (Javascript) there's a difference between passing around the reference to a function, and actually running it (by adding parentheses to it)
-->

The invocation envelope is a thin wrapper around a UCAN that conveys that all of the contained UCAN actions SHOULD be performed.

All of the roles from the referenced UCANs MUST persist to the invocation per the [above table](FIXME).

The invocation wrapper MUST be signed by the same principal that issued the UCAN.

## 3.1 Fields

| Field         | Type                               | Value     | Descrption                                       | Required | Default |
|---------------|------------------------------------|-----------|--------------------------------------------------|----------|---------|
| `ucan/invoke` | `CID`                              |           | The CID of the UCAN to invoke                    | Yes      |         |
| `v`           | `SemVer`                           | `"0.1.0"` | SemVer of the UCAN invocation object schema      | Yes      |         |
| `run`         | `"*" or {URI : {Ability : [Any]}}` |           | Which UCAN capabilities to run                   | No       | `"*"`   |
| `nnc`         | `String`                           |           | A unique nonce, to distinguish each invocation   | Yes      |         |
| `ext`         | `Any`                              |           | Nonnormative extended fields                     | No       | `null`  |
| `sig`         | `Bytes`                            |           | Signature of the rest of the field canonicalized | Yes      |         |
  
### 3.1.1 Invocation

The `ucan/invoke` field MUST contain CIDs pointing to the UCANs to invoke. The outmost UCAN being invoked MUST NOT contain any actions that are not intended to be executed.

### 3.1.2 Version

The `v` field MUST contain the version of the invocation object  schema.

### 3.1.3 Capabilities

The OPTIONAL `run` field specifies which actions contained in the UCAN are to be run during the invocation. To run all actions in the underlying UCAN, the `"*"` value MUST be used. If only specific actions (or [pipelines](FIXME)) are intended to be run, then they MUST be treated as a UCAN attenuation: all actions MUST be backed by a matching capability of equal or lesser scope.

### 3.1.4 Nonce

The `nnc` field MUS contain a unique nonce. This field makes each invocation unique.

### 3.1.5 Extended Fields

The OPTIONAL `ext` field MAY contain arbitrary data. If not present, the `ext` field MUST me interpreted as `null`, including for [signature](#315-signature).

### 3.1.6 Signature

The `sig` field MUST contain the signature of the other fields. To construct the payload, the other fields MUST first be canonicalized as `dag-cbor`.

If serialzied as JSON, the `sig` field MUST be serialized as [unpadded base64url](https://datatracker.ietf.org/doc/html/rfc4648#section-5).

## 3.2 IPLD Schema

``` ipldsch
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

### 3.3.1 Simple

 ``` json
{
  "ucan/invoke": "bafyUCAN",
  "v": "0.1.0",
  "nnc": "abcdef",
  "ext": null,
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
}
```

### 3.3.2 Internally Pipelined

 ``` json
{
  "ucan/invoke": "bafyUCAN",
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
          "status": {"ucan/promise": ["/", "dns://example.com?TYPE=TXT", "crud/update", "http/statusCode"]}},
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
            {"ucan/promise": ["/", "mailto://alice@example.com", "msg/send", null]}
            {"ucan/promise": ["/", "https://example.com", "crud/update", null]}
          ]
        }
      ]
    }
  },
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
}
```

### 3.3.3 Externally Pipelined

 ``` json
{
  "ucan/invoke": "bafyUCAN",
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
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT-SRK6v_SX8bjt-VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
}
 
{
  "ucan/invoke": "bafyUCAN",
  "v": "0.1.0",
  "nnc": "abcdef",
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
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT+SRK6v/SX8bjt+VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
}
```

# 4 Receipt

An invocation receipt is an attested result about the output of an invocation. A receipt MUST be attested via signature of the principal (the `aud` of the associated UCAN).

Note that this does not guarantee correctness of the result! The statement's veracity MUST be only understood as an attestation from the executor.

## 4.1 Fields

| Field          | Type                     | Descrption                                        | Required | Default |
|----------------|--------------------------|---------------------------------------------------|----------|---------|
| `ucan/receipt` | `CID`                    | CID of the Invocation that generated this respose | Yes      |         |
| `rlt`          | `{URI : {Ability: Any}}` | The results of each call                          | Yes      |         |
| `v`            | `SemVer`                 | SemVer of the UCAN invocation object schema       | Yes      |         |
| `nnc`          | `String`                 | A unique nonce, to distinguish each receipt       | Yes      |         |
| `ext`          | `Any`                    | Non-normative extended fields                     | No       | `null`  |
| `sig`          | `Bytes`                  | Signature of the rest of the field canonicalized  | Yes      |         |

## 4.1 IPLD

``` ipldsch
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
    "https://example.com": {
      "msg/read": [{
        "from": "alice@example.com",
        "text": "Hello world!"
      }, {
        "from": "bob@example.com",
        "text": "What's up?"
      }]
    },
    "dns://sub.example.com?TYPE=TXT": {
      "crud/update": {
        "12345": {
          "http": { 
            "status": 200
          },
          "value": "lorem ipsum"
        }
      }
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

The authority to execute later actions often cannot be fully attenuated in advance, since the executor controls the reported output of the prior step in a pipeline. When chosing to use pipelining, the invoker MUST delegate capabilities for any of the possible outputs. If tight control is required to limit authority, pipelining SHOULD NOT be used.

A promise MUST contain the CID of the target invocation, and the path of the 

becaus eth eresource may have a path in it, the resource needs to be broken out!

Inverts the version from the outer invocation

## 5.1 Fields

| Field           | Type      | Description                     | Required |
|-----------------|-----------|---------------------------------|----------|
| `ucan/promised` | `CID`     | The invocation being referenced | Yes      |
| `using`         | `URI`     | The resource called             | Yes      |
| `called`        | `Ability` | The ability used                | Yes      |
| `path`          | `Path`    | Path to the specific output     | Yes      |

``` json
```

the  know the concrete value required to scope the resource down sufficiently. This MAY be caused either by invoking them both in the same payload, or following one after another by CID reference.

Variables relative to the result of some other action MAY be used. In this case, the attested (signed) receipt of the previous action MUST be included in the following form:

Refeernced by invocation CID


## 5.2 IPLD Schema

``` ipldsch
type Promise struct {
  pse &Invocation
  usg URI
  cll Ability -- output path
  pth Path
} representation tuple
```

## 5.3 JSON Example

``` json
// IPLD
["bafyInvocation", "example.com/foo/bar", "http/put", "http/statusCode"]

// Full struct representation
{ 
  "ucan/promise": "bafyInvocation",
  "using": "example.com/foo/bar",
  "called": "http/put",
  "path": "http/status"
}
```

# 6 Appendix

## 6.1 Support Types

``` ipldsch
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
