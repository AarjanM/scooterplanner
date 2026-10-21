# European alternatives to Google AdMob for a small app

Research note for issue #19 (wayfinder:research). All sources were read on
2026-10-21 unless stated otherwise; every claim carries the URL it was taken
from. Where a vendor publishes nothing on a point, the note says "none
published" instead of estimating. Items marked **UNVERIFIED** could not be
confirmed from a primary source.

Scope: ScooterPlanner is a free Android + iOS app (React Native + Expo) for
Dutch Bromfiets and Snorfiets riders. Ads appear only on planning screens: a
banner, plus one interstitial after a Route plan is computed. The companion web
planner has one display slot. Target 5k-20k monthly active users (MAU), EU
hosting posture, consent through a TCF 2.3 CMP (Google UMP assumed).

## Summary and recommendation

**Recommendation: AdMob only for v1, with the option to switch on EU demand
through AdMob bidding later. Do not integrate an EU vendor's own SDK.**

Why, in short:

1. No EU-owned network offers self-serve onboarding for a 5k-20k MAU app.
   Every EU candidate with an in-app product (Equativ, Azerion/Improve Digital,
   Verve, Adform, Seedtag, YOC) is sales-led: "contact us", a partnership
   agreement, or an account manager who hands out IDs. None publishes a minimum
   traffic threshold, so it is unknowable in advance whether a small app gets
   accepted.
2. Several vendors on the ticket's list are not EU-owned any more: Teads is a
   Delaware company listed on Nasdaq and headquartered in New York; Ogury is a
   UK company; Adtelligent is a Delaware corporation; Appodeal is a Delaware
   corporation; Criteo's publisher SDK is deprecated and the company is
   redomiciling to Luxembourg with a stated path towards the US.
