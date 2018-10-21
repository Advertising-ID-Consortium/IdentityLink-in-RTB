# IdentityLink in RTB

## Overview
This document discusses technical means for enabling programmatic supply and demand partners with IdentityLinks (IDLs) in bid requests. It discusses design considerations, mechanisms by which identifiers are obtained, and means for passing them to demand partners.

### IdentityLink Encoding
Each IdentityLink customer today receives a unique encoding of the total space of IDLs, where underlying links are encrypted with a customer-specific key held by LiveRamp before transmission. This is done for a variety of reasons, including customer privacy, consumer privacy, regulatory compliance, and protection of LiveRamp intellectual property and business interests. As such LiveRamp must be able to reasonably protect encryption material from disclosure.

IDLs received by IdentityLink customers in programmatic requests should match those already received today through server-side integrations, which implies a capability of encoding to each demand partner key.

## Components

### OpenRTB Protocol Updates
We [seek to extend](https://iabtechlab.com/wp-content/uploads/2017/09/OpenRTB-3.0-Draft-Framework-for-Public-Comment.pdf) the OpenRTB specification to allow for passing additional device and people-based identifiers on BidRequest objects, as a new third-party IDs array field on the User object.

Until ratified by the IAB, implementers will utilize the `ext` field of the User request object to pass the third-party ID array, including all of the common identifiers included in the Advertising ID Consortium as:

```
{
    "user": {
        "id": "... unchanged ...",
        "buyerid": "... unchanged ...", #dsp specific identifier
        "ext": {
            "eids": [{
                "source": "adserver.org",
                "uids": [{
                    "id": "uid123", #id received from request
                    "ext": {
                        "rtiPartner": "TDID" #outgoing name
                    }
                }
                }]
            }]
        }
    }
}
```

For current LiveRamp IdentityLink customers, the `uid` values will match those delivered through existing server-side integrations.

### Identity Envelopes
Envelopes are small but opaque base64-encoded strings, encrypted with a random and single-use [initialization vector](https://en.wikipedia.org/wiki/Initialization_vector), such that each generation of an Envelope is unique regardless of its content. They are intended to capture IdentityLinks without directly exposing those links to the Envelope bearer; only parties having an integration with LiveRamp are provided the means to access underlying links.

With appropriate protection of encryption keys, this makes Envelopes suitable for protecting and passing IdentityLinks through public channels such as Web APIs and shared Cookie spaces. As intentionally opaque payloads, Envelopes also provide a joint for future extensibility: LiveRamp can coordinate modifications to the internal encoded structure without disrupting partner integrations.

Envelopes are key to integrating IDLs into programmatic requests, and we plan for multiple and overlapping means of obtaining Envelopes. Integrating partners would select the most appropriate means of integration for their systems.

**Via Mapping Files**

LiveRamp will provide file-based feeds of mappings between Consortium device identifiers ⇔ Identity Envelopes, as well as Mobile AdID ⇔ Identity Envelopes for partners electing to host their own key-value infrastructure.

**Via CORS API**

LiveRamp will provide a CORS API (with open origin permissions) to return an envelope for a client-side request.

```
$ curl "https://api.rlcdn.com/api/identity?pid=99&rt=envelope" -H "Origin:http://liveramp.com"

{"envelope":"abc123"}
```

### Decrypting Identity Envelopes
LiveRamp has developed a "sidecar" appliance for SSP partners to deploy within their environment as a Docker container. The sidecar presents an API available via interprocess communication (IPC) through which mappings may be quickly performed. On startup, the sidecar would fetch, hold, and periodically refresh configured key material from LiveRamp so that no network calls are required at bid time.

Containerization of the sidecar is intended to minimize the risk of inadvertent disclosure, or of disclosure due to partial compromise of the host environment. For example containerization ensures that keys and unencrypted IDs never enter the exchange application memory space, meaning a core dump or use-after-free memory bug could not result in disclosure. Similarly, a bug in the sidecar cannot crash or compromise the exchange application.

For more information on the sidecar and its data flow, see the [complete documentation](https://sidecar.readme.io/docs).

## Appendix

### Related documentation
- [Sidecar](https://sidecar.readme.io)
- [Index Exchange, DSP](https://knowledgebase.indexexchange.com/display/ID/How+to+Use+adsrvr.org)
- [Index Exchange, RTB Adapter](https://knowledgebase.indexexchange.com/display/ID/How+to+Use+adsrvr.org)

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
