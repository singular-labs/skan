

# SKAN: A standard for the implementation of SKAdNetwork
### Version 1.1

- [Background](#background)
	- [Dealing with a fragmented attribution landscape](#landscape)
- [1 - SKAN Participants](#a1)
- [2 - Conversion Value Management](#a2)
	- [2.1 - Introduction to Conversion Value Management](#a2.1)
	- [2.2 - Existing Limitations](#a2.2)
	- [2.3 - Intelligent conversion value management](#a2.3)
	- [2.4 - Conversion value management data flow](#a2.4)
	- [2.5 - Supported models](#a2.5)
	- [2.6 - Frequency of conversion value updates](#a2.6)
	- [2.7 - Max duration of conversion value](#a2.7)
- [3. SKAN Campaign ID Management](#a3)
	- [3.1 - Background](#a3.1)
	- [3.2 - Mapping API](#a3.2)
- [4 - Secure Data Flow](#a4)
	- [4.1 - Background](#a4.1)
	- [4.2 - List of Secure Ad Networks](#a4.2)
	- [4.3 - Secure Data Flow Diagram](#a4.3)
	- [4.4 - Detailed Data Flow](#a4.4)
- [5 - Postback Structures](#a5)
	- [5.1 - Attribution Postback](#a5.1)

<a name="background"></a>
# Background

SKAdNetwork is a framework provided by Apple to enable privacy-friendly measurement of app install ad campaigns on iOS (read more here). The framework was last updated to version 2.0 by Apple on June 22nd.

SKAdNetwork, in its current form, is a bare-bones spec, that requires careful implementation and responsible governance between multiple entities in the ecosystem to ensure that it’s practical and useful.

This document outlines the specifications for practical SKAdNetwork implementations, an open standard, endorsed by ad networks, publishers, advertisers and MMPs, designed to be vendor-agnostic and shared across the entire mobile ecosystem. We are calling this spec **SKAN**.

**With SKAN, we are aiming to achieve the following:**

- Create a reference implementation plan for SKAdNetwork, to avoid fragmentation and a broken mobile advertising ecosystem.
- Design a way for SKAdNetwork to be compatible with parallel methods, as an example IDFA-based attribution (when consent is given), probabilistic matching, and other methods that might be available in the future.
- Provide a way to centralize and manage the scattered SKAdNetwork postbacks.
- Govern SKAdNetwork’s “conversion value” and define a standard implementation.
- Provide a solution for the fraud risk presented in SKAdNetwork (such as the conversion value manipulation, repeating transaction ID attacks, and geo manipulation).
- Define how marketers and ad networks can measure ROAS and other KPIs.
- Define the infrastructure necessary for centralized reporting for advertisers.

*This spec is evolving. We would love to get more input from the industry and Apple!*

<a name="landscape"></a>
## Dealing with a fragmented attribution landscape

SKAdNetwork has its benefits, but it also has severe limitations, and we are actively participating in the development of alternative privacy-friendly methods.

Given what we know today, we believe the measurement universe on iOS will not have a single winning method. Attribution and credit assignment will have more data points to consider than ever before, and will rely on multiple methods:

- User-level attribution when there’s consent (and perhaps broader if one-sided consent techniques will be adopted by Apple).
- Universal links attribution.
- Attribution based on other identifiers (1st party login, email, etc).
- On-device attribution and attribution anonymization.
- And… SKAdNetwork based attribution.

With SKAN, we are trying to minimize that fragmentation as much as possible by:
- Creating a unified language between advertisers, ad networks, partners and MMPs.
- Standardizing the postbacks sent to the partners, to be as similar as possible between the various attribution methods listed above.
- Making reporting easier between the various attribution methods, and try to establish a common language between them.

<a name="a1"></a>
# 1 - SKAN Participants
- Ad Network: Manages displayed ads and connects between advertisers and publishers
- Publisher: Source app that displays ads provided by various Ad Networks
- Advertiser: Target app that appears in the displayed ads
- SSP/Exchange: Manages displayed ads of winning bids in publisher apps.
- DSP: Bids on behalf of advertisers in exchanges.
- MMP: Facilitator of secure and unbiased measurement to the Advertiser and Ad Network / DSP.

<a name="a2"></a>
# 2 - Conversion Value Management

<a name="a2.1"></a>
## 2.1 - Introduction to Conversion Value Management
SKAdNetwork 2.0 includes a mechanism to perform post-install conversion measurement, however it is rather limited. Since SKAdNetwork works for all users without consent, Apple has deliberately added limitations so it would be impossible to abuse the mechanism to leak the identities of any user who has opted out.

The mechanism works in the following way:

1. Once `registerAppForAdNetworkAttribution()` is called for the first time, a 24-hour timer will begin. We'll call it `Timer 1`.
2. The app can then call the `updateConversionValue()` function and provide a 6-bit number (values between `0` to `63`), which is the conversion value. The app may do so as long as `Timer 1` hasn’t expired.
4. Every subsequent call to `updateConversionValue()` with a higher value than the previous value will update the conversion value, and reset `Timer 1`, to get a new 24-hour window.
5. Once `Timer 1` expires, the operating system will randomly pick a new duration (between 0 to 24 hours), and create `Timer 2`. At this point, any subsequent calls to `updateConversionValue()` are meaningless.
6. Once `Timer 2` expires, the device will send a postback to the registered SKAdNetwork URL (the appropriate MMP registration according to 2.2).
7. The postback will contain the conversion value (which, as previously mentioned, is **not signed** by Apple)

Conversion value is the **only mechanism** for an advertiser to connect the post-install activity to an app install campaign and achieve some form of CPA/ROAS, which is why it’s imperative that advertisers are intelligent about how they manage this value.

<a name="a2.2"></a>
## 2.2 - Existing Limitations
The following limitations exist:
- Conversion values are limited to 64 values (6 bits).
- Conversion values will only trigger within the specific timing constraints.
- Advertisers may want to change how conversions are represented and optimized without needing to deploy changes to the app on every change.

<a name="a2.3"></a>
## 2.3- Intelligent conversion value management
As part of SKAN, the MMP should manage this logic for the advertiser, and enable an automated and standardized mechanism for managing conversion values.

This mechanism is designed for the following purposes:
- Provide the advertiser an easy way to report key metrics in a standardized and well thought out method.
- Provide the infrastructure for reporting key metrics like ROAS or CPA.
- Provide a standard set of conversion values for ad networks for their optimization purposes.
- Ensure compatibility, whenever possible, with existing event postback integrations.

<a name="a2.4"></a>
## 2.4 - conversion value management data flow

1. Each event and session tracked by the MMP SDK also queries the MMP Server for a conversion value.
2. The MMP server decides on an appropriate conversion value based on past tracked events and other information such as user geo. 
3. The MMP SDK calls updateConversionValue using the value from the server’s response.
4. When an SKAdNetwork postback is handled by the MMP, the conversion value is re-mapped into meaningful events and metrics to be used in reporting.
5. Postbacks are sent to AdNetworks and BI endpoints with the appropriate events and mapped conversion values.

<a name="a2.5"></a>
## 2.5 - Supported models

We suggest the following models as the core framework for managing conversions using SKAdNetwork. SKAN enables the creation of new models quite easily since the logic is ran on the server-side, and as such - MMPs and advertisers will be able to create additional models in future updates.

Please note that a single model is used across all ad networks publishing the app at a given point in time.

| 5 | 4 | 3 | 2 | 1 | 0 |
| --- | --- | --- | --- | --- | --- | 

`_Table 2: Conversion value bitmap`

List of supported models:

1. Revenue model
	- Bits [0:3] : 4 bits to encode different buckets of IAP/Ad Revenue (See Appendix A for explicit variant of this model)
	- Bits [4:5] : 2 bits to encode the count of days since the install, supporting 0-3 days

2. Retention model
	- Bits [0:5] 6 bits to encode the count of days since the install
	- Retention is only recorded only as long as the user opens the app every day.

3. Simple Event-based model
	- Bits [0:2] 3 bits that can represent different events, including the initial install event.
For example:
`001` = install
`010` = sign up
`100` = first purchase
`011` = third purchase

	- Bits [3:5] 3 bits that can represent days since install, supporting 0-7 days

4. Non-linear event-based optimization
	- This model is a variant of model #1 or #3, where the first X values are used to represent simple day changes, until an important conversion event is happening
	- After first conversion happened, following conversion values will be all event-based
	- Advertiser can also choose a certain number of days (e.g., first 3 days) they want to track

5. Predicted LTV-based optimization
	- Existing pLTV modeling can be used to dynamically manage conversions
	- Conversion events can be finetuned at the device level taking into account pLTV, which for mobile games for example can be typically established on the first 1-3 days.
	- Conversion management can then be done at higher resolution given the predicted LTV is established for the device.

> We expect to continue and update the Supported Models section as new advertiser use cases will arise and new methods will become available

<a name="a2.6"></a>
## 2.6 - Frequency of conversion value updates

Conversion values in SKAdNetwork need to be updated every 24 hours once SKAdNetwork is initiated. If an app is not active for more than 24 hours, SKAdNetwork conversion value update will be stopped and a postback will be sent.

Therefore, it is important to:
1. Initiate SKAdNetwork tracking at the right time (time of install or other meaningful event)
2. Use a model that captures the right activity in the app while still allowing for keep-alive updates if needed


<a name="a2.7"></a>
## 2.7 - Max duration of conversion value
Further to 3.4, it is recommended that conversion values are sent for the first 3 days of app usage, and up to 7 days max.

Outside of the SKAdNetwork timer mechanism, which needs to be used effectively, attempting to shorten the time between the install and the delivery of the conversion event will support better optimization capabilities for ad networks. It is also crucial for better reporting, as Campaign ID meanings can change over time, and unless reported close to the time of served ad can lose the original meaning.

<a name="a3"></a>
# 3. Campaign ID Management

<a name="a3.1"></a>
## 3.1 - Background
Ad Networks and DSPs will be in charge of controlling the SKAdNetwork Campaign ID (1-100), and they are likely to use it to its fullest potential, by assigning different campaign IDs to different creatives, time periods, GEOs, etc. As such, the SKAdNetwork Campaign ID is not going to be the same as the campaign that advertisers are using today in the ad network dashboards or their cost aggregation solution.

Networks are also likely to limit the number of campaigns and creatives they allow the advertiser to run, to enable a consistent mapping to the limited SK Campaign ID values.

While the actual details of assigning SKAdNetworkCampaignId values may vary from each network, ad networks implementing SKAN will provide a mapping between the SKAdNetwork Campaign IDs and their outward-facing ad network Campaign & Creative IDs. This information can be consumed by the advertiser or MMP to build the various performance reporting for the customer.

<a name="a3.2"></a>
## 3.2 - Mapping API


Networks will provide an API that will contain the following fields:

- `Start Date` - the start date in which ad impressions used this particular mapping
- `End Date` - the start date in which ad impressions used this particular mapping
- `SKCampaignId` - The SKAdNetwork Campaign ID (value between `1-100`)
- `Country` [optional] - if there’s a different meaning for `SKCampaignId` for this country (`ISO 3166-1 alpha-2 encoding`)
- `Publisher ID` [optional] - if there’s a different meaning for `SKCampaignId` for this publisher
- `Campaign ID` - internal ad network campaign ID
- `Ad Group ID` [optional] - used if ad groups are part of the mapping
- `Creative ID` [optional] - used if creatives are part of the mapping

### Example:

Dates | SKCampaignId | Country | Publisher | Campaign Id | Ad Group ID | Creative ID
--- | --- | --- | --- | --- | --- | ---
`9/1/2020 - 10/1/2020` | `7` | | | `1238712` | | `848483`
`9/1/2020 - 10/1/2020` | `7` | `US` | | `1238714` | | `848489`

SKAdNetwork Campaign ID `7` has two different mappings:
- In the US - it’s mapped to campaign ID `1238712` and creative ID `848483`
- Otherwise - it’s mapped to campaign ID `1238714` and creative ID `848489`

<a name="a4"></a>
# 4 - Secure Data Flow

<a name="a4.1"></a>
## 4.1 - Background

An optional, but highly recommended data flow, puts the advertiser, represented by their selected MMP, as the direct receiver of the SKAdNetwork postbacks sent by Apple. This design decision serves multiple requirements, including security, data validation and data governance:

- Automatic verification of Apple signature on the receipts, and prevents miscounting of duplicate postbacks
- Prevents tampering of SKAdNetwork “conversion value” and “device geo”.
- Creates a centralized view for the advertiser.
- Provides a consolidated flow for the other attribution methods that wil be managed by the MMP (partial IDFA based on opt-in, probabilistic algorithms, deeplinking of existing users).

This design is achieved by creating specific SKAdNetworkIdentifier registrations for every ad network & MMP pair.

<a name="a2.2"></a>
## 2.2 - List of Secure Ad Networks

MMP | Ad Network | SKAdNetworkIdentifier
--- | --- | ---
Singular | ironSource | `DZG6XY7PWJ.skadnetwork`
Singular | Vungle | `Hdw39hrw9y.skadnetwork`
Singular | Unity Ads | `**y45688jllp.skadnetwork**`

Please submit a pull request to this repo to register your network.

<a name="a2.3"></a>
## 2.3 - Secure Data Flow Diagram


Image 1: Diagram describing the Secure Data Flow design

<a name="a2.4"></a>
## 2.4 - Detailed Data Flow

### Ad Serving

- The Ad Network SDK sends an Ad Request from the Publisher App

- The Ad Network Server serves an ad. Based on the advertiser’s MMP, the Ad Network Server will select the right SKAdNetworkIdentifier, and will sign the request with the appropriate key (see section 2.2 for a list of secure ad networks).

- Ad Network Server sends the Ad Response to the Ad Network’s SDK with its signed SKAdNetwork fields. Note that the Publisher App must have all registered Ad Network IDs in its Info.plist

- The Ad Network SDK displays the ad in any desired format. Upon ad click, the Ad Network SDK will call the StoreKit’s `loadProductWithParameters` function, and includes the SKAdNetwork fields.

### Click Tracking

- The Ad Network SDK sends the Ad Click notification to the Ad Network Server (same mechanism as today).

- The Ad Network SDK sends the click to the MMP Server (same mechanism as today), optionally including the SKAdNetwork fields.

- For S2S click postbacks, we recommend moving the click notification to be sent from the client.

### Install Attribution

- The user installs the app from the App Store and opens it.

- The MMP SDK within the Advertiser App will notify the MMP Server on the app install, and will automatically call the registerAppForAdNetworkAttribution method.

- Advertiser App tracks events via the MMP SDK (same mechanism as today). The MMP SDK will send those events to the MMP Server, and in addition, will call the updateConversionValue. (More details on different models for handling conversionValue below).

- The device’s operating system sends the SKAdNetwork notification to the MMP Server once the random timer ends (and this may be delayed depending on the conversion value model the advertiser chose, and how many events the user performed)

- The MMP Server validates the SKAdNetwork notification (validates the signature, dedupes transaction IDs, etc)

- The MMP Server sends an Attribution postback to the Ad Network:
	- If user-level attribution is approved, an attribution postback will be sent, including post-install events, in a similar fashion to how it works today.
	- In addition, the MMP server will be responsible for sending an SKAdNetwork postback to the network.


<a name="a5"></a>
# 5 - Postback Structures

<a name="a5.1"></a>
## 5.1 - Attribution Postback

**Data flow:** MMP Server -> AdNetwork Server
**URL:** Same endpoint networks use today for installs/event postbacks from MMPs, or a separate SKAN postback
**METHOD:** POST

Given that there are already existing attribution integrations between thousands of Ad Networks and the top MMPs, we will utilize the same postback integrations that exist today for the SKAN postbacks as well.

The SKAN Attribution postback will be a new postback type that is sent in addition to the classic install/event postbacks that exist today (where Device ID is used for attribution). MMPs implementing SKAN will keep populating the existing postback format they have today, while applying some of the following changes:

#### New Fields:

`attribution_type`
This existing field typically has the relevant attribution method such as “idfa”, “idfv”, etc.
This field will now also support a new type named “skadnetwork”

`skan_postback`
This will be a base64 encoded value of the JSON object as received in the SKAdNetwork install validation postback (as documented here). An example value would look like this:

```json{
 "version" : "2.0",
 "ad-network-id" : "com.example",
 "campaign-id" : 73,
 "source-app-id": 1234567891,
 "target-app-id" : 1066498020,
 "transaction-id" : "6aafb7a5-0170-41b5-bbe4-fe71dedf1e28",
 "timestamp": 1593760672497,
 "attribution-signature" : "MDYCGQCsQ4y8d4BlYU9b8Qb9BPWPi+ixk\/OiRysCGQDZZ8fpJnuqs9my8iSQVbJO\/oU1AXUROYU=",
 "redownload": 1,
 "conversion-value: 20
}
```

`revenue`
will be populated if the advertiser used the “revenue model”
- Cohort length
- The revenue range (from X to Y)

`events`
Will be populated if the advertiser used the “event model”
- Cohort length
- List of event names (e.g. `[“Registration”, “Subscription”, “Tutorial Complete”]`)

#### Other fields:
- In case of an “skadnetwork” postback type, any fields relying on the device or on a 1:1 match with the click will not be available in this particular postback, such as click_id, idfa, idfv and others. These can still be sent in the user-level attribution postback types (upon consent).
- Other standard attribution fields can still be derived from the SKAdNetwork postback such as the app, timestamp, IP address, country, etc.

#### Example

For a single install, an ad network could receive the two following postbacks:

_IDFA-based postback_:
```
http://postback.adnetwork.com/conversion/install?src=MMP&
attribution_type=idfa&app_id=...&country=...&idfa=...&click_id=...&...
```

_SKAdNetwork-based postback_:
```
http://postback.adnetwork.com/conversion/install?src=MMP&
attribution_type=skadnetwork&app_id=...&country=...&
skan_postback=b8Qb9BPWPi+ixk\/OiRys...
```
