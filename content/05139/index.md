---
eip: 5139
title: Remote Procedure Call Provider Lists
description: Format for lists of RPC providers for Ethereum-like chains.
author: Sam Wilson (@SamWilsn)
discussions-to: https://ethereum-magicians.org/t/eip-5139-remote-procedure-call-provider-lists/9517
status: Stagnant
type: Standards Track
category: ERC
created: 2022-06-06
requires: 155, 1577
---

## Abstract
This proposal specifies a JSON schema for describing lists of remote procedure call (RPC) providers for Ethereum-like chains, including their supported [EIP-155](../00155.md) `CHAIN_ID`.

## Motivation
The recent explosion of alternate chains, scaling solutions, and other mostly Ethereum-compatible ledgers has brought with it many risks for users. It has become commonplace to blindly add new RPC providers using [EIP-3085](../03085.md) without evaluating their trustworthiness. At best, these RPC providers may be accurate, but track requests; and at worst, they may provide misleading information and frontrun transactions.

If users instead are provided with a comprehensive provider list built directly by their wallet, with the option of switching to whatever list they so choose, the risk of these malicious providers is mitigated significantly, without sacrificing functionality for advanced users.

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY" and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### List Validation & Schema

List consumers (like wallets) MUST validate lists against the provided schema. List consumers MUST NOT connect to RPC providers present only in an invalid list.

