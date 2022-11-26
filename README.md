# UCAN Invocation Specification v0.1.0

## Editors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)

## Authors

* [Brooklyn Zelenka](https://github.com/expede/), [Fission](https://fission.codes/)
* [Irakli Gozalishvili](https://github.com/Gozala), [DAG House](https://dag.house/)

# 0 Abstract

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1 Introduction

## Separation

Why separate the UCAN from the invocation format? Surely the UCAN itself already contains all the infomation required.

I would argue that this is not the case for a few reasons.

1. Authority is not intent to execute. Authority transfer is first-class in UCAN, other actions are not
2. Mixing the two levels means that invocation info will live with the token (as a proof) on subdelegations
3. Other detail, such as scheduling

A job request may be picked up by anyone, PRIOR to UCAN delegation. A handshake or matchmaking may need to be performed.

## Authority Is Not Intention

Or to put it another way: "just because you can, doens't mean you should". Granting a UCAN to a peer means that they are allowed to perform some actions for a period of time. This is a feature not a bug, but also says nothing about the intention of when it should be run. I may grant a collaborative process the ability to perform actions on an ongoing basis (hmm, but vioating POLA)

I could preload the service with `n` UCANs with narrow time windows and `nbf`s in the future. That would certainly be very secure, but it would be less convenient since I need to come online to issue more or to 

# 2 Envelope

UCAN uses a capabilities model. The 

* UCANTO
* zcap
* IPVM

 ``` json
{
  "ucan/invoke": [ "bafyLeft", "bafyRight", "bafyEnd" ]
  "version": "0.1.0",
  /* "nonce": "abcdef" -- I think the nonce inside each UCAN is sufficent? */
  "siganture": 0xCOFFEE
}
```

# 3 UCAN Pipelining

At time of creation, a UCAN MAY not know the concrete value required to scope the resource down sufficiently. This MAY be caused either by invoking them both in the same payload, or following one after another by CID reference.
