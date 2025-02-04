### Create Presentation Request

The [[ref: verifier]] starts the presentation process in step 1 of the [AnonCreds Presentation Data
Flow](#anoncreds-presentation-data-flow) by creating and sending a [[ref: presentation
request]] to the [[ref: holder]].

The [[ref: presentation request]] provides information about the attributes and
predicates the [[ref: verifier]] is asking the the [[ref: holder]] to reveal,
restrictions on what verifiable credentials can be the sources for the
attributes and predicates, and limitations on the freshness of the credential
revocation status. Presentation requests are defined at the "business logic"
layer, with any cryptographic processing applied. The verification process
includes verifications that the presentation satisfies the request. The [[ref:
verifier]] SHOULD validate that the presentation satisfies the business
requirements for which the presentation was provided.

In reading this section, the term `attribute` is used in two ways, and readers
should be aware of the context of each use. A presentation request has **requested
attributes that are to be included in the presentation provided from the [[ref:
holder]]. Those requested attributes in turn reference **attribute names and
values** from source verifiable credentials held by the [[ref: holder]].

The [[ref: presentation request]] is created by the [[ref: verifier]] in JSON format, as follows:

```json
{
    "name": string,
    "version": string,
    "nonce": string,
    "requested_attributes": {
        "<attr_referent>": <attr_info>,
        ...,
    },
    "requested_predicates": {
        "<predicate_referent>": <predicate_info>,
        ...,
     },
    "non_revoked": Optional<non_revoc_interval>,
    "ver": Optional<str>
}
```

* `name` is a string set by the [[ref: verifier]], a name for the [[ref: presentation request]].
* `version` is a string set by the [[ref: verifier]], the version of the [[ref: presentation request]]
* `nonce` is a string, a decimal, 80-bit number generated by the [[ref: verifier]] that
  SHOULD be unique per [[ref: presentation request]]. The nonce is included in the request
  to prevent replay attacks through its use in creating and verifying the presentation.
