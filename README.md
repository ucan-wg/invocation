# UCAN Invocation Specification v0.1.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)

## Authors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
* [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

## Depends On

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

The invocation envelope is a thin wrapper around a UCAN that conveys that all of the contained UCAN actions SHOULD be performed.

All of the roles from the referenced UCANs MUST persist to the invocation per the [above table](FIXME).

The invocation wrapper MUST be signed by the same principal that issued the UCAN.

## 3.1 Fields

| Field         | Type     | Value     | Descrption                                       | Required |
|---------------|----------|-----------|--------------------------------------------------|----------|
| `ucan/invoke` | `CID`    |           | The CID of the UCAN to invoke                    | Yes      |
| `v`           | `SemVer` | `"0.1.0"` | SemVer of the UCAN invocation object schema      | Yes      |
| `nnc`         | `String` |           | A unique nonce, to distinguish each invocation   | Yes      |
| `sig`         | `Bytes`  |           | Signature of the rest of the field canonicalized | Yes      |
  
### 3.1.1 Invocation

The `ucan/invoke` field MUST contain CIDs pointing to the UCANs to invoke. The outmost UCAN being invoked MUST NOT contain any actions that are not intended to be executed.

### 3.1.2 Version

The `v` field MUST contain the version of the invocation object  schema.

### 3.1.3 Nonce

The `nnc` field MUS contain a unique nonce. This field makes each invocation unique.

### 3.1.4 Signature

The `sig` field MUST contain the signature of the other fields. To construct the payload, the other fields MUST first be canonicalized as `dag-cbor`.

If serialzied as JSON, the `sig` field MUST be serialized as [unpadded base64url](https://datatracker.ietf.org/doc/html/rfc4648#section-5).

## 3.2 IPLD Schema

``` ipldsch
type Invocation struct {
  inv &UCAN  -- To invoke
  v   SemVer -- Version
  nnc String -- Nonce
  sig Bytes  -- Signature
}
```

## 3.3 JSON Exmaple

 ``` json
{
  "ucan/invoke": "bafyUCAN",
  "v": "0.1.0",
  "nnc": "abcdef",
  "sig": "bdNVZn_uTrQ8bgq5LocO2y3gqIyuEtvYWRUH9YT+SRK6v/SX8bjt+VZ9JIPVTdxkWb6nhVKBt6JGpgnjABpOCA"
}
```

# 3 Receipt & Attestation

An invocation receipt is a claim about what the output of an invocation is. A receipt MUST be attested via signature of the principal (the audience of the associated invocation).

Note that this does not guarantee correctness of the result! The level of this statement's veracity MUST be ony taken that the signer claims this to be a fact.

The receipt MUST be signed with by the `aud` from the UCAN.

## 3.1 IPLD

``` ipldsch
type Receipt struct {
  inv {&UCAN : {URI : Any}} <!-- not sure if this actually works? My guess is that the link doesn't work because it should not  -->
  v   SemVer
  nnc String
  ext optional {String : Any}
  sig Bytes
}

type URI = String
```

## 3.2 JSON Example

``` json
{
  "ucan/receipt": {
    "bafyLeft": {
      "a": 42,
      "example.com": {
        "msg/read": [
          "from": "alice@example.com",
          "text": "hello world"
        ]
      }
    },
    "bafyRight": {
      "sub.example.com?TYPE=TXT": {
        "crud/update": {
          "12345": {
            "http": { 
              "status": 200
            },
            "value": "lorem ipsum"
          }
        }
      }
    }
  },
  "v": "0.1.0",
  "nnc": "xyz",
  "ext": {
    "notes": "wow, what a "
  },
  "sig": 0xB00
}
```

# 4 Promise Pipelining

> Machines grow faster and memories grow larger. But the speed of light is constant and New York is not getting any closer to Tokyo. As hardware continues to improve, the latency barrier between distant machines will increasingly dominate the performance of distributed computation. When distributed computational steps require unnecessary round trips, compositions of these steps can cause unnecessary cascading sequences of round trips
>
> Robust Composition, Mark Miller

At time of creation, a UCAN MAY not know the concrete value required to scope the resource down sufficiently. This MAY be caused either by invoking them both in the same payload, or following one after another by CID reference.

Variables relative to the result of some other action MAY be used. In this case, the attested (signed) receipt of the previous action MUST be included in the following form:

Refeernced by invocation CID


### 4.1 IPLD Schema

``` ipldsch
type RelativeOutput struct {
  i &Invocation
  o Path -- output path
} 

type Path String
```

### 4.2 JSON Example

Some alternates:

``` json
{ 
  "ucan/invoked": "bafy12345",
  "output": "example/a/b"
}

"ucan:out:bafy12345/example/a/b"


```

Inside a next UCAN, substitution of a previous unresolved step MUST be represented as:

``` js
{
  // ...,
  "att": {
    "example.com/$PATH": {
      "$PATH": { 
        "ucan/invoked": "bafy12345",
        "output": "example.com/a/b"
      }
    }
  }
}
```

NOTE security, becase the aud controls the receipt of the first part of the pipeline, they control anything under the example.com namespace


# 5 Appendix

## 5.1 Support Types

``` ipldsch
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