3. The one cheap EU path that exists is **AdMob bidding ("no third-party SDK
   required")**: Google lists Equativ (FR), Improve Digital (Azerion, NL), Verve
   Group and Smaato (Verve Group Media SE, IE/DE) as bidding-only ad sources
   that need no extra SDK, no React Native work, and reuse Google UMP consent.
   The catch: each still requires a signed partnership agreement that the vendor
   approves on unpublished criteria, and the money is paid by Google under AdMob
   terms, so this does not move payouts or data processing away from Google.
4. Nobody publishes eCPMs. The only independent EU figure found (Appodeal,
   games, Q4 2024, read off a chart) puts European interstitials around USD 3.6
   (Android) and USD 5.3 (iOS) and banners under USD 0.20. At 5k-20k MAU the
   whole ad budget of this app is roughly EUR 100-500 per month gross (see the
   envelope in section 7), which is below the effort of a second SDK and in some
   months below the EUR 70 AdMob payout threshold.

An "EU network only" build is not realistic today: the only EU vendor with an
official React Native package that covers banners and interstitials is Azerion's
BlueStack SDK, and it is a full mediation SDK that replaces AdMob, documented
for bare React Native only, with app registration through a publisher
representative.

## 1. Baseline: Google AdMob

What the EU candidates are compared against.

| Item | AdMob | Source |
|---|---|---|
| Onboarding | Self-serve account; no minimum traffic published | https://support.google.com/admob/answer/2772208 (thresholds page); onboarding issues page https://support.google.com/admob/answer/9905175 |
| Payout | EUR 70 payment threshold (USD 100), EUR 10 payment-method and cancellation thresholds; final payment after cancellation "within approximately 90 days" | https://support.google.com/admob/answer/2772208 |
| Revenue share | Not published for AdMob. AdMob policy page only says "AdMob revenue share will be paid only to the publisher". AdSense publishes 68% for display bought through Google Ads; AdMob does not | https://support.google.com/admob/answer/2753860 ; https://support.google.com/adsense/answer/32783 |
| Consent | Google UMP SDK "is IAB-certified for TCF v2.3 across all SDK versions"; Google LLC is CMP ID 300 on the IAB Europe CMP list (274 CMPs, list updated 2026-10-15) and on Google's own certified-CMP list (143 entries) | https://support.google.com/admob/answer/10113207 ; https://cmplist.consensu.org/v2/cmp-list.json ; https://support.google.com/admob/answer/13554116 |
| Consent for mediation partners | Consent is stored per TCF v2 in local storage (`IABTCF_*` keys); partners not on the GVL get Google's Additional Consent string; "Failure to [add partners] can lead to partners failing to serve ads on your app"; any Google-certified CMP may replace UMP | https://developers.google.com/admob/android/privacy/gdpr |
| React Native / Expo | `react-native-google-mobile-ads` 17.3.1 (published 2026-10-19, 114,357 downloads/week, maintainer Invertase). Ships an Expo config plugin (`app.plugin.js`; peer dep `expo >= 47`); "not supported by Expo Go", "You must use a development build". Wraps UMP as `AdsConsent`. Mediation adapters are added through `expo-build-properties` | https://registry.npmjs.org/react-native-google-mobile-ads ; https://api.npmjs.org/downloads/point/last-week/react-native-google-mobile-ads ; https://docs.page/invertase/react-native-google-mobile-ads/installation/expo ; https://docs.page/invertase/react-native-google-mobile-ads/european-user-consent ; https://docs.page/invertase/react-native-google-mobile-ads/mediation |
| Bidding partners without SDK | AdMob lists 23 "no third-party SDK required, bidding only" ad sources, among them Equativ (formerly Smart Adserver), Improve Digital, Smaato, Verve Group, OneTag Exchange. "To send an ad request to a bidding ad source, you must have an active partnership agreement with the ad source"; review "typically takes 2 business days"; "By signing this agreement, you consent to sharing your publisher ID with the ad source" | https://developers.google.com/admob/android/choose-networks ; https://support.google.com/admob/answer/10511265 ; https://support.google.com/admob/answer/11555701 |
| Published eCPM | None published | checked the pages above |

## 2. Comparison table

Columns follow the ticket. "RN" = React Native. "OB" = AdMob bidding without
a third-party SDK. GVL IDs are from the IAB TCF Global Vendor List v3,
version 181, 2026-10-15 (https://vendor-list.consensu.org/v3/vendor-list.json).

| Vendor | Ownership / HQ | Indie onboarding and thresholds | SDK or mediation path | RN / Expo | CMP requirements | Published eCPM | Payout terms |
|---|---|---|---|---|---|---|---|
| **Equativ** (ex Smart AdServer) | Equativ SAS, Paris; majority owner Bridgepoint (UK private equity) since 2023; private (https://www.equativ.com/legal-mentions, https://www.equativ.com/press/equativ-announces-bridgepoint-as-new-majority-shareholder-to-accelerate-scale-and-growth-of-the-independent-adtech-company) | No self-serve sign-up; help centre only covers login for existing accounts. OB route needs "an active partnership agreement with Equativ"; acceptance "takes on average 7 days"; Equativ "is not able to provide answers to individual publishers"; rejection reasons: brand safety, region. No minimum traffic published (https://help.equativ.com/set-up-google-open-bidding-connection, https://help.equativ.com/welcome-to-monetization-help-center) | (a) **AdMob OB, no SDK**; (b) Equativ Display SDK 8.6.2 (Android 2026-10-05; iOS min 17.6) with Equativ-maintained AdMob custom-event and MAX adapters; rewarded "not planning" (https://help-display-sdk.equativ.com/android/releasenotes.html, https://help-display-sdk.equativ.com/ios/gettingstarted.html, https://github.com/smartadserver/equativ-gma-mediation-adapter-android) | None official; only stale community `react-native-smartad` 1.6.5 (2020) (https://registry.npmjs.org/-/v1/search?text=smartadserver) | SDK reads `IABTCF_TCString` from SharedPreferences / NSUserDefaults; GPP takes priority; CMP must be IAB-compliant; GVL 45 (https://help-display-sdk.equativ.com/privacy.html) | None published | Direct: invoice requests only when "a minimum revenue of EUR/GBP/USD 200 is reached", payment date "defined according to contractual terms"; a "contractual CPM" may apply below Equativ's profitability threshold. Via OB: Google pays (https://help.equativ.com/manage-invoice-requests-and-invoices) |
| **Azerion / Improve Digital** | Azerion Group N.V., Schiphol-Rijk NL, Euronext Amsterdam AZRN (https://www.azerion.com/azerion-investor-relations/, Annual Report 2025) | "Request access" form (name, company, country, web or app); no self-serve; BlueStack console app ID "from your Azerion Publisher Representative". No minimum published (https://www.improvedigital.com/contact, https://docs.app.improvedigital.com/react-native) | (a) **AdMob OB, no SDK** ("Improve Digital"); (b) BlueStack SDK 6.1.2 (Android 2026-10-07), itself a mediation SDK that replaces AdMob and mediates Google, Equativ, BidMachine, AppLovin (https://docs.app.improvedigital.com/android, https://docs.app.improvedigital.com/releases) | **Official** `@azerion/bluestack-sdk-react-native` 6.1.0 (published 2026-10-20, 127 downloads/month, peer RN >= 0.80, no Expo config plugin, Expo not documented) (https://registry.npmjs.org/@azerion%2Fbluestack-sdk-react-native) | "Integrate any CMP that supports TCF v2"; whether SDK reads `IABTCF_*` keys: not stated. SSP returns no-ad without a TC string; GVL 253 (Improve Digital), 610 (Azerion Holding) (https://developers.bluestack.app/android/privacy, https://azerion-advertising.atlassian.net/wiki/spaces/imp/pages/459537) | None published | None published |
| **Verve Group** (HyBid / PubNative, Smaato) | Verve Group Media SE, registered office moved Stockholm to Dublin on 2026-10-02; listed Frankfurt (VRV) and Nasdaq First North Stockholm (VER); GVL vendor is Verve Group Europe GmbH (DE) 512, Smaato Inc. 82 (https://investors.verve.com/regulatory-news/the-registered-office-of-verve-group-media-se-has-been-successfully-relocated-from-sweden-to-ireland/) | Contact-sales ("Get started" leads to a contact form); Smaato "Request Access". No minimum published (https://verve.com/publishers/, https://www.smaato.com/get-started/) | (a) **AdMob OB, no SDK** ("Verve Group", "Smaato"); (b) HyBid SDK 3.9.3 (2026-10-09) with a Google-listed AdMob adapter `com.google.ads.mediation:verve:3.9.3.0` (bidding "in alpha", "contact Verve" for setup) and official MAX adapter (https://developers.google.com/admob/android/mediation/verve, https://github.com/pubnative/pubnative-hybid-android-sdk) | None official; community `react-native-hybidrn` (0 stars, 2026-06); `react-native-smaato` last 2018 | HyBid reads `IABTCF_TCString`, `IABTCF_gdprApplies`, `IABUSPrivacy_String`, `IABGPP_HDR_GppString` from SharedPreferences; Google: "automatically reads GDPR consent set by consent management platforms including UMP SDK" (https://raw.githubusercontent.com/pubnative/pubnative-hybid-android-sdk/main/hybid.sdk/src/main/java/net/pubnative/lite/sdk/UserDataManager.java, https://developers.google.com/admob/android/mediation/verve) | None published | Smaato/SPX only: default "70/30" net revenue share, "60-day payment schedule", USD 100 minimum, USD via Tipalti. HyBid marketplace terms: none published (https://developers-brand.verve.com/docs/publisher-faq-1) |
| **Adform** | Adform A/S, Copenhagen; private; ownership structure **UNVERIFIED** (annual report PDF not parseable) (https://site.adform.com/contact-us/) | Sales/Insertion Order only; "For Publishers" link is empty; sell-side T&C apply "together with the IO". No minimum published (https://site.adform.com/, https://site.adform.com/terms-and-conditions/adform-special-tc-sell-side/special-tc-version-10) | Not an AdMob partner. Adform Advertising SDK 2.22.0 (Dec 2025 commits) but Getting Started still targets compileSdk 20; AdMob custom event for native ads only (https://github.com/adform/adform-android-sdk, https://github.com/adform/adform-android-sdk/wiki/Adform-Native-Mediation-Adapter) | None | "IAB TCF 2.0" supported; SDK "will check CMP settings"; `IABTCF_*` keys not named; GVL 50 (https://github.com/adform/adform-android-sdk/wiki/GDPR) | None published | None published |
| **Seedtag** | Seedtag Advertising S.L., Madrid; private, Advent International-funded (https://www.seedtag.com/about-us/) | Contact-sales; commercial terms in a negotiated Purchase Order; no minimum published (https://www.seedtag.com/publishers/) | Not an AdMob or MAX partner. SDK example repos last touched 2021-08; no public SDK docs; Prebid Server adapter needs "your seedtag account team" (https://github.com/seedtag/seedtag-android-sdk-examples, https://docs.prebid.org/dev-docs/bidders/seedtag.html) | None | GVL 157; device-storage disclosure lists cookies only, empty `sdks` array (https://tcf.seedtag.com/vendor.json) | None published | None published |
| **YOC** | YOC AG, Berlin (https://yoc.com/imprint) | Contact-only; IDs "provided to you by YOC" (https://support.yoc.com/visx-sdk/overview/) | VIS.X SDK (Android 4.1.3 2024-04, iOS 4.1.6 2024-11); custom-event mediation with Google Mobile Ads; no MAX adapter (https://support.yoc.com/visx-sdk/release-notes/, https://github.com/yoc-ag/visx-sdk-open-mediation) | Vendor RN module `@yoc/react-native-visx-module` 1.4.1 (2024-02) per docs; **UNVERIFIED on npm**; iOS needs manual XCFramework drag-in (https://support.yoc.com/visx-sdk/react-native-module/) | "Working IAB TCF v2.0 certified CMP, with YOC AG (ID: 154) added as a Vendor" | None published | None published |
| **Teads** | Teads Holding Co., Delaware, HQ New York, Nasdaq TEAD; Altice Teads S.A. holds ~46% (FY2025 10-K: https://www.sec.gov/Archives/edgar/data/1454938/000145493826000015/tead-20251231.htm). **Not EU-owned** | Contact-sales; "Production PIDs come from your account manager" (https://sdk.teads.com/docs/android/Mediation/admob) | Teads SDK 6.2.3 (2026-09-17, "~1408.5 KiB"); AdMob custom-event adapters for banner and interstitial (https://sdk.teads.com/docs/android/Getting-Started/) | **Official** `teads-react-native` 6.2.0 (2026-06-22, 479 downloads/month) but only Feed and Media components, no banner or interstitial; Expo not mentioned (https://registry.npmjs.org/teads-react-native, https://sdk.teads.com/docs/react-native/Getting-Started/) | Reads IAB TC string automatically; "Outbrain requires partners to implement an IAB approved consent management platform"; GVL 132 (https://sdk.teads.com/docs/android/Getting-Started/privacy-guide) | None published | None published |
| **Ogury** | Ogury Ltd, England and Wales no. 08904951, London (https://www.ogury.com/terms-conditions/). **UK, not EU** | **Self-serve registration** at https://publishers.ogury.co/registration with human validation; vetting: monthly app updates, store reviews, non-free-mail contact, developer site with app-ads.txt; no traffic minimum stated (https://support.ogury.com/help-center-for-publishers/setting-up-your-account/prerequisites-and-vetting-process.md) | Ogury SDK 6.3.1 (Android 2026-08-06), custom AdMob adapters; on AppLovin's official MAX list (https://support.ogury.com/inapp/ogury-mediate/google-admob/adapter-setup.md, https://support.applovin.com/en/max/android/preparing-mediated-networks/) | None | "Ogury accepts only TCF v2.3 consent strings crafted after the 28th of February 2026"; reads `IABTCF_TCString`; GVL 31 (https://support.ogury.com/help-center-for-publishers/setting-up-your-account/consent-management.md, https://support.ogury.com/inapp/privacy-and-consent/privacy-compliance/iab-compliance.md) | None published ("Expected CPM" undisclosed) | Invoices paid "within 60 days from the date of approval"; USD 50 minimum; amounts in USD, EUR converted at ECB monthly average (https://www.ogury.com/terms-conditions/) |
| **Adtelligent** | Adtelligent Inc., Delaware, "headquartered in United States"; Odesa engineering office (https://www.adtelligent.com/privacy-policy/, https://careers.adtelligent.com/). **Not EU** | SSP marketed as self-serve but sign-up page could not be loaded (**UNVERIFIED**); main CTA "Request Demo" (https://www.adtelligent.com/ssp/) | iOS-only binary SDK 0.5.0 (pre-1.0); Android repo "is empty"; no AdMob or MAX adapter (https://github.com/Adtelligent/AdtelligentSwiftSDK, https://github.com/Adtelligent/In_App_Ads_SDK_Android_Public) | None | GVL 410; SDK consent handling: none published | None published | None published |
| **Criteo** | Criteo S.A., Paris, Nasdaq CRTO; redomiciling to Luxembourg, possible later US move (https://www.criteo.com/news/press-releases/2025/10/criteo-announces-intention-to-redomicile-to-luxembourg-and-list-ordinary-shares-on-nasdaq/) | Contact-only | Publisher SDK "deprecated as of September 30, 2025" (https://github.com/criteo/android-publisher-sdk) | None | GVL 91 | None published | None published |
| **Appodeal** (reference only) | Appodeal, Inc., Delaware, McLean VA (https://www.appodeal.com/terms-of-service). **Not EU** | Self-serve | Mediation SDK 4.3.0; mediates Ogury, Smaato, Verve among others | Official `react-native-appodeal` 4.3.0 (2026-09-12); **no Expo config plugin** (https://registry.npmjs.org/react-native-appodeal, https://faq.appodeal.com/en/articles/94944-frameworks-supported) | Stack Consent Manager built on Google UMP (CMP 432) | None published for NL | Net 45; USD 100 minimum (PayPal/ACH), USD 1,000 wire (https://faq.appodeal.com/en/articles/1838672-appodeal-s-payment-net-term, https://faq.appodeal.com/en/articles/1838761-minimum-payout-amount) |
| **Adnuntius** | Adnuntius AS, Oslo (NO, EEA not EU) (https://adnuntius.com/company) | Paid SaaS ad server, tiered pricing with no public figures, sales form (https://adnuntius.com/build-your-own) | Android SDK 1.7.3 (2025-04), iOS 1.10.3 (2024-04); no AdMob/MAX adapter | None (`@adnuntius/react-hooks` is web) | TCF v2 relied on; GVL 855 | None published | Not applicable (you pay them) |
| **Opt Out Advertising** | Opt Out Advertising B.V., Amstelveen NL; contextual, consentless (https://www.optoutadvertising.com/publishers) | Contact form only; no pricing or minimums published | No SDK; "In-App" listed as a channel without an SDK. **Web and server-side only** | None | Not applicable (no personal data); GVL 227 | None published | None published |
| **Kobler** | Kobler, Norway; contextual | Set up by Kobler via Prebid/Livewrapped/RTB; Nordic editorial focus (https://kobler.no/en/publishers) | No SDK. **Web only** | None | Not in GVL by design | None published | None published |

## 3. Answers to the ticket's numbered points

### 3.1 Candidates, ownership and HQ

See the table. The EU-owned candidates with any in-app product are **Equativ
(FR)**, **Azerion/Improve Digital (NL)**, **Verve Group (IE/DE)**, **Adform
(DK)**, **Seedtag (ES)** and **YOC (DE)**. Of the names on the ticket, **Teads**
(US, Nasdaq, ~46% Altice), **Ogury** (UK), **Adtelligent** (US Delaware) and
**Criteo** (FR to LU, SDK deprecated) fail the "European-owned" test or are
unusable. Contextual/privacy-first options (Seedtag, Opt Out Advertising,
Kobler) are web-first; none offers an in-app SDK an indie can obtain.

### 3.2 Thresholds, indie onboarding, payout, first-party data

- **Minimum traffic or revenue thresholds:** none published by any EU vendor
  (checked every publisher and terms page cited in the table). The only numeric
  thresholds found are payout minimums: Equativ EUR/GBP/USD 200 per invoice
  request (direct contracts), Smaato USD 100, Ogury USD 50, AdMob EUR 70.
- **Onboarding:** every EU vendor is sales-led. Equativ via AdMob bidding is
  the closest to a form-based flow, but Equativ states it does not answer
  individual publishers. Ogury (UK) is the only one with a public registration
  page, and it is still human-validated with vetting rules (monthly updates,
  non-free-mail address, store reviews) that a hobby app may fail.
- **Payout terms:** published only by Equativ (EUR 200 invoice minimum, net
  days contractual), Smaato (70/30, net 60, USD 100, USD only), Ogury (net 60
  after invoice approval, USD 50, USD). Adform, Azerion, Seedtag, YOC, Verve
  HyBid: none published.
- **App with no first-party user data:** no vendor states a first-party data
  requirement. All in-app SDKs key on the advertising ID (GAID/IDFA) gated by
  TCF consent and, on iOS, ATT. Equativ's SDK withholds UID, IFA and IFV
  without consent (https://help-display-sdk.equativ.com/privacy.html); Improve
  Digital's SSP "will return a no-ad for any impressions that do not contain" a
  TC string. So without consent these networks serve nothing or contextual
  only, which is the same outcome as AdMob's limited ads.

### 3.3 SDK footprint, React Native / Expo, mediation reach

- **Footprint:** only Teads publishes a size ("~1408.5 KiB" on Android). All
  others: none published.
- **React Native:** official packages exist only from Azerion
  (`@azerion/bluestack-sdk-react-native` 6.1.0, 2026-10-20, 127 downloads/month)
  and Teads (`teads-react-native` 6.2.0, feed/media only). Neither documents
  Expo; neither ships an Expo config plugin (verified on the npm tarball
  manifest for Azerion). Any of them would need `expo prebuild` and a
  development build, as `react-native-google-mobile-ads` already does.
- **Through AdMob mediation instead of their own SDK:** yes, for Equativ,
  Improve Digital, Verve Group and Smaato, as bidding-only ad sources that need
  no third-party SDK (https://developers.google.com/admob/android/choose-networks).
  This is the only path that is zero native code. With-SDK adapters exist for
  Verve (Google-listed, bidding alpha), Equativ, Teads, Ogury, YOC (vendor
  custom events); Adform has a native-ads-only custom event; Seedtag,
  Adtelligent, Adnuntius have none.
- **Other open mediation layers:** AppLovin MAX officially lists Verve, Smaato
  and Ogury (https://support.applovin.com/en/max/android/preparing-mediated-networks/);
  Equativ and Teads ship their own MAX adapters. Prebid Server has adapters for
  Adform, Equativ, Teads, Ogury, Seedtag, Improve Digital, Adtelligent, but all
  require a vendor account first, and Prebid Mobile would mean running a Prebid
  Server: more infrastructure, not less.

### 3.4 Consent

- **Required CMP:** Equativ, Teads, Verve/HyBid, Ogury and Adform read the IAB
  in-app keys (`IABTCF_TCString`, `IABTCF_gdprApplies`) written by any TCF CMP.
  Google UMP is TCF 2.3 certified (CMP ID 300), so **its TC string is accepted
  by all of them**; Google's Verve page says so explicitly ("including UMP
  SDK"). YOC demands a "TCF v2.0 certified CMP with YOC AG (ID 154) added as a
  vendor"; Ogury only accepts TCF 2.3 strings created after 2026-02-28 and asks
  publishers to load its full vendor list. Azerion BlueStack: "any CMP that
  supports TCF v2"; whether it reads the IAB keys is not stated
  (**UNVERIFIED**).
- **What must be configured in UMP:** every partner has to be in the GDPR
  message's ad-partner list, otherwise "partners [may fail] to serve ads on your
  app" (https://developers.google.com/admob/android/privacy/gdpr). All EU
  candidates are on the GVL (Equativ 45, Improve Digital 253, Verve Group
  Europe 512, Smaato 82, Adform 50, Seedtag 157, YOC 154, Teads 132, Ogury 31),
  so the TC string suffices; no Additional Consent entries are needed.
- **Data collected by the SDKs (per their own docs):** Equativ iOS privacy
  table marks Device ID and Advertising Data as linked to the user and used for
  tracking, coarse location required, precise location optional
  (https://help-display-sdk.equativ.com/ios/appprivacydetails.html). HyBid
  collects the ad ID "when made available by the user", impressions/clicks,
  and location only if the app holds the permission, on by default
  (`HyBid.setLocationTrackingEnabled(false)` to disable)
  (https://developers.verve.com/docs/google-play-data-safety-guidance.md).
  Ogury: Device ID and Advertising Data linked to user, no location
  (https://support.ogury.com/inapp/privacy-and-consent/privacy-compliance/apple-privacy.md).
  Teads: requests the Android Advertising ID; no data-safety table. BlueStack:
  optional fine/coarse location permissions "improve targeting"; nothing
  itemised. Via AdMob bidding, no extra SDK runs, so the data surface stays
  Google's.
- **EU hosting posture:** Improve Digital states EU traffic is processed in
  Frankfurt and Dublin data centres
  (https://azerion-advertising.atlassian.net/wiki/spaces/imp/pages/459537).
  Equativ invoices from Equativ SAS (Paris). No other vendor publishes
  processing locations on the pages checked.

### 3.5 Expected eCPM for Dutch traffic

- **Vendors:** none publishes any eCPM or CPM figure (every vendor site and
  help centre cited above was checked). AdMob publishes none either.
- **Independent source:** Appodeal, "The Latest eCPM Report 2025", Oct-Dec
  2024, "100,000+ apps & 70+ ad networks", "200+ billion ad views", **mobile
  games only**
  (https://appodeal.com/wp-content/uploads/2025/03/Appodeal-The-Latest-eCPM-Report-2025.pdf).
  The Netherlands is not in its top-10 country chart. Read off the "eCPM by
  Region & Ad Format" chart (page 8; values are approximate, in USD):

  | Europe, Q4 2024 | Banner | Interstitial |
  |---|---|---|
  | Android | ~0.15 | ~3.6 |
  | iOS | ~0.2 | ~5.3 |

  Caveats: games inventory (a utility app has different demand), Q4 is the
  seasonal peak, a regional average, chart-read values. Treat as an order of
  magnitude, not a forecast. Appodeal's newer "Benchmarks" tool is
  login-gated and was not consulted.
- **Web (for section 4):** EthicalAds publishes "~USD 2.50 CPM" for EU/NA
  traffic on developer sites (https://www.ethicalads.io/publishers/); not
  transferable to a moped planner, but it is the only published EU-relevant
  web CPM found.

### 3.6 Web planner slot

See section 4.

## 4. The web planner display slot

Baseline: Google AdSense has no minimum traffic (eligibility is original
content, HTML access, policy compliance, 18+;
https://support.google.com/adsense/answer/9724), pays at EUR 70 / USD 100
(https://support.google.com/adsense/answer/1709871), between the 21st and 26th
of the following month (https://support.google.com/adsense/answer/1709858).

EU-owned options checked for a site expected to stay under 100k pageviews a
month:

| Option | Country | Finding | Realistic for one slot? |
|---|---|---|---|
| Equativ web | FR | "Contact us" only; EUR 200 invoice minimum on direct contracts (https://equativ.com/publishers/, https://help.equativ.com/manage-invoice-requests-and-invoices) | No |
| Adform | DK | Sales/IO only (https://site.adform.com/) | No |
| Opt Out Advertising | NL | Consentless contextual ad server, "implementation within 5 working days", contact form, no pricing or minimums published (https://www.optoutadvertising.com/publishers) | Unknown; the one EU vendor worth a direct email. **UNVERIFIED** whether a small site is accepted |
| Adnuntius | NO | Paid SaaS ad server with optional marketplace demand; tiered pricing, no public figures (https://adnuntius.com/build-your-own) | No (you would pay) |
| Kobler | NO | Contextual, Nordic editorial media, set up by Kobler (https://kobler.no/en/publishers) | No |
| Yieldlove (Ströer) | DE | Header-bidding for publishers, "free demo" contact (https://www.yieldlove.com/en-gb/company/) | No |

Non-EU references, for scale: Ezoic requires 250k monthly users
(https://www.ezoic.com/pricing/); Mediavine's Journey tier requires 1,000
Tier-1 sessions per 30 days (https://www.mediavine.com/introducing-journey-new-ad-management/);
EthicalAds requires 50k+ pageviews/month, pays 70% with a USD 50 minimum, and
must be the only ad on the page (https://www.ethicalads.io/publishers/,
https://www.ethicalads.io/publisher-policy/); Carbon Ads is invite/manual
review for developer and design sites (https://www.carbonads.net/join).

**Conclusion for the web slot:** no EU-owned option fills one display slot on a
sub-100k-pageview site self-serve. AdSense is the only self-serve, no-minimum
choice. Opt Out Advertising (NL, contextual, no consent banner needed for its
ads) is the one EU vendor worth asking; the answer is unknown.

## 5. The three options compared

| | AdMob only | AdMob + EU network through AdMob bidding | EU network only |
|---|---|---|---|
| Native code beyond `react-native-google-mobile-ads` | None | None (bidding-only partners) | A second SDK: BlueStack RN (bare RN, no Expo plugin) or a hand-written Expo module for Equativ / HyBid |
| Onboarding effort | Self-serve, PIN verification up to 3 weeks | Plus one partnership request per partner; vendor approval on unpublished criteria, "2 business days" (Google) to "7 days" (Equativ); may be rejected without explanation | Sales conversation; account manager issues IDs; no published acceptance criteria for a 5k-20k MAU app |
| Where the money comes from | Google, EUR 70 threshold | Google, same threshold (EU partners bid into Google's auction) | Vendor; Equativ EUR 200 invoice minimum; Smaato USD 100 net 60 in USD; Azerion none published |
| EU posture gained | None | Demand partly EU-owned; data processing, consent and payment still Google | Data processing and payment move to the EU vendor; consent still needs a TCF CMP (UMP or an EU CMP such as Didomi 7, Usercentrics 5, consentmanager 31, all Google-certified) |
| Revenue effect | Baseline | Unknown; more bidders can only raise the clearing price, but no vendor publishes figures and Equativ cannot set floors ("Any settings must be made in the Google UI") | Unknown; single-network fill for Dutch utility traffic is the risk, and no vendor publishes NL eCPMs |
| Reversibility | - | Toggle in AdMob console | Rewrite of the ad layer |

## 6. Recommendation and trade-off

**Ship v1 with AdMob only** (`react-native-google-mobile-ads` with the Expo
config plugin and a development build, UMP for TCF 2.3, banner on planning
screens, one interstitial after a Route plan is computed). Keep the ad layer
behind one small module so a network change does not touch planning code.

**After launch, once the app has a few months of AdMob data**, request Equativ,
Improve Digital and Verve Group / Smaato as AdMob bidding sources. Cost: four
forms and four `app-ads.txt` lines (Equativ's are
`smartadserver.com, 4194-OB, RESELLER, 060d053dcf45cbf3` and
`smartadserver.com, 4194, RESELLER, 060d053dcf45cbf3`); no code. Benefit:
whatever uplift extra EU demand brings, measurable in the AdMob console. Honest
framing: this is EU demand, not EU infrastructure. Consent, payment and the SDK
stay with Google.

**Do not build an EU-network-only ad layer now.** The revenue at stake (section
7) does not cover the engineering of a second native SDK without Expo support,
the vendor negotiation, or the risk of unfilled Dutch inventory. Revisit if the
app grows well past 20k MAU or if a vendor opens self-serve app onboarding.

**For the web planner slot:** AdSense, or no ad. Send one enquiry to Opt Out
Advertising (NL, contextual); if they accept small sites, their consentless
model would also remove the need for a cookie banner on the web planner, which
is a user-experience gain independent of revenue.

## 7. Revenue envelope (assumptions, not data)

All inputs below are assumptions or chart-read figures; none is vendor data for
this app.

- MAU 5k-20k (ticket). Assume 6 Route plans per MAU per month (assumption; no
  usage data exists yet) and one interstitial per Route plan: 30k-120k
  interstitials per month. Assume 2 banner impressions per planning session:
  60k-240k banner impressions per month.
- eCPM from Appodeal Europe Q4 2024 games data (section 3.5): interstitial
  blended ~USD 4, banner ~USD 0.17.
- Gross: interstitials USD 120-480, banners USD 10-40, so roughly **USD
  130-520 (EUR 110-450) per month**, before any policy-driven limited-ads share
  for users who decline consent, and before seasonality (Q4 figures are the
  high point).
- At the low end this does not clear the EUR 70 AdMob threshold every month.
  At the high end it is a few hundred euros, which does not pay for a second
  SDK integration or a vendor contract with a EUR 200 invoice minimum.

## 8. Unverified items and open questions

Unverified (no primary source found):

- Adform's ownership structure (annual report PDF not text-extractable).
- Whether Azerion's BlueStack SDK reads the IAB `IABTCF_*` keys or needs its
  own consent call.
- Whether YOC's React Native module is published on npm (docs describe it;
  registry search found nothing).
- Whether Adtelligent's SSP has a public sign-up (page blocked by anti-bot).
- Whether Opt Out Advertising accepts a sub-100k-pageview site.
- Whether a 5k-20k MAU app is accepted as a bidding partner by Equativ,
  Improve Digital or Verve; no vendor publishes criteria.
- Appodeal eCPM values are read off a chart image, not a table.
- Which AdMob bidding partners can be signed inside the AdMob console versus
  on the vendor's site; Google's table uses icons that did not survive text
  extraction. Google's prose names AppLovin, Meta, Unity, PubMatic and
  Ad Generation as in-console; the EU partners are not named in prose.

Questions that should become their own tickets:

1. Ad-layer module boundary: define the one module that owns banner and
   interstitial placement so the network can change without touching Route
   plan code (ties to the v1 spec).
2. Consent architecture: UMP alone, or an EU CMP (Didomi, Usercentrics,
   consentmanager are all Google-certified and TCF-registered) feeding both
   AdMob and any later EU partner; decide before the first store release
   because the message set-up is per app.
3. Interstitial policy check: one interstitial after a Route plan is computed
   must fit AdMob's interstitial placement rules and Google Play / App Store
   policies for navigation apps; not researched here.
4. Web planner: decide AdSense versus no ads versus a consentless contextual
   slot (Opt Out Advertising enquiry) in light of the cookie-banner trade-off.
5. Revisit trigger: set a MAU or monthly-revenue threshold at which EU bidding
   partners are requested in AdMob, and what metric decides whether to keep
   them.