* `requested_attributes` specify the set of requested attributes
  * `attr_referent` is a verifier-defined identifier for the requested attribute(s) to be revealed.
  * `attr_info` describes a requested attribute. See [attr_info](#attr_info)
* `requested_predicates` specify the set of requested predicates
  * `predicate_referent` is a verifier-defined identifier for the requested predicate.
  * `predicate_info` describes a requested predicate. See [predicate_info](#predicate_info)
* `non_revoked` specifies an optional non-revocation interval
  `non_revoc_interval`. See the [Request Non-Revocation
  Proofs](#request-non-revocation-proofs) section.
* `ver` is an optional string, specifying the [[ref: presentation request]] version.
  * If omitted, "1.0" is used by default.
  * "1.0" to use unqualified identifiers for restrictions
  * "2.0" to use fully qualified identifiers for restrictions

<a id="attr_info"></a> `attr_info` has the following format:

```json
{
    "name": <string>,
    "names": <[string, string]>,
    "restrictions": <restrictions>,
    "non_revoked": <non_revoc_interval>,
}
```

All of the items are optional, but one of `name` or `names` MUST be included, and not both.

* `name` is a string, the name of an attribute from a source credential.
  * The name is case insensitive with spaces ignored.
* `names` is a array of strings, the names of attributes from a source
  credential
  * The names are case insensitive with spaces ignored.
  * The attribute names MUST be sourced from a single credential.
* `restrictions` is a condition on the source credential that can be used to
  satisfy this attribute request.
  * See [restrictions](#restrictions) for details about supported restrictions.
  * Omitting `restrictions` implies that:
    * The [[ref: holder]] MAY provide self-attested attributes for the request in the
    presentation, or
    * that the name(s) of the requested attributes match the name(s) of the
    claim(s) in the source verifiable credential.
* `non_revoked` specifies a non-revocation interval `non_revoc_interval` for
  this requested attribute.
  * See [Request Non-Revocation Proofs](#request-non-revocation-proofs) section.
  * If `non_revoked` is defined at the outer level of the JSON and is not
    defined at the `attr_info` level, the out level data applies to the
    attribute.
  * If `non_revoked` is defined at the outer level of the JSON AND at the
    `attr_info` layer, the `attr_info` data applies to the attribute.

<a id="predicate_info"></a> `predicate_info` has the following format:

```json
{
    "name": string,
    "p_type": string,
    "p_value": int,
    "restrictions": <restrictions>,
    "non_revoked": <non_revoc_interval>,
}
```

* `name` (required) is a string, the name of an attribute from a source
  credential to use in the predicate expression.
  * The name is case insensitive and spaces are ignored.
  * To be useful, the attribute in the source credential MUST be an integer, but
    that requirement cannot be enforced. The [[ref: verifier]] MUST understand how the attribute value is
    set by the issuer(s) in the expected source credentials.
* `p_type` is a string, the type of the predicate. Possible type values are
  [">=", ">", "<=", "<"].
* `p_value` is an integer value.
  * The boolean expression that is to proven in zero knowledge is evaluated as:
    "`<name> <p_type> <p_value>`." For example, to check an "older than" based
    on date of birth, the expression might be "`birth_dateint <= 20020116`"
* `restrictions` is a condition on the source credential that can be used to
  satisfy this predicate request.
  * See [restrictions](#restrictions) for details about supported restrictions.
  * If omitted, the only restriction on the requested predicate is that the
    `name` matches the attribute name in the source credential used to satisfy the
    predicate.
* `non_revoked` specifies a non-revocation interval `non_revoc_interval` for
  this predicate attribute.
  * See [Request Non-Revocation Proofs](#request-non-revocation-proofs) section.

#### Restrictions

The `restrictions` item on attributes (optional) and predicates (required) is a
JSON structure that forms a logical expression involving properties of the
source verifiable credential. The [[ref: holder]] must use source verifiable
credentials that satisfy the `restrictions` expression for each
attribute/predicate entry. Each element of the logic expression is a property of
source credentials and a value to be matched for that property. The following
properties can be specified in the JSON. All except the `marker` property is
specified with a value that must match the property. For the `marker` property,
the value is always `1`.

* `schema_id` - the identifier of the schema upon which the source credential is
  based.
* `schema_issuer_did` - the identifier, usually a [[ref: DID]], of the publisher
  of the schema upon which the source credential is based.
* `schema_name` - the `name` property of the schema upon which the source
  credential is based.
* `schema_version` - the `version` property of the schema upon which the source
  credential is based.
* `issuer_did` - the identifier, usually a [[ref: DID]], of the [[re: issuer]] of the
  source credential.
* `cred_def_id` - the identifier of the [[ref: Credential Definition]] of the
  source credential.
* `attr::<attribute-name>::marker` - an attribute `<attribute-name>`
  must exist in the source credential.
  * When used, the value of the JSON item must be "1".
* `attr::<attribute-name>::<attribute-value>` - the attribute `<attribute-name>`
  must be found in the source credential with a value of `<attribute-value>`.
  * When this property is used, the [[ref: verifer]] MUST request that the
  `<attribute-name>` be revealed as otherwise there is no way to be sure the
  restriction has been satisfied.

A boolean expression is formed by ORing and ANDing the source credential
properties. The following JSON is an example. Any of the source credential
properties listed above can be used in the expression components:


``` json
      "restrictions": [
        {
          "issuer_did": "<did>",
          "schema_id": "id"
        },
        {
          "cred_def_id" : "<id>",
          "attr::color::marker": "1",
          "attr::color::value" : "red"
        }
      ]
```

The properties in each list item are OR'd together, and the array elements are
AND'd together. As such, the example above defines the logical expression:

```text
The attributes must come from a source verifiable credential such that:
   issuer_did = <did> OR
     schema_id = <id>
   AND
   cred_def_id = <id>" OR
      the credential must contain an attribute name "color" OR
      the credential must contain an attribute name "color" with the attribute value "red"
```

#### Request Non-Revocation Proofs

The [[ref: presentation request]] JSON item `non_revoked` allows the [[ref:
verifier]] to define an acceptable non-revocation interval for a requested
attribute(s) / predicate(s), as follows:

```json
{
    "from": Optional<int>,
    "to": Optional<int>,
}
```

* `from` is an unsigned long long value, the [[ref: Unix Time]] timestamp of the
  interval beginning.
* `to`  is an unsigned long long value, the [[ref: Unix Time]] timestamp of the
  interval end.

As noted in the [[ref: presentation request]] specification above, a `non-revoked` item
be may at the outer level of the [[ref: presentation request]] such that it applies to
all requested attributes and predicates, and/or at the attribute/predicate level, applying
only to specific requested attributes and/or predicates and overriding the outer layer item.

The `non-revoked` items apply only to requested attributes/predicates in a presentation
that derive from revocable credentials. No proof of non-revocation is needed (or
possible) from credentials that cannot be revoked. Verifiers should be aware
that different issuers of the same credential type (same `schemaId`) may or may
not use revocation for the credentials they issue.

The use of a "non-revoke interval" was designed to have the semantic meaning
that the [[ref: verifier]] will accept a non-revocation Proof (NRP) from any
point in the `from` to `to` interval. The intention is that by being as flexible
as the business rules allow, the [[ref: holder]] and/or [[ref: verifier]] may
have cached [[ref: VDR]] revocation data such that they don't have to go to the
[[ref: VDR]] to get additional [[ref: RevRegEntry]] data. The verification of
the provided non-revocation interval in a [[ref: presentation request]] is
limited. For additional details, see the
[Verify Non-Revocation Proof](#verify-non-revocation-proof) section of this
specification.

In practice, the use of the interval is not well understood and tends to cause
confusion amongst those building [[ref: presentation requests]]. The AnonCreds
community recommends using matching `from` and `to` values as outlined in the
[Aries RFC 0441 Present Proof Best
Practices](https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0441-present-proof-best-practices#semantics-of-non-revocation-interval-endpoints).
The [[ref: verifier]] can then use business rules (outside of AnonCreds) to
decide if the revocation is sufficiently up to date.

While one might expect the `to` value to always be the current time ("Prove the
credential is not revoked now"), its inclusion allows the [[ref: verifier]] to
ask for a non-revocation proof sometime in the past. This addresses use cases
such as "Prove that your car insurance policy was not revoked on June 12, 2021
when the accident occurred."

#### Presentation Request Example

The following is an example of a full [[ref: presentation request]] for a presentation
for a set of revealed attribute names from a single source credential, a self-attested
attribute, and a predicate.

```json
{
   "nonce":"168240505120030101",
   "name":"Proof of Education",
   "version":"1.0",
   "requested_attributes":{
      "0_degree_uuid":{
         "names":[
            "name",
            "date",
            "degree"
         ],
         "restrictions":[
            {
               "schema_name":"degree schema"
            }
         ]
      },
      "0_self_attested_thing_uuid":{
         "name":"self_attested_thing"
      },
      "non_revoked": {
        "from": 1673885735,
        "to": 1673885735,
      }
   },
   "requested_predicates":{
      "0_age_GE_uuid":{
         "name":"birthdate_dateint",
         "p_type":"<=",
         "p_value":20030101,
         "restrictions":[
            {
               "schema_name":"degree schema"
            }
         ]
      }
   }
}
```

In step 2 of the [AnonCreds Presentation Data Flow](#anoncreds-presentation-data-flow),
the [[ref: verifier]] sends the [[ref: presentation request]] to the [[ref: holder]].