Lists MUST conform to the following JSON Schema:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",

  "title": "Ethereum RPC Provider List",
  "description": "Schema for lists of RPC providers compatible with Ethereum wallets.",

  "$defs": {
    "VersionBase": {
      "type": "object",
      "description": "Version of a list, used to communicate changes.",

      "required": [
        "major",
        "minor",
        "patch"
      ],

      "properties": {
        "major": {
          "type": "integer",
          "description": "Major version of a list. Incremented when providers are removed from the list or when their chain ids change.",
          "minimum": 0
        },

        "minor": {
          "type": "integer",
          "description": "Minor version of a list. Incremented when providers are added to the list.",
          "minimum": 0
        },

        "patch": {
          "type": "integer",
          "description": "Patch version of a list. Incremented for any change not covered by major or minor versions, like bug fixes.",
          "minimum": 0
        },

        "preRelease": {
          "type": "string",
          "description": "Pre-release version of a list. Indicates that the version is unstable and might not satisfy the intended compatibility requirements as denoted by its major, minor, and patch versions.",
          "pattern": "^[1-9A-Za-z][0-9A-Za-z]*(\\.[1-9A-Za-z][0-9A-Za-z]*)*$"
        }
      }
    },

    "Version": {
      "type": "object",
      "additionalProperties": false,

      "allOf": [
      {
        "$ref": "#/$defs/VersionBase"
      }
      ],

      "properties": {
        "major": true,
        "minor": true,
        "patch": true,
        "preRelease": true,
        "build": {
          "type": "string",
          "description": "Build metadata associated with a list.",
          "pattern": "^[0-9A-Za-z-]+(\\.[0-9A-Za-z-])*$"
        }
      }
    },

    "VersionRange": {
      "type": "object",
      "additionalProperties": false,

      "properties": {
        "major": true,
        "minor": true,
        "patch": true,
        "preRelease": true,
        "mode": true
      },

      "allOf": [
        {
          "$ref": "#/$defs/VersionBase"
        }
      ],

      "oneOf": [
        {
          "properties": {
            "mode": {
              "type": "string",
              "enum": ["^", "="]
            },
            "preRelease": false
          }
        },
      {
        "required": [
          "preRelease",
          "mode"
        ],

        "properties": {
          "mode": {
            "type": "string",
            "enum": ["="]
          }
        }
      }
      ]
    },

    "Logo": {
      "type": "string",
      "description": "A URI to a logo; suggest SVG or PNG of size 64x64",
      "format": "uri"
    },

    "ProviderChain": {
      "type": "object",
      "description": "A single chain supported by a provider",
      "additionalProperties": false,
      "required": [
        "chainId",
        "endpoints"
      ],
      "properties": {
        "chainId": {
          "type": "integer",
          "description": "Chain ID of an Ethereum-compatible network",
          "minimum": 1
        },
        "endpoints": {
          "type": "array",
          "minItems": 1,
          "uniqueItems": true,
          "items": {
            "type": "string",
            "format": "uri"
          }
        }
      }
    },

    "Provider": {
      "type": "object",
      "description": "Description of an RPC provider.",
      "additionalProperties": false,

      "required": [
        "chains",
        "name"
      ],

      "properties": {
        "name": {
          "type": "string",
          "description": "Name of the provider.",
          "minLength": 1,
          "maxLength": 40,
          "pattern": "^[ \\w.'+\\-%/À-ÖØ-öø-ÿ:&\\[\\]\\(\\)]+$"
        },
        "logo": {
          "$ref": "#/$defs/Logo"
        },
        "priority": {
          "type": "integer",
          "description": "Priority of this provider (where zero is the highest priority.)",
          "minimum": 0
        },
        "chains": {
          "type": "array",
          "items": {
            "$ref": "#/$defs/ProviderChain"
          }
        }
      }
    },

    "Path": {
      "description": "A JSON Pointer path.",
      "type": "string"
    },

    "Patch": {
      "items": {
        "oneOf": [
          {
            "additionalProperties": false,
            "required": ["value", "op", "path"],
            "properties": {
              "path": {
                "$ref": "#/$defs/Path"
              },
              "op": {
                "description": "The operation to perform.",
                "type": "string",
                "enum": ["add", "replace", "test"]
              },
              "value": {
                "description": "The value to add, replace or test."
              }
            }
          },
          {
            "additionalProperties": false,
            "required": ["op", "path"],
            "properties": {
              "path": {
                "$ref": "#/$defs/Path"
              },
              "op": {
                "description": "The operation to perform.",
                "type": "string",
                "enum": ["remove"]
              }
            }
          },
          {
            "additionalProperties": false,
            "required": ["from", "op", "path"],
            "properties": {
              "path": {
                "$ref": "#/$defs/Path"
              },

              "op": {
                "description": "The operation to perform.",
                "type": "string",
                "enum": ["move", "copy"]
              },
              "from": {
                "$ref": "#/$defs/Path",
                "description": "A JSON Pointer path pointing to the location to move/copy from."
              }
            }
          }
        ]
      },
      "type": "array"
    }
  },

  "type": "object",
  "additionalProperties": false,

  "required": [
    "name",
    "version",
    "timestamp"
  ],

  "properties": {
    "name": {
      "type": "string",
      "description": "Name of the provider list",
      "minLength": 1,
      "maxLength": 40,
      "pattern": "^[\\w ]+$"
    },
    "logo": {
      "$ref": "#/$defs/Logo"
    },
    "version": {
      "$ref": "#/$defs/Version"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "The timestamp of this list version; i.e. when this immutable version of the list was created"
    },
    "extends": true,
    "changes": true,
    "providers": true
  },

  "oneOf": [
    {
      "type": "object",

      "required": [
        "extends",
        "changes"
      ],

      "properties": {
        "providers": false,

        "extends": {
          "type": "object",
          "additionalProperties": false,

          "required": [
            "version"
          ],

          "properties": {
            "uri": {
              "type": "string",
              "format": "uri",
              "description": "Location of the list to extend, as a URI."
            },
            "ens": {
              "type": "string",
              "description": "Location of the list to extend using EIP-1577."
            },
            "version": {
              "$ref": "#/$defs/VersionRange"
            }
          },

          "oneOf": [
            {
              "properties": {
                "uri": false,
                "ens": true
              }
            },
            {
              "properties": {
                "ens": false,
                "uri": true
              }
            }
          ]
        },
        "changes": {
          "$ref": "#/$defs/Patch"
        }
      }
    },
    {
      "type": "object",

      "required": [
        "providers"
      ],

      "properties": {
        "changes": false,
        "extends": false,
        "providers": {
          "type": "object",
          "additionalProperties": {
            "$ref": "#/$defs/Provider"
          }
        }
      }
    }
  ]
}
```

For illustrative purposes, the following is an example list following the schema:

```json
{
  "name": "Example Provider List",
  "version": {
    "major": 0,
    "minor": 1,
    "patch": 0,
    "build": "XPSr.p.I.g.l"
  },
  "timestamp": "2004-08-08T00:00:00.0Z",
  "logo": "https://mylist.invalid/logo.png",
  "providers": {
    "some-key": {
      "name": "Frustrata",
      "chains": [
        {
          "chainId": 1,
          "endpoints": [
            "https://mainnet1.frustrata.invalid/",
            "https://mainnet2.frustrana.invalid/"
          ]
        },
        {
          "chainId": 3,
          "endpoints": [
            "https://ropsten.frustrana.invalid/"
          ]
        }
      ]
    },
    "other-key": {
      "name": "Sourceri",
      "priority": 3,
      "chains": [
        {
          "chainId": 1,
          "endpoints": [
            "https://mainnet.sourceri.invalid/"
          ]
        },
        {
          "chainId": 42,
          "endpoints": [
            "https://kovan.sourceri.invalid"
          ]
        }
      ]
    }
  }
}
```

### Versioning

List versioning MUST follow the [Semantic Versioning 2.0.0](./assets/semver.md) (SemVer) specification.

The major version MUST be incremented for the following modifications:

 - Removing a provider.
 - Changing a provider's key in the `providers` object.
 - Removing the last `ProviderChain` for a chain id.

The major version MAY be incremented for other modifications, as permitted by SemVer.

If the major version is not incremented, the minor version MUST be incremented if any of the following modifications are made:

 - Adding a provider.
 - Adding the first `ProviderChain` of a chain id.

The minor version MAY be incremented for other modifications, as permitted by SemVer.

If the major and minor versions are unchanged, the patch version MUST be incremented for any change.

### Publishing

Provider lists SHOULD be published to an Ethereum Name Service (ENS) name using [EIP-1577](../01577.md)'s `contenthash` mechanism on mainnet.

Provider lists MAY instead be published using HTTPS. Provider lists published in this way MUST allow reasonable access from other origins (generally by setting the header `Access-Control-Allow-Origin: *`.)

### Priority

Provider entries MAY contain a `priority` field. A `priority` value of zero SHALL indicate the highest priority, with increasing `priority` values indicating decreasing priority. Multiple providers MAY be assigned the same priority. All providers without a `priority` field SHALL have equal priority. Providers without a `priority` field SHALL always have a lower priority than any provider with a `priority` field.

List consumers MAY use `priority` fields to choose when to connect to a provider, but MAY ignore it entirely. List consumers SHOULD explain to users how their implementation interprets `priority`.

### List Subtypes

Provider lists are subdivided into two categories: root lists, and extension lists. A root list contains a list of providers, while an extension list contains a set of modifications to apply to another list.

#### Root Lists

A root list has a top-level `providers` key.

#### Extension Lists

An extension list has top-level `extends` and `changes` keys.

##### Specifying a Parent (`extends`)

The `uri` and `ens` fields SHALL point to a source for the parent list.

If present, the `uri` field MUST use a scheme specified in [Publishing](#publishing).

If present, the `ens` field MUST specify an ENS name to be resolved using EIP-1577.

The `version` field SHALL specify a range of compatible versions. List consumers MUST reject extension lists specifying an incompatible parent version.

In the event of an incompatible version, list consumers MAY continue to use a previously saved parent list, but list consumers choosing to do so MUST display a prominent warning that the provider list is out of date.

###### Default Mode

If the `mode` field is omitted, a parent version SHALL be compatible if and only if the parent's version number matches the left-most non-zero portion in the major, minor, patch grouping.

For example:

```javascript
{
  "major": "1",
  "minor": "2",
  "patch": "3"
}
```

Is equivalent to:

```
>=1.2.3, <2.0.0
```

And:

```javascript
{
  "major": "0",
  "minor": "2",
  "patch": "3"
}
```

Is equivalent to:

```
>=0.2.3, <0.3.0
```

###### Caret Mode (`^`)

The `^` mode SHALL behave exactly as the default mode above.

###### Exact Mode (`=`)

In `=` mode, a parent version SHALL be compatible if and only if the parent's version number exactly matches the specified version.

##### Specifying Changes (`changes`)

The `changes` field SHALL be a JavaScript Object Notation (JSON) Patch document as specified in RFC 6902.

JSON pointers within the `changes` field MUST be resolved relative to the `providers` field of the parent list. For example, see the following lists for a correctly formatted extension.

###### Root List

```json
TODO
```

###### Extension List

```json
TODO
```

##### Applying Extension Lists

List consumers MUST follow this algorithm to apply extension lists:

 1. Is the current list an extension list?
    * Yes:
       1. Ensure that this `from` has not been seen before.
       1. Retrieve the parent list.
       1. Verify that the parent list is valid according to the JSON schema.
       1. Ensure that the parent list is version compatible.
       1. Set the current list to the parent list and go to step 1.
    * No:
       1. Go to step 2.
 1. Copy the current list into a variable `$output`.
 1. Does the current list have a child:
    * Yes:
       1. Apply the child's `changes` to `providers` in `$output`.
       1. Verify that `$output` is valid according to the JSON schema.
       1. Set the current list to the child.
       1. Go to step 3.
    * No:
       1. Replace the current list's `providers` with `providers` from `$output`.
       1. The current list is now the resolved list; return it.


List consumers SHOULD limit the number of extension lists to a reasonable number.

## Rationale

This specification has two layers (provider, then chain id) instead of a flatter structure so that wallets can choose to query multiple independent providers for the same query and compare the results.

Each provider may specify multiple endpoints to implement load balancing or redundancy.

List version identifiers conform to SemVer to roughly communicate the kinds of changes that each new version brings. If a new version adds functionality (eg. a new chain id), then users can expect the minor version to be incremented. Similarly, if the major version is not incremented, list subscribers can assume dapps that work in the current version will continue to work in the next one.

## Security Considerations

Ultimately it is up to the end user to decide on what list to subscribe to. Most users will not change from the default list maintained by their wallet. Since wallets already have access to private keys, giving them additional control over RPC providers seems like a small increase in risk.

While list maintainers may be incentivized (possibly financially) to include or exclude particular providers, actually doing so may jeopardize the legitimacy of their lists. This standard facilitates swapping lists, so if such manipulation is revealed, users are free to swap to a new list with little effort.

If the list chosen by the user is published using EIP-1577, the list consumer has to have access to ENS in some way. This creates a paradox: how do you query Ethereum without an RPC provider? This paradox creates an attack vector: whatever method the list consumer uses to fetch the list can track the user, and even more seriously, **can lie about the contents of the list**.

## Copyright
Copyright and related rights waived via [CC0](/LICENSE.md).
