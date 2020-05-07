# IdentityLink in RTB

## Overview
This document discusses technical means for enabling programmatic supply and demand partners with IdentityLinks (IDLs) in bid requests.

## Components
### Identity Envelopes
Envelopes are small, opaque base64-encoded strings, encrypted with a single-use [initialization vector](https://en.wikipedia.org/wiki/Initialization_vector), such that each generation of an envelope is unique regardless of its content. They are intended to represent IdentityLinks without exposing those links to the Envelope bearer; only trusted parties are provided the means to decrypt the envelope and access underlying links.

This makes envelopes suitable for protecting and passing IdentityLinks through public channels such as web APIs and shared cookie spaces. As intentionally opaque payloads, envelopes also provide a joint for future extensibility: LiveRamp can coordinate modifications to the internal encoded structure without disrupting partner integrations.

Envelopes are key to integrating IDLs into programmatic requests, and we plan for multiple and overlapping means of obtaining Envelopes. Integrating partners would select the most appropriate means of integration for their systems.

### Obtaining Identity Envelopes
#### Via Mapping Files
LiveRamp will provide file-based feeds of mappings between Consortium device identifiers ⇔ Identity Envelopes, as well as Mobile AdID ⇔ Identity Envelopes for partners electing to host their own key-value infrastructure.

#### Via CORS API

LiveRamp will provide a CORS API with open origin permissions to return an envelope for a client-side request.

```
$ curl "https://api.rlcdn.com/api/identity?pid=99&rt=envelope" -H "Origin:http://liveramp.com"

{"envelope":"AjfowMv4ZHZQJFM8TpiUnYEyA81Vdgg"}
```
### Sidecar
To enable secure decryption of identity envelopes, LiveRamp has developed a "sidecar" appliance for SSP partners to deploy within their environment as a Docker container. The sidecar presents an API available via interprocess communication (IPC) through which mappings may be quickly performed. On startup, the sidecar would fetch, hold, and periodically refresh configured key material from LiveRamp so that no network calls are required at bid time.

Containerization of the sidecar is intended to minimize the risk of inadvertent disclosure, or of disclosure due to partial compromise of the host environment. For example containerization ensures that keys and unencrypted IDs never enter the exchange application memory space, meaning a core dump or use-after-free memory bug could not result in disclosure. Similarly, a bug in the sidecar cannot crash or compromise the exchange application.

For more information on the sidecar and its data flow, see the [complete documentation](https://sidecar.readme.io/docs).

### OpenRTB
We [seek to extend](https://iabtechlab.com/wp-content/uploads/2017/09/OpenRTB-3.0-Draft-Framework-for-Public-Comment.pdf) the OpenRTB specification to allow for passing additional device and people-based identifiers on BidRequest objects, as a new third-party IDs array field on the User object.

Implementers will utilize the `ext` field of the User request object to pass the third-party ID array, including all of the common identifiers included in the Advertising ID Consortium as:

```
"user": {
  "buyeruid": "example_buyer",
  "id": "example_id",
  "ext": {
    "eids": [
    {
      "source": "liveramp.com",
      "uids": [
        {
          "id": "XY1000bIVBVah9ium-sZ3ykhPiXQbEcUpn4GjCtxrrw2BRDGM",
          "ext": {
            "rtiPartner": "idl"
          }
        }
      ]
    },
      {
        "source": "adserver.org",
        "uids": [
          {
            "id": "6bca7f6b-a98a-46c0-be05-6020f7604598",
            "ext": {
              "rtiPartner": "TDID"
            }
          }
        ]
      }
    ],
    "gdpr": 0
  }
}
```

For current LiveRamp IdentityLink customers, the `uid.id` values will match those IDLs delivered through existing server-side integrations to enabled first and third-party data.

## Appendix

### About IdentityLinks
#### Overview
An IdentityLink is LiveRamp's pseudonymous identifier, connected to devices, that represents an individual. It is an anonymized version of an AbiliTec ID which is based on PII.

#### Taxonomy of an IdentityLink
A typical IdentityLink might look something like: `XY1000bIVBVah9ium-sZ3ykhPiXQbEcUpn4GjCtxrrw2BRDGM`. The prefix `XY` indicates that it is a _maintained_ link, the strongest type of IdentityLink. `1000` indicates the IdentityLink consumer, who the link is encoded for. The remaining string refers to the underlying individual. The remaining string content will vary across IDL consumers, even if it represents the same individual.

#### More on IdentityLink Scoping
Each IdentityLink customer receives a unique encoding of the total space of IDLs, where underlying links are encrypted with a customer-specific key held by LiveRamp before transmission.

IdentityLinks are encoded for a variety of reasons, including customer privacy, consumer privacy, regulatory compliance, and protection of LiveRamp intellectual property and business interests. As such LiveRamp, must be able to reasonably protect encryption material from disclosure.

IDLs received by IdentityLink customers in programmatic requests will match those already received today through server-side integrations, which implies a capability of encoding to each demand partner key.

### Related documentation
- [Sidecar](https://sidecar.readme.io)
- [Index Exchange, _How to Use the LiveRamp IdentityLink_](https://kb.indexexchange.com/Identity/How_to_Use_the_LiveRamp_Identity_Link.htm)

### Design Considerations

#### Minimal Latency
The design must not introduce material latency in the auction process. In particular this fully precludes API calls to LiveRamp infrastructure in order to perform mappings, and even discourages introducing extra intra-datacenter network hops.

#### Minimal Dependence on LiveRamp Infrastructure
A LiveRamp system outage must not be allowed to impact partner systems. Short term LiveRamp outages should have no impact on design function, and it must further allow for graceful degradation of service in the event of an extended outage.

#### Single-Point Dynamic Configuration
To successfully scale across the industry, we must minimize the friction involved in onboarding new supply and demand partners. Setting a new partner live must not require code pushes, service restarts, or complex technical coordination and configuration between multiple partners.

Ideally, new demand partners would be enabled without supply partners needing to be directly involved, and demand partners would automatically receive appropriate IDLs as new supply partners are enabled.

#### Protection of Identifiers in Open Channels
The design will necessarily place IdentityLinks in open and generally accessible environments, such as an open cookie space or public API. As part of LiveRamp’s privacy commitments we must review and certify privacy and ethical compliance of all parties receiving IDLs, which requires that we apply measures to protect IDLs from disclosure in these environments.

#### Extensibility
There are scenarios where we may wish to extend the capabilities of partner integrations. For example, LiveRamp has a planned improvement of the underlying offline PII graph technology used by IdentityLink. The design should include a means for LiveRamp to coordinate and perform appropriate internal migrations, to allow for future improvements without impacting partner integrations.
