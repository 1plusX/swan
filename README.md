# Profiles In Guarded EnvirONments (PIGEON)

<p align="center">
  <img width="70%" height="70%" src="https://upload.wikimedia.org/wikipedia/commons/thumb/f/f5/Trocaz_Pigeon_Madeira.jpg/640px-Trocaz_Pigeon_Madeira.jpg"><br>
  <a href="https://en.wikipedia.org/wiki/Trocaz_pigeon">Image of a Madeira laurel pigeon from Wikipedia</a>
</p>



We propose to generalize [Turtledove](https://github.com/WICG/turtledove) and [First-party sets](https://github.com/privacycg/first-party-sets) to enable full profile based targeting while remaining compliant with the chromium privacy model. More precisely, the goal is to be able to target users based on  behavioral data collected across domains and/or parties while making sure this data never leaves the browser.

This proposal addresses the issue of the access restriction to 3rd-party data that is caused by the removal of the 3rd-party cookies. This is particularly problematic for the market actors that have limited first-party data.

In the following we use the notion of "domain" and "party" as defined in the [First-party sets](https://github.com/privacycg/first-party-sets) proposal.

## The capabilities of Turtledove
In a nutshell, the Turtledove framework consists of the following steps to show a targeted ad:
* Interests groups (aka audiences or segments) are computed based on first-party data only (from one domain).
These interest groups are collected in the browser. Only the browser has a *cross-domain* overview on the interest groups a user belongs to.
* The browser retrieves ad bundles (implemented as web bundles) from ad servers using two types of requests that are kept uncorrelated (for privacy reasons). One request type retrieves bundles for the context of the page (contextual targeting). The other request type retrieves an ad bundle for the collected interests groups. Each bundle also includes a bid.
* An in-browser auction occurs to select the ad to show.

From our perspective, the Turtledove approach is well suited to target users for the following types of targeting:
* Contextual targeting. This is simply achieved by using a contextual request as described in the explainer.
* Intent based targeting (based on specific user actions) using an interest group like "hit-paywall-twice". This also includes re-targeting using an interest group like "looked-at-shoes-xyz".
* Limited profile based targeting. Interest groups can be used to profile users using data observed on one domain only. As described in the explainer, this could be achieved using an interest group like "athletic-shoes". This proposal aims at alleviating the per-domain restriction of interest groups.

Since interest groups can be computed using data of only one domain, the semantics of these groups are necessarily related/constrained to the activity on a domain or its content. In other words, interest groups are not well suited to model user properties that can only be computed using behavioral cross-domain data. For example, estimating if a user is male or female cannot simply be computed using data from "wereallylikeshoes.com": the group will be biased toward women and there is no way to compensate the bias by using data collected on other domains.

# Generalization in a nutshell

We propose to introduce in the browser a "sandboxed private storage" that has limited read capabilities. The role of the storage is to keep a profile that consists of data collected *across domains*. Then, only pre-registered scripts with a constrained signature are allowed to read this data and to output interest groups. No other read/write capabilities are allowed for these scripts. The rest remains identical to Turtledove. In the following, we use the term "audience" instead of "interest group" to better reflect the increased targeting capabilities. The pre-registered script is called an audience definition script.

The following diagram gives an overview of the proposal.

<p align="center">
  <img width="70%" height="70%" src="./overview.svg">
</p>

The difference to Turtledove is that partial profiles (i.e. profiles of one domain) are stored in the browser, not audiences (interest groups). The Turtledove bidding process that is used to select an ad is not affected by this proposal.

Next, we discuss how the browser controls the access to partial profiles.

## Access to the private storage

As depicted in the diagram, the audience definition script always has access to the partial profiles that originates from the same party (the audience definition was registered on a domain of that party). Which domains belong to the same party are defined by the server side first-party-set declaration which is known to the browser as described in the [proposal](https://github.com/privacycg/first-party-sets#declaring-a-first-party-set).

As explained so far, it is possible to perform targeting based on client-side first-party profiles. This is similar to the scenario where First-part-sets and Turtledove are implemented simultaneously with the exception that the profile is server sided (assuming first-party-sets are used to implement [cross-domain first-party cookies](https://www.chromestatus.com/feature/5640066519007232)).

To increase data sharing capabilities, we propose to naturally extend the first-party-set proposal to also support third-party declarations. Similarly to first-party sets that enable browsers to understand the first-party relationships between domains, third-party sets enable browser to understand relationships between parties. With this extension, the involved domains depicted in the diagram shall be able to serve the following resources:

```
https://a.com/.well-known/first-party-set
{
  "owner": "a.com",
  "version": 1,
  "members": ["b.com"],
  ...
}

https://b.com/.well-known/first-party-set
{ "owner": "a.com" }

https://c.com/.well-known/third-party-set
{
  "owner": "c.com",
  "version": 1,
  "members": ["a.com"]
}
```

where `a.com` and `c.com` are owners of their respective first-party set as defined in the [proposal](https://github.com/privacycg/first-party-sets#declaring-a-first-party-set). In the context of this proposal, this makes it possible to grant the appropriate reading permissions to audience definition scripts to also access third-party items stored in the private storage. This is depicted in the following diagram:

<p align="center">
  <img width="70%" height="70%" src="./overview2.svg">
</p>

Next, we briefly show how to programmatically use the private storage to perform targeting based on a cross-party profile.

## API Example Flow

We describe an illustrative scenario where a cross-party profile is built in the browser as a user anonymously navigates the web, i.e., without using a login.

As I anonymously browse "weReallyLoveShopping.com" my behavior reveals that I am interested in athletic shoes. The online shop writes this information in the private storage:

```javascript
window.privateStorage.setItem('interests', 'athletic-shoes');
```

I also regularly and anonymously visit the publisher "myLocalNewspaper.com". The domain is not owned by the same party as "weReallyLoveShopping.com" but the latter is declared as a third-party of the former. Based on the pages I read, the publisher estimates a probability of 0.3% that I am a female and writes this to the private storage:

```javascript
window.privateStorage.setItem('femaleProb', 0.3);
```

Note that `privateStorage` does not provide reading capabilities at this point (i.e. the method `privateStorage.getItem` is not accessible).

In addition, domain owners always have the possibility to register audience definition scripts. In our example, the publisher registers an audience definition script as follows:

```javascript
const maleAthleticShoes =
  {'name': 'femaleAthleticShoesAudience',
   'readers': ['first-ad-network.com',
               'second-ad-network.com'],
    'script': "\
    	return privateStorage.getItem('femaleProb') < 0.5 && \
             privateStorage.getItem('interests') == 'athletic-shoes')"
  };
window.privateStorage.addAudienceDefinition(maleAthleticShoes);
```

The audience definition scripts must return a boolean value to indicate audience membership and are evaluated before fetching the ad bundles. In this example, the script needs to access data that originates from two domains. But because of the declared third-party relationship and because the script is running in a sandboxed environment, both `getItem` calls return a value. Since audience definition scripts are only allowed to determine audience membership, no private information can leak from the private storage.

The rest of the API flow remains identical Turtledove; the browser then contacts `first-ad-network.com` and requests ads targeted at this audience:

```bash
curl -X GET https://first-ad-network.com/.well-known/fetch-ads?audience=femaleAthleticShoesAudience
```

See the [Turtledove](https://github.com/WICG/turtledove) for the rest of the flow.


## Privacy

The following design aspects shall guarantee that the browser does not leak private data:

* Only audience definition scripts are allowed to read the content of the storage. The scripts are only allowed to return booleans which guarantees that no private data can leak.

* The audience definition scripts are associated to their originating domains. A script has access to profile data of another domain (resp. party) provided there is a first-party (resp. third-party) relationship between the domains (resp. parties). In addition, this access can be blocked by the user using the appropriate UI control (see below). In this case, the method `getItem` shall return `null`.

* As with Turtledove, audiences (and the associated definition scripts) that have too few users shall be disabled by the browser. This is to prevent micro-targeting which essentially makes tracking possible again.


## UI controls

In addition to the browser UI controls of Turtledove, this approach would enable the following additional controls:

* The browser shall provide a mean to see the profile data that is collected. In this way, GDPR/CCPA access requests simply amount to reading this storage using a UI control in the browser (provided the profile contains all the relevant data collected).

* The user shall be able to delete audience memberships using a UI control. This can be achieved by removing/blocking the audience definitions. In addition, domain owners can also remove audience definitions using a method `privateStorage.removeAudienceDefinition` (this is similar to Turtledove).

* The user shall be able to delete/block the private storage for an advertiser that he/she does not trust.

* The user shall be able to block access to third-party data. The browser could provide a central configuration panel that shows the third-party associations between parties with the possibility to block them. For stricter privacy configurations, the browser could prompt for consent upon the first access to a third-party item using `getItem`.


## Other design considerations

* The signature of the private storage is inspired by the signature of the existing [local and sessions storages](https://developer.mozilla.org/en-US/docs/Web/API/Storage). To avoid naming collisions on the items of the `privateStorage`, a namespace string parameter shall be added to the signature of the `getItem` method to specify the domain that owns the data. The other methods `setItem`, `removeItem`, `clear` shall implicitly use the domain of the current context and shall not be available in to the audience definitions scripts.

* Third-party data providers very likely need to understand the usage of their data (e.g. to charge the first-party accordingly). Our proposal is to use the [Aggregated Reporting API](https://github.com/csharrison/aggregate-reporting-api) to count the calls to `getItem`  or the number of users accessing a given item.

## Related work and impact on other proposals

This proposal generalizes [Turtledove](https://github.com/WICG/turtledove) and extends the [First-party sets](https://github.com/privacycg/first-party-sets) proposal. In addition, it affects the [SCAUP proposal](https://github.com/google/ads-privacy/blob/master/proposals/scaup/README.md) as explained next.
* The proposal introduces feature vectors that are stored in the browser in a write-only mode. The private storage presented in this proposal could be used exactly for this purpose. Full read-access to the profile shall be provided only to the process converting the data into opaque secret shares before they are sent to the MPC servers.
* The proposal outlines the creation of a profile without specifying its domain and/or party scope. This proposal addresses this issue by introducing a similar notion of profile that clearly differentiates between first-party and third-party features (called items in this proposal).
