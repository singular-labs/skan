# SKAN: A standard for the implementation of SKAdNetwork
## Draft v0.90.2

# Background
SKAdNetwork is a framework provided by Apple to enable privacy-friendly measurement of app install ad campaigns on iOS (read more here). The framework was last updated to version 2.0 by Apple on June 22nd.

SKAdNetwork, in its current form, is a bare-bones spec, that requires careful implementation and responsible governance between multiple entities in the ecosystem to ensure that it’s practical and useful.

This document outlines the specifications for practical SKAdNetwork implementations, an open standard, endorsed by ad networks, publishers, advertisers and MMPs, designed to be vendor-agnostic and shared across the entire mobile ecosystem. We are calling this spec SKAN.

With SKAN, we are aiming to achieve the following:


Create a reference implementation plan for SKAdNetwork, to avoid fragmentation and a broken mobile advertising ecosystem.


Design a way for SKAdNetwork to be compatible with parallel methods, as an example IDFA-based attribution (when consent is given).


Provide a way to centralize and manage the scattered SKAdNetwork postbacks.


Govern SKAdNetwork’s “conversion value” and define a standard implementation.


Provide a solution for the fraud risk presented in SKAdNetwork (such as the conversion value manipulation, repeating transaction ID attacks, and geo manipulation).


Define how marketers and ad networks can measure ROAS and other KPIs.


Define the infrastructure necessary for centralized reporting for advertisers.

This spec is evolving. We would love to get more input from the industry and Apple!

Dealing with a fragmented attribution landscape

SKAdNetwork has its benefits, but it also has severe limitations, and we are actively participating in the development of alternative privacy-friendly methods.

Given what we know today, we believe the measurement universe on iOS will not have a single winning method. Attribution and credit assignment will have more data points to consider than ever before, and will rely on multiple methods:


User-level attribution when there’s consent (and perhaps broader if one-sided consent techniques will be adopted by Apple).
Universal links attribution.
Attribution based on other identifiers (1st party login, email, etc).
And… SKAdNetwork based attribution.

With SKAN, we are trying to minimize that fragmentation as much as possible by:


Creating a unified language between advertisers, ad networks, partners and MMPs.
Standardizing the postbacks sent to the partners, to be as similar as possible between the various attribution methods listed above.
Making reporting easier between the various attribution methods, and try to establish a common language between them.e
SKAN Participants
Ad Networks: Manages displayed ads and connects between advertisers and publishers
Publisher: Source app that displays ads provided by various Ad Networks
Advertiser: Target app that appears in the displayed ads
SSPs: 
DSPs: 
MMP: Facilitator of secure and unbiased measurement to the Advertiser and Ad Network


1 - Secure Data Flow

An optional, but highly recommended data flow, puts the advertiser, represented by their selected MMP, as the direct receiver of the SKAdNetwork postbacks sent by Apple. This design decision serves multiple requirements, including security, data validation and data governance:


Automatic verification of Apple signature on the receipts, and prevents miscounting of duplicate postbacks


Prevents tampering of SKAdNetwork “conversion value” and “device geo”.


Creates a centralized view for the advertiser.


Provides a consolidated flow for the other attribution methods that wil be managed by the MMP (partial IDFA based on opt-in, probabilistic algorithms, deeplinking of existing users).

This design is achieved by creating specific SKAdNetworkIdentifier registrations for every ad network & MMP pair.

2 - List of Secure Networks

MMP
Ad Network
SKAdNetworkIdentifier
Singular
ironSource
DZG6XY7PWJ.skadnetwork
Singular
Vungle
Hdw39hrw9y.skadnetwork
Singular
Unity
y45688jllp.skadnetwork



Listed below is a sample data flow proposed by SKAN:





Ad Serving:


The Ad Network SDK sends an Ad Request from the Publisher App (this implementation is under the purview of each ad network and will remain similar to the existing mechanism today that may include caching, and other waterfall/bidding mechanisms).


