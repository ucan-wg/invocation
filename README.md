# UCAN Invocation

## Editors

* _

## Authors

* _

# 0 Abstract

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# 1 Rough Proposal

THIS DOCUMENT IS CURRENTLY A VERY ROUGH PROPOSAL, USED FOR DESIGN COMMUNICATION WITH COLLABORATORS ONLY

THIS IS HEAVILY PRE-ALPHA. DO NOT USE THIS.


UCAN uses a capabilities model. The 

* UCANTO
* zcap


https://noti.st/expede/oq0ULd/ipvm-interplanetary-vm#smD6TZT

## Invocation Format

```
                   ┌───────────┐
                   │           │
                   │  Receipt  │
                   │           │
                   └┬─────────┬┘
                    │         │
                    │         │
                    │         │
   ┌────────────────▼┐      ┌─▼──────────┐
   │                 │      │            │
   │    Invocation   │      │   Result   │
   │                 │      │            │
   └──────┬────┬─────┘      └────────────┘
          │    │
          │    │
          │    │
┌─────────▼┐  ┌▼───────────┐
│          │  │            │
│   UCAN   │  │ Scheduling │
│          │  │            │
└──────────┘  └────────────┘
```

## Receipts

