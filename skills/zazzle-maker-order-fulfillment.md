---
name: Pull, acknowledge, and ship a Zazzle Maker order
description: >-
  Poll the Zazzle Vendor API for new orders, acknowledge them, fetch the packing sheet, and buy or
  void the carrier shipping label — with the safety rules the API's own design requires.
api: openapi/zazzle-vendor-openapi.yml
operations:
  - callVendorApiMethod
rpc_methods:
  - listneworders
  - listcancelledorders
  - getorder
  - getpackingsheet
  - getshippinglabel
  - voidshippinglabel
  - ackorder
generated: '2026-08-05'
method: generated
source: https://vendor.zazzle.com/v100/api.aspx
---

# Pull, acknowledge, and ship a Zazzle Maker order

## Who this is for

Zazzle **Makers** — contracted manufacturing partners. This surface is not self-service. You need a
`vendorid` and a shared secret issued through Zazzle Maker onboarding
(maker.management@zazzle.com). Without them nothing below is callable.

## The shape of the API

One endpoint. One HTTP verb. The operation is a query parameter.

```
GET https://vendor.zazzle.com/v100/api.aspx?method=<method>&vendorid=<id>&hash=<md5>&...
```

Every reply is XML in a `<Response>` envelope:

```xml
<Response>
  <Status>
    <Code>OK|ERROR</Code>
    <Info>human-readable message</Info>
  </Status>
  <Result>…method-specific payload…</Result>
</Response>
```

### Three things to internalize before writing any client

1. **Errors come back as HTTP 200.** The real status is `Response/Status/Code`. Branching on
   `response.ok` will make every failure look like a success.
2. **`hash` is a per-call signature**, an MD5 over your vendor id, the call's business parameters and
   your shared secret. The exact concatenation is method-specific and comes with your credentials.
   A hash computed for one method or one order is not reusable for another.
3. **The validator reports one missing parameter at a time**, in a fixed order, embedded in prose.
   There is no field-level error list.

## Step 1 — poll for work (`listneworders`)

```
?method=listneworders&vendorid=<id>&hash=<md5>
```

There are no webhooks and no events on this API. Order intake is **poll-only** — you choose the
interval. There is also no pagination, no cursor and no `since` filter: the method returns the new,
unacknowledged set and you reconcile against your own store of what you have already seen.

Also poll `listcancelledorders` (same parameters) so you stop work on orders Zazzle or the buyer
pulled back. Nothing pushes a cancellation to you.

## Step 2 — read the order (`getorder`)

```
?method=getorder&vendorid=<id>&orderid=<orderid>&hash=<md5>
```

Returns the order with its shipping address, line items, print files and preview files. Persist the
print-file URLs to your own storage rather than re-fetching on demand.

## Step 3 — acknowledge (`ackorder`)

```
?method=ackorder&vendorid=<id>&orderid=<orderid>&type=<ack type>&hash=<md5>
```

This is a **state change** even though it is a `GET`. Acknowledge after you have durably stored the
order, not before.

## Step 4 — print the packing sheet (`getpackingsheet`)

```
?method=getpackingsheet&vendorid=<id>&orderid=<orderid>&page=<page>&hash=<md5>
```

`page` selects a page of the printed packing document — it is not result-set pagination.

## Step 5 — buy the label (`getshippinglabel`) — READ THIS FIRST

```
?method=getshippinglabel&vendorid=<id>&orderid=<orderid>&weight=<weight>&format=PNG&hash=<md5>
```

Returns carrier, shipping method, tracking number, weight, and the shipping document URLs (the label,
plus a commercial invoice on international shipments).

**This operation spends money.** Despite the `get` prefix and the HTTP `GET` verb, it purchases a
carrier label. It carries **no idempotency key** and Zazzle documents no deduplication behaviour for
a repeat call. Therefore:

* Never put this call behind a blind retry, a timeout retry, or an at-least-once queue.
* Record the tracking number returned before doing anything else; that is your only handle for
  undoing the purchase.
* If you are wiring this into an agent, gate it behind explicit human confirmation. A read-shaped
  name on a spending operation is exactly the trap an autonomous retry loop falls into.

## Step 6 — undo a label (`voidshippinglabel`)

```
?method=voidshippinglabel&vendorid=<id>&tracking=<tracking number>&hash=<md5>
```

Keyed by **tracking number**, not order id — which is why Step 5 says record it first.

## Methods that do not exist

Confirmed against the live endpoint on 2026-08-05: `getorders`, `listorders`, `getpackingslip`,
`setordershipped`, `getorderstatus`, `cancelorder`, `gettracking`, `getprintfiles` are **not**
implemented. An unimplemented method reports `Invalid parameter 'method'`, identical to a typo, so
guessing method names gives you no signal.

## Reference

* Confirmed method surface and required parameters: `openapi/zazzle-vendor-openapi.yml` (`x-rpc-methods`)
* Error model: `errors/zazzle-problem-types.yml`
* Auth model: `authentication/zazzle-authentication.yml`
* Maker Manual: https://makerhelp.zazzle.com/hc/en-us
* Contact: maker.management@zazzle.com