The Ad Network Server decides which ad to serve. Based on the advertiser’s MMP, the Ad Network Server will select the right SKAdNetworkIdentifier, and will sign the request with the appropriate key.


Ad Network Server sends the Ad Response to the Ad Network’s SDK with its signed SKAdNetwork fields. Note that the Publisher App must have all registered Ad Network IDs in its Info.plist


Ad Network SDK displays the ad (in any way they want). Upon ad click, the Ad Network SDK will call the StoreKit’s loadProductWithParameters function, and includes the SKAdNetwork fields.

Click Tracking:


Ad Network SDK sends the Ad Click notification to the Ad Network Server (same mechanism as today).


Ad Network SDK sends the click to the MMP Server (same mechanism as today), including the SKAdNetwork fields. For S2S click postbacks, SKAN recommends moving the click notification to be sent from the client.

Install Attribution:


The user installs the app from the App Store and opens it.


The MMP SDK within the Advertiser App will notify the MMP Server on the app install, and will automatically call the registerAppForAdNetworkAttribution method.


Advertiser App tracks events via the MMP SDK (same mechanism as today). The MMP SDK will send those events to the MMP Server, and in addition, will call the updateConversionValue. (More details on different models for handling conversionValue below).


The device operating system sends the SKAdNetwork notification to the MMP Server once the random timer ends (and this may be delayed depending on the conversion value model the advertiser chose, and how many events the user performed)


The MMP Server validates the SKAdNetwork notification (validates the signature, dedupes transaction IDs, etc)


The MMP Server sends an Attribution postback to the Ad Network:


If user-level attribution is approved, an attribution postback will be sent, including post-install events, in a similar fashion to how it works today.


In addition, the MMP server will be responsible for sending an SKAdNetwork postback to the network.


SKAN Conversion Value Management

Intro to SKAdNetwork’s Conversion Value
SKAdNetwork 2.0 includes a limited mechanism to perform some post-install conversion measurement. Since SKAdNetwork works for all users, without consent, Apple made it limited on purpose, so that it would be impossible to abuse the mechanism to leak the identities of users who opted out.

The mechanism works in the following way:


Upon calling the registerAppForAdNetworkAttribution method for the first time, a 24 hour timer begins. Let’s call it Timer 1.
The app can then call the updateConversionValue function, and provide a 6-bit number (0-63) which is the conversion value. The app may do that as long as Timer 1 hasn’t expired.
Every subsequent call to updateConversionValue with a value higher than the previous value will update the conversion value and reset Timer 1 again, so you get another 24 hours.
Once Timer 1 expires, the OS randomly picks another duration (between 0 to 24 hours), and creates Timer 2. At this point any subsequent calls to updateConversionValue are meaningless.
Once Timer 2 expires, the device will send a postback to the registered SKAdNetwork URL (in SKAN that will be the MMP). The postback will contain the conversion value, and as we mentioned, it is not currently signed (unlike the rest of the postback).


In its current form, the conversion value poses some fundamental difficulties:


Advertisers have a very limited set of values to work with (0-63) and they only have one chance.
Conversion values mean different things to different publishers as goals and KPIs differ.
In order to set a new conversion value (all before Timer 1 expires), you must set a higher value. But higher doesn’t always mean better. Imagine a situation where we’d like to encode the fact that a user is of low quality -- and we can only determine that on day 2. That means we will need to encode “low quality” as a higher value than perhaps the initial value.
Marketers will want to tweak and adjust the way the value is used. Making code changes and releasing a new app version for each such tweak will make this process impossibly complicated.


From an SKAdNetwork-perspective, this limited conversion value is the only mechanism for an advertiser to connect the post-install activity to an app install campaign and achieve some form of CPA/ROAS, which is why it’s imperative that advertisers are intelligent about how they manage this value.

