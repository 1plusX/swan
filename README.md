# Dove generalization

We propose to generalize [TURTLEDOVE](https://github.com/WICG/turtledove) to enable full profile based targeting while remaining compliant with the chromium privacy model. More precisely, the goal is to be able to target users based on deterministic behavioral data collected across origins while preserving privacy.

## The capabilities of TURTLEDOVE
In a nutshell, the TURTLEDOVE framework consists of the following steps to show a targeted ad:
* Interests groups (aka audiences or segments) are computed based on first-party data only (from one origin).
These interest groups are collected in the browser. Only the browser has a *cross-origin* overview on the interest groups a user belongs to.
* The browser retrieves ad bundles (implemented as web bundles) from ad servers using two types of requests that are kept uncorrelated (for privacy reasons). One request type retrieves bundles for the context of the page (contextual targeting). The other request retrieves an ad web bundle for the collected interests groups. Each bundle also includes a bid.
* An in-browser auction occurs to select the ad to show.

From our perspective, the TURTLEDOVE approach is well suited to target users for the following types of targeting:
* Contextual targeting. This is simply achieved by using a contextual request as described in the explainer.
* Intent based targeting (based on specific user actions) using an interest group like "hit-paywall-twice". This also includes re-targeting using an interest group like "looked-at-shoes-xyz".
* Limited profile based targeting. Interest groups can be used to profile users using data observed on one origin only. As described in the explainer, this could be achieved using an interest group like "athletic-shoes". This proposal aims at alleviating the per-origin restriction of interest groups.

Since interest groups can be computed using data of only one origin, the semantics of these groups are necessarily related/constrained to the activity on a origin or its content. In other words, interest groups are not well suited to model user properties that usually can only be computed using behavioral cross-origin data. For example, understanding if a user is male or female cannot simply be computed using data from "wereallylikeshoes.com": the group will be biased toward women and there is no way to compensate the bias by using data collected on other origins (like "wereallylovecars.com").

# Generalization in a nutshell

We propose to introduce in the browser a cross-origin "sandboxed private storage" that has very limited read capabilities. The role of the storage is to keep a profile that consists of data collected *across origins*. Then, only pre-registered scripts with a constrained signature are allowed to read this data and to output interest groups. No other read/write capabilities are allowed for these scripts. The rest remains identical to TURTLEDOVE. In the following, we use the term "audience" instead of "interest group" to better reflect the increased semantic modeling capabilities. The pre-registered script is called an audience definition script.

The following diagram gives an overview of the proposal.

![overview](./overview.svg)

The main difference to TURTLEDOVE is that partial profiles are stored in the the browser, not interest groups. The attribution to audiences is then performed in the browser using audience definition scripts provided by advertisers. The bidding process to select an ad web bundle is not affected by this proposal.


## API Example Flow

In the following, we describe an illustrative scenario where a cross-origin profile is built in the browser as a user navigates the web.

As I browse "wereallylikeshoes.com", the advertiser that is instrumented on a page creates a private storage instance. Based on the user behavior observed on the domain, the advertiser estimates a probability of 0.8% to be female and adds this information to the storage. Alongside the probability, my visit frequency per month is provided.

```javascript
const advertiserId = ...;
const privateStorage = navigator.getOrCreatePrivateStorage(advertiserId);
privateStorage.appendToArray("genderEstimates", {m: 0.2, f: 0.8, w:5});
```

As I continue to browse the web, I now visit "wereallylovecars.com". The same advertiser now estimates a probability of 0.7 to be male for that origin. The visit frequency on this origin is 25 times per month. This is written into the same storage instance.

```javascript
privateStorage.appendToArray("genderEstimates", {m: 0.7, f: 0.3, w:25});
```

The advertiser always has the possibility to register audience definition scripts as follows:

```javascript
const maleAudience =
  {'name': 'maleAudience',
   'readers': ['first-ad-network.com',
               'second-ad-network.com'],
    'script': "\
      const estimates = storage.get('genderEstimates'))\
      const estimate = … // compute overall estimate based on partial estimates \
    	return estimate > 0.5"
  };
privateStorage.addAudienceDefinition(maleAudience);
```

The audience definition scripts return a boolean value to indicated audience membership. In this example, if the script returns `true`, then the user belongs to the `maleAudience`.

Before requesting the ads, the browser evaluates all the registered scripts to determine audience memberships. The rest of the API flow remains as with TURTLEDOVE: the browser then contacts first-ad-network.com and requests ads targeted at this audience:
GET https://first-ad-network.com/.well-known/fetch-ads?audience=maleAudience

See  the TURTLEDOVE for the rest of the flow.

## Privacy

The following design aspects shall guarantee that the browser does not leak private information:

* Audiences can be defined freely by advertisers. However, as with TURTLEDOVE, browsers should devise an infrastructure to deactivate audiences that have to few users. This is to prevent micro-targeting which essaentialy makes tracking possible again.

* The proposed private storage shall be accessed by one advertiser only to prevent malicious actors from corrupting profiles (and affecting the advertisers business). In the paragraph [API Example Flow](#api-example-flow), the storage is accessed using an advertiser ID. This needs to be improved and an authentication mechanism must be devised to control the access to the storage.

* Only audience definition scripts are are allowed to read the content of the storage (for example, one could imagine a separate javascript runtime per storage). The script is only allowed to return booleans which guarantees that no private data can leak.

## UI controls

In addition to the browser UI controls of TURTLEDOVE, this approach would enable the following additional controls:
* The browser could provide a mean to see the profile data that is collected by a given advertiser. GDPR/CCPA access requests could simply amount to reading this storage using a UI control in the browser (provided the profile contains all the data collected by the advertiser).
* The audience definition scripts could also be inspected by the user. This has the advantage that users have a mean to understand *how* they got attributed to an audience. Maybe this could be useful to discover problematic targeting practices (for example political targeting campaigns). On the other hand, we believe that advertisers have the right to keep their targeting technologies proprietary. However, this could be constrained to the partial profiles that are computed from one origin only (from first-party data, where the heavy lifting is happening).
* The user shall be able to delete audience memberships using a UI control. This can be achieved by removing/blocking the audience definitions. In addition, advertiser can also remove audience definitions using a method `privateStorage.removeAudienceDefinition` (this is similar to TURTLEDOVE).
* The user shall be able to delete/block the private storage for an advertiser that he/she does not trust.

## Compatibility with other proposals

This proposal generalizes TURTLEDOVE and does not conflict with the other main proposals such as [SPARROW](https://github.com/WICG/sparrow) (bidding) and the [Aggregated Reporting API](https://github.com/csharrison/aggregate-reporting-api) (that could also be used for [lookalike targeting](https://github.com/w3c/web-advertising/blob/master/privacy_preserving_lookalike_audience_targeting.md)).
