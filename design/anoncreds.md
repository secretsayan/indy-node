# Anoncreds Design
Here you can find the requirements and design for Anoncreds workflow (including revocation).

* [Anoncreds References](#anoncreds-references)
* [Requirements](#requirements)
* [Referencing Schemas, CredDefs and RevocRegs](#referencing-schemas,-creddefs-and-revocregs)
* [SCHEMA](#schema)
* [CRED_DEF](#cred_def)
* [REVOC_REG_DEF](#revoc_reg_def)
* [REVOC_REG_ENTRY](#revoc_reg_entry)
* [Timestamp Support in State](#timestamp-support-in-state)
* [GET_OBJ](#get_obj)
* [Issuer Key Rotation](#issuer-key-rotation)

## Anoncreds References

Anoncreds protocol links:
- [Anoncreds Sequence Diagram](https://github.com/hyperledger/indy-sdk/blob/master/doc/libindy-anoncreds.svg)
- [Anoncreds Protocol Math](https://github.com/hyperledger/indy-crypto/blob/master/libindy-crypto/docs/AnonCred.pdf)
- [Anoncreds Protocol Crypto API](https://github.com/hyperledger/indy-crypto/blob/master/libindy-crypto/docs/anoncreds-design.md)

## Requirements
1. Creation of Schemas:
    1. Schema Author needs to be able to create multiple schemas by the same issuer DID.
    1. Schema Author needs to be able to evolve schemas by adding new attributes (a new Schema with a new name/version 
    can be created). 
    1. No one can modify existing Schemas.
    1. We need to keep reputation for Schema's Issuer DID.
    1. We should not have any semver assumptions for Schema's version by the Ledger.
1. Creation of Cred Def:
    1. CredDef Issuer may not be the same as Schema Author.
    1. CredDef Issuer needs to be able to create multiple CredDefs by the same issuer DID.
    1. CredDef Issuer needs to be able to create multiple CredDefs for the same Schema by the same issuer DID.
    1. We need to keep reputation for CredDef's Issuer DID.
1. Creation of Revocation entities (Def and Registry):
    1. RevocRegDef Issuer may not be the same as Schema Author and CredDef issuer. 
    1. RevocRegDef Issuer needs to be able to create multiple RevocRegDefs for the same issuer DID.
    1. RevocRegDef Issuer needs to be able to create multiple RevocRegDef for the same CredDef by the same issuer DID.
    1. We need to keep reputation for RevocRegDef's Issuer DID.
1. Referencing Schema/CredDef/RevocRegDef:
    1. Prover needs to know what CredDef (public keys), Schema and RevocRegDef 
    were used for issuing the credential.  
    1. Verifier needs to know what CredDef (public keys), Schema and RevocRegDef 
    were used for issuing the credential from the proof.
    1. The reference must be compact (a single value).
1. Keys rotation:
    1. Issuer needs to be able to rotate the keys and issue new credentials with the new keys.
    1. Issuer needs to be able to rotate the keys using the same Issuer DID.
    1. Only CredDef's Issuer can modify existing CredDef (that is rotate keys).
1. Validity of already issued credentials when key is compromised: 
    1. If the Issuer's key is compromised and the issuer suspects that it's compromised 
    from the very beginning, then the Issuer should be able to rotate the key so that all issued credentials
    becomes invalid.
    All new credentials issued after rotation should be verifiable against the new key.
    1. If the Issuer published a key at time A, and at time C he realised that the key was compromised at time B (A < B < C), 
    then the Issuer should be able to rotate the key so that all credentials
    issued before time B can be successfully verified using old key, and
    all credentials issued between B and C becomes invalid.
    All new credentials issued after C should be verifiable against the new key.
    1. The Issuer needs to be able to rotate the keys multiple times. Requirement 5.ii must be true for each key rotation.  
1. Revocation
    1. Verifier needs to be able to Verify that the credential is not revoked at the current time
    (the time when proof request is created).
    1. Verifier needs to be able to Verify that the credential is not revoked at the given time (any time in the past).       
1. Querying
    1. One needs to be able to get all CRED_DEFs created by the given Issuer DID.
    1. One needs to be able to get all SCHEMAs created by the given Issuer DID.
    1. One needs to be able to get all SCHEMAs with the given name created by the given Issuer DID.
    1. One needs to be able to get all SCHEMAs with the given name and version created by the given Issuer DID.
    1. One needs to be able to get all REVOC_REG_DEFs created by the given Issuer DID.
    1. One needs to be able to get all REVOC_REG_DEFs created by the given Issuer DID for the given CRED_DEF.    

## Referencing Schemas, CredDefs and RevocRegs

<b>Reqs 4</b>

The proposed solution is to identify entities by a unique `id`.
 * State-Trie-based key will be used as `id`. 
 This is the actual key used to store the entities in Patricia Merkle State Trie.
 * It can be deterministically calculated from the primary key tuples (by the client).
 * This single value will be used in anoncreds protocol (in libindy) to identify entities.
 * This ID should be included in all transactions as a new field (`id`).
 * We may expect changes in the format of this field, so it's not just address (key)
 in the State Trie, but can be a descriptive identifier.
 * The `id` attribute is calculated by the client and send as a part of SCHEMA/CRED_DEF/REVOC_REG_DEF txns.
 The ledger doesn't use this key as it is for storing the data in the State Trie.
 The ledger also calculates the key on its side from the primary key tuples, compares it with the provided `id`,
 and orders it only if they match. The request is rejected if they don't match.
 * We need to support [`GET_OBJ`](.get_obj) request to get the current state of the Schema/CredDef/RevocReg by its `id`.

      
## SCHEMA 

<b>Reqs 1, 4, 8</b>

#### SCHEMA txn
```
{
    "data": {
        "id":"L5AD5g65TDQr1PPHHRoiGf1Degree1.0",
        "attrNames": ["undergrad","last_name","first_name","birth_date","postgrad","expiry_date"],
        "name":"Degree",
        "version":"1.0",
    },
    
    "reqMetadata": {
        "submitterDid":"L5AD5g65TDQr1PPHHRoiGf",
        .....
    },
    
....
}
```

#### Restrictions

* Existing Schema (identified by the Schema `id`) can not be modified/changed/evolved.
A new Schema with a new name/version needs to be issued if one needs to evolve it.
* Only the `submitterDid` who created the Schema can modify it (that is we need to keep the ownership).
* `id` field must match the State Trie key (address) for this Schema.

#### State

* key: `schemasubmitterDid | SchemaMarker | schemaName | schemaVersion` 
* value: aggregated txn `data` and `txnMetadata` (as in ledger)


#### GET_SCHEMA
```
{
    "data": {
        "submitterDid":"L5AD5g65TDQr1PPHHRoiGf",
        "name":"Degree",
        "version":"1.0",
    },
...
}
```


## CRED_DEF

<b>Reqs 2, 4, 5, 8</b>

The Definition of credentials for the given Schema by the given Issuer.
It contains public keys (primary and revocation),
signature type and reference to the Schema. 

#### CRED_DEF txn
```
{
    "data": {
        "id":"HHAD5g65TDQr1PPHHRoiGf2L5AD5g65TDQr1PPHHRoiGf1Degree1CLkey1",
        "publicKeys": {},
        "schemaId":"L5AD5g65TDQr1PPHHRoiGf1Degree1",
        "signatureType":"CL",
        "tag": "key1",
    },
    
    "reqMetadata": {
        "submitterDid":"HHAD5g65TDQr1PPHHRoiGf",
        .....
    },
    
....
}
```
 

#### Restrictions

* Existing CredDef (identified by the CredDef `id`) can be modified/changed/evolved.
That is rotation of keys is supported.
* Only the `submitterDid` created the CredDef can modify it (that is we need to keep the ownership). 
* `id` field must match the State Trie key (address) for this CredDef.

#### State

* key: `credDefsubmitterDid | CredDefMarker | schemaID | signatureType | credDefTag` 
* value: aggregated txn `data` and `txnMetadata` (as in ledger)



 
#### GET_CRED_DEF
```
{
    "data": {
        "submitterDid":"L5AD5g65TDQr1PPHHRoiGf",
        "schemaId":"L5AD5g65TDQr1PPHHRoiGf1Degree1",
        "signatureType":"CL",
        "tag": "key1"
    },
...
}
```

### REVOC_REG_DEF

<b>Reqs 3, 4, 8</b>

The Definition of revocation registry for the given CredDef.
It contains public keys, maximum number of credentials the registry may contain,
reference to the CredDef, plus some revocation registry specific data.

#### REVOC_REG_DEF txn
```
{
    "data": {
        "id":"MMAD5g65TDQr1PPHHRoiGf3HHAD5g65TDQr1PPHHRoiGf2L5AD5g65TDQr1PPHHRoiGf1Degree1CLkey1CL_ACCUMreg1",
        "type":"CL_ACCUM",
        "credDefId":"HHAD5g65TDQr1PPHHRoiGf2L5AD5g65TDQr1PPHHRoiGf1Degree1CLkey1",
        "tag": "reg1",
        "maxCredNum": 1000000,
        "publicKeys": {},
        "metadata": {
            "tailsHash": "<SHA256 hash>",
            "tailsLocation": "<URL>"
        }
    },
    
    "reqMetadata": {
        "submitterDid":"MMAD5g65TDQr1PPHHRoiGf",
        .....
    },
    
....
}
```
 

#### Restrictions

* Existing RevocRegDef (identified by the RevocRegDef `id`) can be modified/changed/evolved.
That is rotation of keys is supported.
* Only the `submitterDid` who created the RevocRegDef can modify it (that is we need to keep the ownership). 
* `id` field must match the State Trie key (address) for this RevocRegDef.

#### State

* key: `revocDefSubmitterDid | RevocRegMarker | credDefID | revocDefType | revocDefTag` 
* value: aggregated txn `data` and `txnMetadata` (as in ledger)



#### GET_REVOC_REG_DEF
```
{
    "data": {
        "submitterDid":"MMAD5g65TDQr1PPHHRoiGf",
        "type":"CL_ACCUM",
        "credDefId":"HHAD5g65TDQr1PPHHRoiGf2L5AD5g65TDQr1PPHHRoiGf1Degree1CLkey1",
        "tag": "reg1",
    },
...
}
```


### REVOC_REG_ENTRY

<b>Reqs 3, 4, 8</b>

The delta of the RevocReg current state (accumulator, issued and revoked indices, etc.).

#### REVOC_REG_ENTRY txn
```
{
    "data": {
        "id":"MMAD5g65TDQr1PPHHRoiGf4MMAD5g65TDQr1PPHHRoiGf3HHAD5g65TDQr1PPHHRoiGf2L5AD5g65TDQr1PPHHRoiGf1Degree1CLkey1CL_ACCUMreg1",
        "type": "<issued by default or not>",
        "revocRegId": "MMAD5g65TDQr1PPHHRoiGf3HHAD5g65TDQr1PPHHRoiGf2L5AD5g65TDQr1PPHHRoiGf1Degree1CLkey1CL_ACCUMreg1"
        "accum":"<accum_value>",
        "issued": [], (optional)
        "revoked": [],
    },
    
    "reqMetadata": {
        "submitterDid":"MMAD5g65TDQr1PPHHRoiGf",
        .....
    },
    
....
}
```
* `uuid` must match an existing `REVOC_REG_DEF` with the given UUID.
* `type`: issuance by default (`issued` will be empty in this case assuming that everything is already issued)
or issuance by request (`issued` will be fulfilled by newly issued indices).
* `issued`: an array of issued indices (may be absent/empty if the type is "issuance by default"); this is delta; will be accumulated in state.
* `revoked`: an array of revoked indices (delta; will be accumulated in state)

#### Restrictions

* Existing RevocRegEntry (identified by the RevocRegEntry's `id`) can be modified/changed/evolved.
* Only the `submitterDid` who created the corresponding `REVOC_REG_DEF` can modify it. 


#### State

* key: `revocDefSubmitterDid | RevocRegEntryMarker | revocRegId`

* value: aggregated txn `data` and `txnMetadata` (as in ledger); 
contains aggregated aggregated accum_value, issued and revoked arrays.


#### GET_REVOC_REG_ENTRY
```
{
    "data": {
        "revocRegId": "MMAD5g65TDQr1PPHHRoiGf3HHAD5g65TDQr1PPHHRoiGf2L5AD5g65TDQr1PPHHRoiGf1Degree1CLkey1CL_ACCUMreg1",
        "timestamp": 20,
    },
...
}
```
See next sections on how to get the state for the given `timestamp`. 

## Timestamp Support in State

<b>Reqs 6, 7</b>

We need to have a generic way to get the State at the given time.
- State Trie allows to go to the past, that is given the root hash, get the state with this root.
- We may have a mapping of each State update (timestamp) to the corresponding root.
- We need to find a data structure that can help us to find the nearest state timestamp (to get the root) for the given
time.
- So, we will be able to get the data (state) at the given time.

This approach can be used for
* getting `REVOC_REG_ENTRY` at desired time (the same for both proper and verifier),
possibly long ago in the past;
* dealing with Requirement 6. 

## GET_OBJ

<b>Reqs 4</b>

`GET_OBJ` request can be used to get the current state of the Object by its `id` (the key in the State Trie).
```
{
    "data": {
        "id": "L5AD5g65TDQr1PPHHRoiGf1Degree1",
    },
...
}
```

## Issuer Key Rotation

<b>Reqs 6</b>

#### Changes in Anoncreds Protocol

If want to support Requirement 6, then the following changes are required in the
anoncerds protocol:

* Each Credential must have a reserved mandatory attribute: `issuanceTime`.
    * It's set by the Issuer to specify the time of Issuance.
    * It's needed to fulfill Requirements 5.
* This attribute can be considered as `m3` special attribute (`m1` is master secret, `m2` is credential context, `m3` is issuance time).
* Since the Issuer may be malicious (if keys were compromised already), then 
a proof that `issuanceTime` is really the current time and not the time from the past is needed.
    * We can use our blockchain to prepare such a proof.
    * Issuer signs (by his Cred Def's public key) the `issuanceTime` and sends it to the pool.
    * Each node verifies that `issuanceTime` is not less that the current one, and signs the result with BLS key.
    * Each node then sends the signed result to the Issuer (no need to write anything to the ledger).
    * The issuer prepares a BLS multi-signature (making sure that there is a consensus)
    and adds the BLS-signed proof of the `issuanceTime` to the credential.
    * The verifier will then use the proof to make sure that the `issuanceTime` is really the correct one.
* The `issuanceTime` needs to be verified in each proof.
    * The Verifier should use Predicates (instead of disclosing) for the value of `issuanceTime`
    to avoid correlation. 
    * It's possible also to disclose `issuanceTime`, but we don't force it.
    * If it's not disclosed and not verified as a Predicate, then there is a chance the the proof verification will fail because 
of key rotations, since the latest keys will be used.

#### Changes in Transactions
A new field `oldKeyTrustTime` needs to be added to `CRED_DEF` and `REVOC_REF_DEF` txns.
`oldKeyTrustTime` can be set each time a key is rotated and defines
 `the intervals when we still can trust the previous value of the key`. 
It is needed to deprecate credentials issued during the time when we suspect
the keys were stolen.
We can not always use revocation to deprecate old credentials, since revocation keys can
be stolen as well.  


#### How oldKeyTrustTime works
Let's assume that 
* `key1` was issued at `timeA`
* `key2` was issued at `timeC`, and we suspect that `key1` is stolen at `timeB`
* `key3` was issued at `timeE`, and we suspect that `key2` is stolen at `timeD`

So, we need to use (and return by `GET_CRED_DEF`) the following keys, depending on 
the interval the `issuanceTime` belongs to:
* [A,B] -> key1
* [B,C] -> key2 
* [C,D] -> key2
* [D,E] -> key3
* [E, current] -> key3

So, the Credentials issued at intervals [B,C] and [D,E], that is at intervals
when keys are suspicious, will not be verifiable anymore, because they were issued using key1 (key2),
but the Verifies will use key2 (key3) for verification (as returned by `GET_CRED_DEF`). 

The following txns will be put on Ledger:
1. At `timeA`:
 ```
 "data": {
        "uuid":"TYzcdDLhCpGCYRHW82kjHd",
        "publicKeys": {key1},
        "schemaRef":"GEzcdDLhCpGCYRHW82kjHd",
        "signatureType":"CL",
    },
```
2. At `timeC`:   
 ```
 "data": {
        "uuid":"TYzcdDLhCpGCYRHW82kjHd",
        "publicKeys": {key2},
        "schemaRef":"GEzcdDLhCpGCYRHW82kjHd",
        "signatureType":"CL",
        "oldKeyTrustTime": {          
            ("from": "A", "to": "B"),
            ("from": "C"),
        }
    },
```
3. At `timeE`:   
 ```
 "data": {
        "uuid":"TYzcdDLhCpGCYRHW82kjHd",
        "publicKeys": {key3},
        "schemaRef":"GEzcdDLhCpGCYRHW82kjHd",
        "signatureType":"CL",
        "oldKeyTrustTime": {          
            ("from": "C", "to": "D"),
            ("from": "E"),
        }
    },
```


The current state (Record1) will look the following:
* key: `HHAD5g65TDQr1PPHHRoiGf|CRED_DEF|GEzcdDLhCpGCYRHW82kjHd|CL|TYzcdDLhCpGCYRHW82kjHd`
* value:
 ```
 ....
 "data": {
        "uuid":"TYzcdDLhCpGCYRHW82kjHd",
        "publicKeys": {key3},
        "schemaRef":"GEzcdDLhCpGCYRHW82kjHd",
        "signatureType":"CL",
        "oldKeyTrustTime": {          
            ("from": "A", "to": "B"),
            ("from": "C", "to": "D"),
            ("from": "E"),
        }
    },
 ....
```

#### Changes in GET request

An optional field `issuanceTime` needs to be supported for 
* `GET_CRED_DEF`
* `GET_REVOC_REG_DEF`
* `GET_OBJ`.

There is a special logic to get the valid and trusted value of the keys
depending on the issuance time:
1. Lookup State Trie to get the current state by the key (either `id` in case of `GET_OBJ` or a key created
from primary key fields provided in the request).
1. If no `issuanceTime` provided, then just return the current value.  
1. Try to find the interval (in `oldKeyTrustTime` array) the `issuanceTime` belongs to.
    * If it's greater than the most right interval, then return the current value.
    * If it belongs to an interval, then get the left value (`from`) of the interval.
    * If it's in between intervals, then get the right interval, and get the left value (`from`)
of this interval.
1. Use generic logic to get the root of the State trie at the time `to` found above. 
1. Lookup State Trie with the found root to find the state at that time (the same way as in Step 1)

So, we will have not more than 2 lookups for each request. 

Result for the Example above:
* `issuanceTime < A` => [A,B] => state at timeA => key1 (OK)
* `A <= issuanceTime <= B` => [A,B] => state at timeA => key1 (OK)
* `B < issuanceTime < C` => [C,D] => state at timeC => key2 (deprecated credentials)
* `C <= issuanceTime <= D` => [C,D] => state at timeC => key2 (OK)
* `D < issuanceTime < E` => [E,...] => state at timeE => key3 (deprecated credentials)
* `issuanceTime > E` => [E,...] => state at timeE => key3 (OK)