Intelligent conversion value management
As part of SKAN, the MMP should manage this logic for the advertiser, and enable an automated and standardized mechanism for managing conversion values.

This mechanism is designed for the following purposes:
Provide the advertiser an easy way to report key metrics in a standardized and well thought out method.
Provide the infrastructure for reporting key metrics like ROAS or CPA.
Provide a standard set of conversion values for ad networks for their optimization purposes.
Ensure compatibility, whenever possible, with existing event postback integrations.

Data flow:
Each event and session tracked by the MMP SDK also queries the MMP Server for a conversion value.


The MMP server decides on an appropriate conversion value based on past tracked events and other information such as user geo. 


The MMP SDK calls updateConversionValue using the value from the server’s response.


When an SKAdNetwork postback is handled by the MMP, the conversion value is re-mapped into meaningful events and metrics to be used in reporting.


Postbacks are sent to AdNetworks and BI endpoints with the appropriate events and mapped conversion values.



Examples for conversion value models:


These are just examples of how intelligent conversion value management can be used to encode key metrics -

Revenue-based model
Uses 2 bits (4 values) to encode days since the install
Uses 4 bits to encode different buckets of IAP/Ad Revenue


Simple continuous retention-based optimization
Every time the app opens (up to a certain threshold) the number of days is recorded as the value.
The retention is recorded only as long as the user opens the app every day.


Blended continuous retention-based optimization
Every time the app opens (up to a certain threshold) the number of days is recorded in the higher X bits (depending on the maximal number of days being tracked).
The rest of the available bits can be used to represent additional events happening in the app (for example, finished level 10).


Simple event-based optimization
Assign 3 key events to 3 bits. These bits will be applied through a “logical OR” to combine together. For example:
001 = registration
010 = subscription
100 = share
011 = register and subscribed
Assign 3 bits (8 days) to encode the days since the install


Advanced event-based optimization
Assign values to specific funnel stages.
Use some of the values (for example 0-3) as “fillers” to make sure users have got at least 3 days to use the app before they advance to the first step of the funnel.
    OR
Use 2 bits to make sure the funnel is tracked for at least 4 days.
Can be mixed with other methods such as retention or revenue

SKAN Campaign ID Management

Ad Networks and DSPs will be in charge of controlling the SKAdNetwork Campaign ID (1-100), and they are likely to use it to its fullest potential, by assigning different campaign IDs to different advertisers, different time periods, different GEOs and even potentially different publishers.

As such, this number is not going to be the same as the campaign that advertisers are using today in the ad network dashboards or their cost aggregation solution.

MMPs will be able to maintain a mapping between the SKAdNetwork Campaign IDs and their outward-facing ad network Campaign IDs by receiving the SKAdNetwork fields as part of the click postbacks (see the details below).
The mapping can be based on the following input parameters:


SKAdNetwork Campaign ID
Advertiser App ID
Publisher App ID
Click Date
Country

The mapping can then be used to join an install and post-install conversion values from SKAdNetwork with an Ad Network campaign, and the associated ad spend, to achieve Cost Per KPI or ROAS reporting. Note that it’s important to keep the mapping consistent for at least a few days to make sure that joining aggregated SKAdNetwork postbacks with aggregated click data is possible.

To illustrate how it works let's look at an example. A user clicked an ad for a game, the user was targeted by a campaign that targets iOS users from the US, it’s incentivized and playable. He did it in an app with publisher-id 1234567 and the assigned SKAdNetwork campaign ID the network has allotted for this campaign, creative, target app and source app is 10. 

Upon click, two things will happen simultaneously:

A store view will be presented for the user with the loadProduct API. The network will provide it with the SKAdNetwork parameters. Including the publisher-id 1234567 and SKAdNetwork campaign ID 10.


At the same time, a click postback will be fired from the network to the MMP.
It will look like:
http://c.mmp.com/?network_id=53&ip=45.23.6.1&target_app=8345&publisher_id=1234567&skan_payload=...&campaign_name=iOS_US_incent_4&creative_name=playable_large_2
(see more details about the skan_payload parameter in Click Postback section)

