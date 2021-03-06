# Prebid Server Auction Endpoint

This document describes the behavior of the Prebid Server auction endpoint, including:

- Request/response formats
- OpenRTB extensions
- Debugging and performance tips
- How user syncing works
- Departures from OpenRTB

## `POST /openrtb2/auction`

This endpoint runs an auction with the given OpenRTB 2.5 bid request.

### Sample request

The [Prebid sample ad](http://prebid.org/examples/pbjs_demo.html) can be loaded with the request sample [here](../../../endpoints/openrtb2/sample-requests/valid-whole/exemplary/prebid-test-ad.json).

Other examples can be found in [endpoints/openrtb2/sample-requests/valid-whole/exemplary](../../../endpoints/openrtb2/sample-requests/valid-whole/exemplary).

### Sample Response

This endpoint will respond with either:

- An OpenRTB 2.5 BidResponse, or
- An HTTP 400 status code if the request is malformed

A "hello world" response from the prebid sample ad request is shown below.

```
{
  "id": "some-request-id",
  "seatbid": [
    {
      "seat": "appnexus"
      "bid": [
        {
          "id": "4625436751433509010",
          "impid": "some-impression-id",
          "price": 0.5,
          "adm": "<script type=\"application/javascript\">... the creative javascript is in here ... </script>",
          "adid": "29681110",
          "adomain": [
            "appnexus.com"
          ],
          "iurl": "http://nym1-ib.adnxs.com/cr?id=29681110",
          "cid": "958",
          "crid": "29681110",
          "w": 300,
          "h": 250,
          "ext": {
            "bidder": {
              "appnexus": {
                "brand_id": 1,
                "auction_id": 6127490747252133000,
                "bidder_id": 2
              }
            }
          }
        }
      ]
    }
  ]
}
```

### OpenRTB Extensions

#### Conventions

OpenRTB 2.5 permits exchanges to define their own extensions to any object from the spec.
These fall under the `ext` property of JSON objects.

If `ext` is defined on an object, Prebid Server uses the following conventions:

1. `ext` in "Request objects" uses `ext.prebid` and/or `ext.{anyBidderCode}`.
2. `ext` on "Response objects" uses `ext.prebid` and/or `ext.bidder`.
The only exception here is the top-level `BidResponse`, because it's bidder-independent.

`ext.{anyBidderCode}` and `ext.bidder` extensions are defined by bidders.
`ext.prebid` extensions are defined by Prebid Server.

Exceptions are made for extensions with "standard" recommendations:

- `request.user.ext.digitrust` -- To support Digitrust support
- `request.regs.ext.gdpr` and `request.user.ext.consent` -- To support GDPR
- `request.site.ext.amp` -- To identify AMP as the request source

#### Bid Adjustments

Bidders [are encouraged](../../developers/add-new-bidder.md) to make Net bids. However, there's no way for Prebid to enforce this.
If you find that some bidders use Gross bids, publishers can adjust for it with `request.ext.prebid.bidadjustmentfactors`:

```
{
  "appnexus: 0.8,
  "rubicon": 0.7
}
```

This may also be useful for publishers who want to account for different discrepancies with different bidders.

#### Targeting

Targeting refers to strings which are sent to the adserver to
[make header bidding possible](http://prebid.org/overview/intro.html#how-does-prebid-work).

`request.ext.prebid.targeting` is an optional property which causes Prebid Server
to set these params on the response at `response.seatbid[i].bid[j].ext.prebid.targeting`.

**Request format** (optional param `request.ext.prebid.targeting`)

```
{
    "pricegranularity": {
        "precision": 2,
        "ranges": [
            {
                "max":20.00,
                "increment":0.10 // This is equivalent to the deprecated "pricegranularity": "medium"
            }
        ]
    },
    "includewinners": false // Optional param defaulting to true
    "includebidderkeys": false // Optional param defaulting to true
}
```
The list of price granularity ranges must be given in order of increasing `max` values. If `precision` is omitted, it will default to `2`. The minimum of a range will be 0 or the previous `max`. Any cmp above the largest `max` will go in the `max` pricebucket.

For backwards compatibility the following strings will also be allowed as price granularity definitions. There is no guarantee that these will be honored in the future. "One of ['low', 'med', 'high', 'auto', 'dense']" See [price granularity definitions](http://prebid.org/prebid-mobile/adops-price-granularity.html)

One of "includewinners" or "includebidderkeys" must be true (both default to true if unset). If both were false, then no targeting keys would be set, which is better configured by omitting targeting altogether.

**Response format** (returned in `bid.ext.prebid.targeting`)

```
{
  "hb_bidder_{bidderName}": "The seatbid.seat which contains this bid",
  "hb_size_{bidderName}": "A string like '300x250' using bid.w and bid.h for this bid",
  "hb_pb_{bidderName}": "The bid.cpm, rounded down based on the price granularity."
}
```

The winning bid for each `request.imp[i]` will also contain `hb_bidder`, `hb_size`, and `hb_pb`
(with _no_ {bidderName} suffix). To prevent these keys, set `request.ext.prebid.targeting.includeWinners` to false.

**NOTE**: Targeting keys are limited to 20 characters. If {bidderName} is too long, the returned key
will be truncated to only include the first 20 characters.

#### Cookie syncs

Each Bidder should receive their own ID in the `request.user.buyeruid` property.
Prebid Server has three ways to popualte this field. In order of priority:

1. If the request payload contains `request.user.buyeruid`, then that value will be sent to all Bidders.
In most cases, this is probably a bad idea.

2. The request payload can store a `buyeruid` for each Bidder by defining `request.user.ext.prebid.buyeruids` like so:

```
{
  "appnexus": "some-appnexus-id",
  "rubicon": "some-rubicon-id"
}
```

Prebid Server's core logic will preprocess the request so that each Bidder sees their own value in the `request.user.buyeruid` field.

3. Prebid Server will use its Cookie to map IDs for each Bidder.

If you're using [Prebid.js](https://github.com/prebid/Prebid.js), this is happening automatically.

If you're using another client, you can populate the Cookie of the Prebid Server host with User IDs
for each Bidder by using the `/cookie_sync` endpoint, and calling the URLs that it returns in the response.

#### Native Request

For each native request, the `assets` objects's `id` field must not be defined. Prebid Server will set this automatically, using the index of the asset in the array as the ID.


#### Bidder Aliases

Requests can define Bidder aliases if they want to refer to a Bidder by a separate name.
This can be used to request bids from the same Bidder with different params. For example:

```
{
  "imp": [
    {
      "id": "some-impression-id",
      "video": {
        "mimes": ["video/mp4"]
      },
      "ext": {
        "appnexus: {
          "placementId": 123
        },
        "districtm": {
          "placementId": 456
        }
      }
    }
  ],
  "ext": {
    "prebid": {
      "aliases": {
        "districtm": "appnexus"
      }
    }
  }
}
```

For all intents and purposes, the alias will be treated as another Bidder. This new Bidder will behave exactly
like the original, except that the Response will contain seprate SeatBids, and any Targeting keys
will be formed using the alias' name.

If an alias overlaps with a core Bidder's name, then the alias will take precedence.
This prevents breaking API changes as new Bidders are added to the project.

For example, if the Request defines an alias like this:

```
{
  "aliases": {
    "appnexus": "rubicon"
  }
}
```

then any `imp.ext.appnexus` params will actually go to the **rubicon** adapter.
It will become impossible to fetch bids from Appnexus within that Request.

#### Bidder Response Times

`response.ext.responsetimemillis.{bidderName}` tells how long each bidder took to respond.
These can help quantify the performance impact of "the slowest bidder."

#### Bidder Errors

`response.ext.errors.{bidderName}` contains messages which describe why a request may be "suboptimal".
For example, suppose a `banner` and a `video` impression are offered to a bidder
which only supports `banner`.

In cases like these, the bidder can ignore the `video` impression and bid on the `banner` one.
However, the publisher can improve performance by only offering impressions which the bidder supports.

For example, a request may return this in `response.ext`

```
{
  "errors": {
    "appnexus": [
      {
        "code": 2,
        "message": "A hybrid Banner/Audio Imp was offered, but Appnexus doesn't support Audio."
      }
    ],
    "rubicon": [
      {
        "code": 1,
        "message": "The request exceeded the timeout allocated"
      }
    ]
  }
}
```

The codes currently defined are:

```
0   NoErrorCode
1   TimeoutCode
2   BadInputCode
3   BadServerResponseCode
999 UnknownErrorCode
```

#### Debugging

`response.ext.debug.httpcalls.{bidder}` will be populated **only if** `request.test` **was set to 1**.

This contains info about every request and response sent by the bidder to its server.
It is only returned on `test` bids for performance reasons, but may be useful during debugging.

`response.ext.debug.resolvedrequest` will be populated **only if** `request.test` **was set to 1**.

This contains the request after the resolution of stored requests and implicit information (e.g. site domain, device user agent).

#### Stored Requests

`request.imp[i].ext.prebid.storedrequest` incorporates a [Stored Request](../../developers/stored-requests.md) from the server.

A typical `storedrequest` value looks like this:

```
{
  "id": "some-id"
}
```

For more information, see the docs for [Stored Requests](../../developers/stored-requests.md).

#### Cache bids

Bids can be temporarily cached on the server by sending the following data as `request.ext.prebid.cache`:

```
{
  "bids": {},
  "vastxml": {}
}
```

Both `bids` and `vastxml` are optional, but one of the two is required. Thils property will have no effect
unless `request.ext.prebid.targeting` is also set in the request.

If `bids` is present, Prebid Server will make a _best effort_ to include these extra
`bid.ext.prebid.targeting` keys:

- `hb_cache_id`: On the highest overall Bid in each Imp.
- `hb_cache_id_{bidderName}`: On the highest Bid from {bidderName} in each Imp.

Clients _should not assume_ that these keys will exist, just because they were requested, though.
If they exist, the value will be a UUID which can be used to fetch Bid JSON from [Prebid Cache](https://github.com/prebid/prebid-cache).
They may not exist if the host company's cache is full, having connection problems, or other issues like that.

If `vastxml` is present, PBS will try to add analogous keys `hb_uuid` and `hb_uuid_{bidderName}`.
In addition to the caveats above, these will exist _only if the relevant Bids are for Video_.
If they exist, the values can be used to fetch the bid's VAST XML from Prebid Cache directly.

These options are mainly intended for certain limited Prebid Mobile setups, where bids cannot be cached client-side.

#### GDPR

Prebid Server supports the IAB's GDPR recommendations, which can be found [here](https://iabtechlab.com/wp-content/uploads/2018/02/OpenRTB_Advisory_GDPR_2018-02.pdf).

This adds two optional properties:

- `request.user.ext.consent`: Is the consent string required by the IAB standards.
- `request.regs.ext.gdpr`: Is 0 if the caller believes that the user is *not* under GDPR, 1 if the user *is* under GDPR, and undefined if we're not certain.

These fields will be forwarded to each Bidder, so they can decide how to process them.

### OpenRTB Differences

This section describes the ways in which Prebid Server **breaks** the OpenRTB spec.

#### Allowed Bidders

Prebid Server returns a 400 on requests which define `wseat` or `bseat`.
We may add support for these in the future, if there's compelling need.

Instead, an impression is only offered to a bidder if `bidrequest.imp[i].ext.{bidderName}` exists.

This supports publishers who want to sell different impressions to different bidders.

#### Deprecated Properties

This endpoint returns a 400 if the request contains deprecated properties (e.g. `imp.wmin`, `imp.hmax`).

The error message in the response should describe how to "fix" the request to make it legal.
If the message is unclear, please [log an issue](https://github.com/prebid/prebid-server/issues)
or [submit a pull request](https://github.com/prebid/prebid-server/pulls) to improve it.

#### Determining Bid Security (http/https)

In the OpenRTB spec, `request.imp[i].secure` says:

> Flag to indicate if the impression requires secure HTTPS URL creative assets and markup,
> where 0 = non-secure, 1 = secure. If omitted, the secure state is unknown, but non-secure
> HTTP support can be assumed.

In Prebid Server, an `https` request which does not define `secure` will be forwarded to Bidders with a `1`.
Publishers who run `https` sites and want insecure ads can still set this to `0` explicitly.

### See also

- [The OpenRTB 2.5 spec](https://www.iab.com/wp-content/uploads/2016/03/OpenRTB-API-Specification-Version-2-5-FINAL.pdf)
