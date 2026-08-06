---
name: Create a purchasable Zazzle product from a template
description: >-
  Turn an image URL and/or a line of text from your own site into a real, purchasable Zazzle product
  by building a Create-a-Product linkover URL, and render a RealView preview of it first.
api: openapi/zazzle-create-a-product-openapi.yml
operations:
  - createProductFromTemplate
  - getRealViewProductImage
generated: '2026-08-05'
method: generated
source: https://asset.zcache.com/assets/graphics/z4/uniquePages/zAPI/ZazzleApiGuide.v3.pdf
---

# Create a purchasable Zazzle product from a template

## What this API actually is

There is no `POST /products`. Zazzle's Create-a-Product API is a **URL contract**: you construct a
`GET` link, and the "response" is the rendered Zazzle product page the buyer lands on. Nothing is
created server-side until the buyer arrives. Treat the output of this skill as a **URL you emit**,
not a call you await.

## Before you can call it

Four out-of-band gates must be satisfied on the Zazzle account whose ID you put in the URL. None of
them is checkable through the API — if they are unmet the linkover simply fails at render time.

1. A Zazzle account — https://www.zazzle.com/lgn/registration
2. Enrollment in the Associates Program — https://www.zazzle.com/my/associate/associate
3. Acceptance of the Create-a-Product API terms — https://www.zazzle.com/my/associate/create_a_product_api_signup
4. **Declaration of every domain your image URLs are served from** — https://www.zazzle.com/my/associate/domains

Gate 4 is the one that bites. An image URL on an undeclared host produces
`Zazzle API Error: Image failed to upload`.

You also need at least one **template product**: a Zazzle product posted for sale with
`Make product a template = Yes`, whose image and text objects have been marked as template objects
and given URL Parameter Names.

## Step 1 — render a preview first (`getRealViewProductImage`)

Always preview before you emit a buy link. It is a cheap, idempotent `GET` that catches a bad
template ID, a bad image URL, or a wrong parameter name before a customer ever sees it.

```
GET https://rlv.zazzle.com/svc/view
  ?pid=<18-digit template product id>
  &max_dim=512
  &at=<18-digit member account id>
  &t_image1_url=<percent-encoded image URL>
  &t_text1_txt=<percent-encoded text>
  &t_text1_txtclr=000000
```

* `max_dim` must be between 10 and 2212. **Anything above 699 comes back with a Zazzle-branded
  frame** — use 512 or 640 if you want a clean image.
* The image parameter here is `t_image1_url`. On the linkover in Step 2 the *same object* is
  `t_image1_iid`. Getting this backwards is Zazzle's own most-listed support issue.

A bare `GET https://rlv.zazzle.com/svc/view` with no parameters returns **406**.

## Step 2 — build the linkover (`createProductFromTemplate`, `ax=Linkover`)

One template in, one product out.

```
GET https://www.zazzle.com/api/create/at-<member account id>
  ?ax=Linkover
  &pd=<18-digit template product id>
  &rf=<associate id>          # optional, for referral attribution
  &ed=true                    # let the buyer personalize further
  &tc=<tracking code>         # optional, shows in your referral history
  &ic=<image code>            # optional, shows in your royalty history
  &t_image1_iid=<percent-encoded image URL>
  &t_text1_txt=<percent-encoded text>
  &t_text1_txtclr=4169e1
```

## Step 3 — one link, many products (`ax=DesignBlast`)

Same operation, different mode. Instead of one template (`pd`), point at a whole store category and
Zazzle renders a "Templates Buffet" page applying your content across every template in it.

```
GET https://www.zazzle.com/api/create/at-<member account id>
  ?ax=DesignBlast
  &sr=<18-digit store id>
  &cg=<18-digit category id>
  &ed=true
  &continueUrl=<percent-encoded URL for the Go Back link>
  &t_image1_iid=<percent-encoded image URL>
  &t_text1_txt=<percent-encoded text>
```

Constraints Zazzle documents for this mode:

* The default **New Products** category cannot be used. Create a purpose-made category.
* Templates in it must be **Public** and **G-Rated**.
* The store ID (`sr`) is obtainable only through the in-account link builder at
  https://www.zazzle.com/my/associate/create.
* A new category or template can take **up to 24 hours** to become usable.

## Parameter naming — the rule that is not obvious

Template object parameters are **not a fixed list**. Each is named after the *URL Parameter Name* you
assigned to that object in Zazzle's design tool. Defaults increment: `image1`, `image2`, `text1`,
`text2` → `t_image1_iid`, `t_text1_txt`. Rename an object to `photo` and its parameter becomes
`t_photo_iid`.

If you plan to use DesignBlast, give the **same** parameter name to equivalent objects across every
template in the category, or one link cannot fill them all.

## Rules that will burn you

* **Percent-encode everything** — every image URL, every text value, `continueUrl`. An unencoded
  image URL is a documented cause of upload failure.
* **`continueUrl` is case-sensitive.** `continueurl` silently fails.
* **Templates are immutable once posted for sale.** To change a design you must create a new product
  and delete the old one — so validate a handful of templates thoroughly before authoring many.
* **Nothing here is idempotent in the transactional sense, but nothing here is a write either.** All
  three calls are safe to repeat. See `conventions/zazzle-conventions.yml`.

## Errors

There is no structured error body on this surface. Failures render as text on the Zazzle page. The
catalog of documented failure modes is in `errors/zazzle-problem-types.yml`.

## Reference

* Provider guide: https://asset.zcache.com/assets/graphics/z4/uniquePages/zAPI/ZazzleApiGuide.v3.pdf
* Working example values and live example links: `sandbox/zazzle-sandbox.yml`
* Help desk: api.help@zazzle.com