When the SKAdNetwork postback is finally sent to the MMP, the MMP could look up clicks from that same network, publisher, geo and SKAdNetwork campaign-id to that same target app in the last day. If most clicks match the same campaign_name and creative_name then those parameters can be concluded as the actual campaign and creative name for the SKAdNetwork attribution.




SKAN Postback Structure
Click Postback
Data flow: Ad Network SDK (in the publisher app) -> MMP Server
URL: Changes per MMP, probably same click handler URL that MMPs have today
METHOD: POST/GET - depends on the ad network<>MMP existing setup

Given that there are already existing click integrations between thousands of Ad Networks and the top MMPs, we will keep all the existing fields, and add some new ones as follows:


att_status - the AuthorizationStatus of the ATTrackingManager class


skan_payload - a base64 encoded value of the JSON object used for the loadProductWithParameters function (keys available here).

A sample decoded JSON object would look like this:
{
 "SKStoreProductParameterITunesItemIdentifier" : "1066498020",
 "SKStoreProductParameterAdNetworkIdentifier" : "com.example",
 "SKStoreProductParameterAdNetworkCampaignIdentifier" : 73,
 "SKStoreProductParameterAdNetworkSourceAppStoreIdentifier": "1234567891",
 "SKStoreProductParameterAdNetworkVersion": 2.0
 "SKStoreProductParameterAdNetworkTimestamp": 1593760672497,
 "SKStoreProductParameterAdNetworkNonce": "02bd7c16-747d-41e2-b845-59f150626c35",
 "SKStoreProductParameterAdNetworkAttributionSignature" : "MDYCGQCsQ4y8d4BlYU9b8Qb9BPWPi+ixk\/OiRysCGQDZZ8fpJnuqs9my8iSQVbJO\/oU1AXUROYU="
}

Attribution Postback
Data flow: MMP Server -> AdNetwork Server
URL: Can be the same endpoint networks use today for installs/event postbacks from MMPs, or a separate SKAN postback
METHOD: POST

Given that there are already existing attribution integrations between thousands of Ad Networks and the top MMPs, we will utilize the same postback integrations that exist today for the SKAN postbacks as well.

The SKAN Attribution postback will be a new postback type that is sent in addition to the classic install/event postbacks that exist today (where Device ID is used for attribution). MMPs implementing SKAN will keep populating the existing postback format they have today, while applying some of the following changes:


New Fields:


attribution_type:
This existing field typically has the relevant attribution method such as “idfa”, “idfv”, etc.
This field will now also support a new type named “skadnetwork”


skan_postback:
This will be a base64 encoded value of the JSON object as received in the SKAdNetwork install validation postback (as documented here). An example value would look like this:

{
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


revenue: will be populated if the advertiser used the “revenue model”
Cohort length
The revenue range (from X to Y)


events: will be populated if the advertiser used the “event model”
Cohort length
List of event names (e.g. [“Registration”, “Subscription”, “Tutorial Complete”])


Other fields:
In case of an “skadnetwork” postback type, any fields relying on the device or on a 1:1 match with the click will not be available in this particular postback, such as click_id, idfa, idfv and others. These can still be sent in the user-level attribution postback types (upon consent).
Other standard attribution fields can still be derived from the SKAdNetwork postback such as the app, timestamp, IP address, country, etc.


As an example, for a single install, an ad network could receive the two following postbacks:


IDFA-based postback:

http://postback.adnetwork.com/conversion/install?src=MMP&
	attribution_type=idfa&app_id=...&country=...&idfa=...&click_id=...&...


SKAdNetwork-based postback:

http://postback.adnetwork.com/conversion/install?src=MMP&
	attribution_type=skadnetwork&app_id=...&country=...&
            skan_postback=b8Qb9BPWPi+ixk\/OiRys...







