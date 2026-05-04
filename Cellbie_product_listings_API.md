# Cellbie Listings API Documentation

- 1 Getting Started
  - 1.1 Overview
  - 1.2 Base URL
  - 1.3 Authentication
  - 1.4 Development Environment
  - 1.5 Request Format
  - 1.6 Quick Start: Create Your First Listing
- 2 Core Concepts
  - 2.1 C-SKU (Cellbie Stock-Keeping Unit)
  - 2.2 Quality Codes
  - 2.3 Listing Statuses
  - 2.4 Listing Lifecycle
  - 2.5 Order Lifecycle
  - 2.6 Pricing
  - 2.7 Pagination
- 3 Error Handling
- 4 Rate Limits
- 5 API Reference
  - 5.1 Catalog
    - 5.1.1 `getcatalog` — Browse the Product Catalog
    - 5.1.2 `getcsku` — Look Up a C-SKU
  - 5.2 Listings
    - 5.2.1 `createlisting` — Create or Replace a Listing
    - 5.2.2 `updatelisting` — Update a Listing
    - 5.2.3 `deletelisting` — Delete a Listing
    - 5.2.4 `getlistings` — Get Listings
  - 5.3 Orders
    - 5.3.1 `getorders` — Get Orders
    - 5.3.2 `updateorder` — Fulfill & Ship an Order
- 6 Appendix
  - 6.1 Field Reference
- 7 Additional Notes

## 1 Getting Started

### 1.1 Overview

The Cellbie Listings API lets sellers manage product listings and fulfill inventory orders on the Cellbie marketplace. With this API, you can:

- Browse the product catalog and look up C-SKUs
- Create, update, and delete listings
- Retrieve and fulfill orders
- Record shipment tracking

### 1.2 Base URL

Callers should use the following URL for API access, where `{domain}` is the wbapp.ca subdomain for the customer:
```
POST https://{domain}.wbapp.ca/apiv1/{endpoint}
```

### 1.3 Authentication

All requests require a valid API token. Tokens are domain-scoped, meaning a token can only access data belonging to the domain (seller account) it was issued for. Tokens are also permission-scoped; your token must be enabled for the specific endpoint you call.

Include your token in every request body. For JSON requests, send it as a `token` property:

```json
{
  "token": "your_api_token"
}
```

For form-encoded requests, send it as a standard form field named `token`:

```bash
curl -X POST https://{domain}.wbapp.ca/apiv1/getcatalog \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "token=your_api_token&make=Apple"
```

To obtain your API token, contact your Cellbie account manager describing what you are looking to accomplish with the Product Listings API. You will then receive your API token. Permissions are granted per endpoint — your token will only have access to the specific endpoints enabled for your integration.

### 1.4 Development Environment

Contact **support@cellbie.com** to get access to a development environment and API key. The development environment includes dummy data and pricing, and will not generate real transactions.

### 1.5 Request Format

- All requests use `POST`.
- Inputs can be sent as standard form fields with `Content-Type: application/x-www-form-urlencoded` or as a JSON body with `Content-Type: application/json`.
- Field names are the same for both formats.

### 1.6 Quick Start: Create Your First Listing

**Step 1 — Browse the catalog** to find the C-SKU for the product you want to list:

```bash
curl -X POST https://{domain}.wbapp.ca/apiv1/getcatalog \
  -H "Content-Type: application/json" \
  -d '{"token": "your_api_token", "make": "Apple"}'
```

**Step 2 — Create a listing** using the C-SKU from the catalog:

```bash
curl -X POST https://{domain}.wbapp.ca/apiv1/createlisting \
  -H "Content-Type: application/json" \
  -d '{
    "token": "your_api_token",
    "csku": "C_APPLE_IPHONE15_128_B",
    "count": 25,
    "minimum_price": 430
  }'
```

**Step 3 — Check your listings:**

```bash
curl -X POST https://{domain}.wbapp.ca/apiv1/getlistings \
  -H "Content-Type: application/json" \
  -d '{"token": "your_api_token"}'
```

**Step 4 — When an order comes in**, retrieve it and fulfill:

```bash
curl -X POST https://{domain}.wbapp.ca/apiv1/getorders \
  -H "Content-Type: application/json" \
  -d '{"token": "your_api_token"}'
```

---

## 2 Core Concepts

### 2.1 C-SKU (Cellbie Stock-Keeping Unit)

Every product in Cellbie's catalog is identified by a C-SKU — a unique code derived from four attributes: **make**, **model**, **memory**, and **quality**. For example, `C_APPLE_IPHONE15_128_B` represents an Apple iPhone 15 with 128GB storage in "B" quality grade (see below).

You don't create C-SKUs — they come from Cellbie's catalog. Use `getcatalog` to browse available products or `getcsku` to look up the exact C-SKU for a known make/model/memory/quality combination.

### 2.2 Quality Codes

Each C-SKU includes a single-letter quality grade describing the device condition:

| Code | Grade         | Description                                                        |
|------|---------------|--------------------------------------------------------------------|
| `A`  | Like New      | Reserved — available to select sellers only.                       |
| `B`  | Very Good     |                                                                    |
| `C`  | Good          |                                                                    |
| `D`  | Acceptable    |                                                                    |
| `E`  | Unacceptable  |                                                                    |
| `R`  | Recycle       | Reserved — available to select sellers only.                       |

> **Note:** Like New (`A`) and Recycle (`R`) grades are reserved for very few sellers due to the specific device conditions involved.

### 2.3 Listing Statuses

| Status     | Meaning                                                                                      |
|------------|----------------------------------------------------------------------------------------------|
| `ACTIVE`   | You are currently winning the market and hold available inventory on the listing.             |
| `LOST`     | Another seller is currently winning the market with a more competitive minimum price.        |
| `INACTIVE` | Your listing has no available inventory, or your minimum price is significantly higher than current buyer bids. |

### 2.4 Listing Lifecycle

A listing becomes `ACTIVE` when its available quantity is greater than 0 and its minimum price is within acceptable range of current buyer bids. If another seller offers a more competitive minimum price, the listing status changes to `LOST`. A listing becomes `INACTIVE` when its quantity drops to 0 or its minimum price is too high compared to current buyer pricing.

```
Created → ACTIVE (quantity > 0, price in range)
           ↕
         LOST (another seller has a lower minimum price)
           ↕
         ACTIVE (you lower your price or competitor leaves)
           ↓
       INACTIVE (quantity = 0 or price too high)
```

### 2.5 Order Lifecycle

When buyers meet your price expectations on one or several listings, Cellbie generates purchase orders. Orders are created frequently — approximately every hour. Every night, sellers receive a notification with the orders created and pending fulfillment.

Once an order comes in, the seller fulfillment flow is:

1. **Retrieve orders** — Use `getorders` with `fulfilled=false` to see pending fulfillment.
2. **Fulfill** — Use `updateorder` to assign an IMEI to each order row for the corresponding C-SKU.
3. **Ship** — Use `updateorder` with `tracking` and `courier` to record shipment information per buyer.

### 2.6 Pricing

All prices are *whole-dollar integers*. For example, both `430` and `430.99` represent `$430`. There are no decimal/cent values in any price field.

### 2.7 Pagination

All collection endpoints (`getcatalog`, `getlistings`, `getorders`) support pagination via `limit` and `offset`:

| Parameter | Type    | Description                                      |
|-----------|---------|--------------------------------------------------|
| `limit`   | integer | Maximum number of rows to return.                |
| `offset`  | integer | Number of rows to skip before returning results. |

The response includes a `total` field with the count of all matching rows before pagination is applied, so you can calculate the number of pages. If `limit` is omitted or set to `0`, all matching rows are returned. There is currently no endpoint-level maximum `limit`; negative `limit` or `offset` values return an error.

---

## 3 Error Handling

Failed requests return a non-200 HTTP status code and a JSON object with an `error` field:

```json
{
  "error": "Bad Request - Invalid IMEI."
}
```

Common status codes:

| Status | Meaning |
|--------|---------|
| `400` | Bad request, such as missing required fields, invalid C-SKU, invalid pagination values, invalid IMEI, invalid courier, order not found, or listing not found. |
| `401` | Missing, invalid, or incorrect token type. |
| `403` | Token domain does not match the request domain. |
| `405` | Endpoint does not exist or the token is not enabled for that endpoint. |
| `429` | Token rate limit exceeded. |
| `500` | Unexpected server error. |

Error messages are intended for troubleshooting and may include endpoint-specific details, for example `Bad Request - Listing not found.`, `Bad Request - Inventory line not found.`, or `Bad Request - IMEI is required when supplying tracking information.`

---

## 4 Rate Limits

Rate limits are enforced per token over a rolling 10-second window. The exact limit may vary by integration and is assigned with the API token. If the limit is exceeded, the API returns HTTP `429` with an error message.

---

## 5 API Reference

### 5.1 Catalog

#### 5.1.1 `getcatalog` — Browse the Product Catalog

Returns the available catalog of C-SKUs for your domain.

**Input**

| Field    | Type    | Required | Description                                      |
|----------|---------|----------|--------------------------------------------------|
| `token`  | string  | Yes      | API token for the calling domain.                |
| `csku`   | string  | No       | Return only this exact C-SKU.                    |
| `make`   | string  | No       | Filter by make (case-insensitive).               |
| `limit`  | integer | No       | Maximum rows to return.                          |
| `offset` | integer | No       | Rows to skip before returning results.           |

**Response**

| Field     | Type    | Description                                           |
|-----------|---------|-------------------------------------------------------|
| `ok`      | boolean | `true` on success.                                    |
| `catalog` | array   | Array of catalog row objects.                         |
| `total`   | integer | Total matching rows before pagination.                |

**Catalog row fields**

| Field          | Type    | Description                        |
|----------------|---------|------------------------------------|
| `csku`         | string  | Cellbie stock-keeping unit.        |
| `make`         | string  | Manufacturer code.                 |
| `title`        | string  | Catalog title for the device family. |
| `model`        | string  | Machine-oriented model identifier. |
| `memory`       | string  | Device memory value.               |
| `quality`      | string  | Single-letter quality code.        |
| `market_price` | integer | Current market price (whole dollars). |

**Example Request**

```json
{
  "token": "your_api_token",
  "make": "Apple",
  "limit": 2
}
```

**Example Response**

```json
{
  "ok": true,
  "total": 142,
  "catalog": [
    {
      "csku": "C_APPLE_IPHONE15_128_A",
      "make": "Apple",
      "title": "iPhone 15",
      "model": "iPhone 15",
      "memory": "128",
      "quality": "A",
      "market_price": 520
    },
    {
      "csku": "C_APPLE_IPHONE15_128_B",
      "make": "Apple",
      "title": "iPhone 15",
      "model": "iPhone 15",
      "memory": "128",
      "quality": "B",
      "market_price": 460
    }
  ]
}
```

**Empty result:**

```json
{
  "ok": true,
  "total": 0,
  "catalog": []
}
```

---

#### 5.1.2 `getcsku` — Look Up a C-SKU

Returns the C-SKU that corresponds to a specific make/model/memory/quality combination.

**Input**

| Field     | Type   | Required | Description                    |
|-----------|--------|----------|--------------------------------|
| `token`   | string | Yes      | API token.                     |
| `make`    | string | Yes      | Manufacturer code.             |
| `model`   | string | Yes      | Model identifier.              |
| `memory`  | string | Yes      | Device memory value.           |
| `quality` | string | Yes      | Single-letter quality code.    |

**Response**

| Field     | Type    | Description                  |
|-----------|---------|------------------------------|
| `ok`      | boolean | `true` on success.           |
| `csku`    | string  | The matching C-SKU.          |
| `make`    | string  | Echoed make.                 |
| `model`   | string  | Echoed model.                |
| `memory`  | string  | Echoed memory.               |
| `quality` | string  | Echoed quality.              |

**Example Request**

```json
{
  "token": "your_api_token",
  "make": "Apple",
  "model": "iPhone 15",
  "memory": "128",
  "quality": "B"
}
```

**Example Response**

```json
{
  "ok": true,
  "csku": "C_APPLE_IPHONE15_128_B",
  "make": "Apple",
  "model": "iPhone 15",
  "memory": "128",
  "quality": "B"
}
```

If required fields are missing or cannot be parsed into a C-SKU, the endpoint returns HTTP `400` with an `error` message.

---

### 5.2 Listings

#### 5.2.1 `createlisting` — Create or Replace a Listing

Creates a new seller listing for a specified C-SKU. If your domain already has a listing for that C-SKU, this endpoint replaces the listing's count, minimum price, display names, and optional `ref_id`, then returns the updated listing.

**Input**

| Field           | Type    | Required | Description                          |
|-----------------|---------|----------|--------------------------------------|
| `token`         | string  | Yes      | API token.                           |
| `csku`          | string  | Yes      | C-SKU to list.                       |
| `count`         | integer | Yes      | Quantity available.                  |
| `minimum_price` | integer | Yes      | Minimum acceptable price (whole dollars). |
| `ref_id`        | string  | No       | Your own reference ID for this listing. |

**Response**

| Field     | Type    | Description                              |
|-----------|---------|------------------------------------------|
| `ok`      | boolean | `true` on success.                       |
| `feedback`| string  | Human-readable success message.          |
| `warning` | string  | Non-fatal warning, if any.               |
| `listing` | object  | The created listing object (see below).  |

**Listing object fields**

| Field              | Type    | Description                                           |
|--------------------|---------|-------------------------------------------------------|
| `csku`             | string  | C-SKU.                                                |
| `ref_id`           | string  | Seller's reference ID.                                |
| `make`             | string  | Manufacturer code.                                    |
| `model`            | string  | Machine-oriented model identifier.                    |
| `memory`           | string  | Device memory value.                                  |
| `quality`          | string  | Single-letter quality code.                           |
| `make_name`        | string  | Human-facing make name.                               |
| `model_name`       | string  | Human-facing model name.                              |
| `units_sold`       | integer | Ordered but not yet fulfilled quantity.                |
| `count`            | integer | Listed quantity.                                      |
| `minimum_price`    | integer | Seller minimum price (whole dollars).                  |
| `market_price`     | integer | Current market price.                                 |
| `price_30day`      | integer | 30-day price reference.                               |
| `volume_estimate`  | integer | Volume estimate.                                      |
| `status`           | string  | Listing market status: `ACTIVE`, `INACTIVE`, or `LOST`. |
| `min_price_to_beat` | integer | Minimum price to beat the current market. Typically meaningful for `LOST` listings. |
| `best_min_price`   | integer | Best minimum price target.                            |

**Example Request**

```json
{
  "token": "your_api_token",
  "csku": "C_APPLE_IPHONE15_128_B",
  "count": 25,
  "minimum_price": 430,
  "ref_id": "EXT-12345"
}
```

**Example Response**

```json
{
  "ok": true,
  "feedback": "Listing created successfully.",
  "listing": {
    "csku": "C_APPLE_IPHONE15_128_B",
    "ref_id": "EXT-12345",
    "make": "Apple",
    "model": "iPhone 15",
    "memory": "128",
    "quality": "B",
    "make_name": "Apple",
    "model_name": "iPhone 15",
    "units_sold": 0,
    "count": 25,
    "minimum_price": 430,
    "market_price": 460,
    "price_30day": 455,
    "volume_estimate": 120,
    "status": "ACTIVE",
    "min_price_to_beat": 425,
    "best_min_price": 420
  }
}
```

`warning` is returned as an empty string when no warning applies. If a per-SKU quantity limit caps the submitted `count`, `warning` contains a message such as `Count was capped at the SKU limit of 10.` Existing listings are updated by this endpoint rather than rejected as duplicates.

---

#### 5.2.2 `updatelisting` — Update a Listing

Updates an existing listing. Only the fields you include will be changed (partial update).

**Input**

| Field           | Type    | Required | Description                                  |
|-----------------|---------|----------|----------------------------------------------|
| `token`         | string  | Yes      | API token.                                   |
| `csku`          | string  | Yes      | C-SKU of the listing to update.              |
| `ref_id`        | string  | No       | New seller reference ID.                     |
| `count`         | integer | No       | Updated quantity.                            |
| `minimum_price` | integer | No       | Updated minimum price (whole dollars).       |

**Response**

Same shape as `createlisting`: `ok`, `feedback`, optional `warning`, and the updated `listing` object.

**Example Request**

```json
{
  "token": "your_api_token",
  "csku": "C_APPLE_IPHONE15_128_B",
  "count": 18,
  "minimum_price": 425
}
```

**Example Response**

```json
{
  "ok": true,
  "feedback": "Listing updated successfully.",
  "listing": {
    "csku": "C_APPLE_IPHONE15_128_B",
    "ref_id": "EXT-12345",
    "make": "Apple",
    "model": "iPhone 15",
    "memory": "128",
    "quality": "B",
    "make_name": "Apple",
    "model_name": "iPhone 15",
    "units_sold": 3,
    "count": 18,
    "minimum_price": 425,
    "market_price": 460,
    "price_30day": 455,
    "volume_estimate": 120,
    "status": "ACTIVE",
    "min_price_to_beat": 425,
    "best_min_price": 420
  }
}
```

If `csku` does not match an existing listing for the calling domain, the endpoint returns HTTP `400` with `Bad Request - Listing not found.`

---

#### 5.2.3 `deletelisting` — Delete a Listing

Deletes a listing by C-SKU.

**Input**

| Field   | Type   | Required | Description                     |
|---------|--------|----------|---------------------------------|
| `token` | string | Yes      | API token.                      |
| `csku`  | string | Yes      | C-SKU of the listing to delete. |

**Response**

| Field      | Type    | Description                      |
|------------|---------|----------------------------------|
| `ok`       | boolean | `true` on success.               |
| `feedback` | string  | Human-readable success message.  |
| `csku`     | string  | The deleted C-SKU.               |

**Example Request**

```json
{
  "token": "your_api_token",
  "csku": "C_APPLE_IPHONE15_128_B"
}
```

**Example Response**

```json
{
  "ok": true,
  "feedback": "Listing deleted successfully.",
  "csku": "C_APPLE_IPHONE15_128_B"
}
```

Deleting a listing removes the seller listing and recalculates marketplace status for other sellers of the same C-SKU. Existing inventory orders are not deleted by this endpoint.

---

#### 5.2.4 `getlistings` — Get Listings

Returns your listings as a collection.

**Input**

| Field    | Type    | Required | Description                                      |
|----------|---------|----------|--------------------------------------------------|
| `token`  | string  | Yes      | API token.                                       |
| `csku`   | string  | No       | Return only this C-SKU's listing.                |
| `make`   | string  | No       | Filter by make (case-insensitive).               |
| `status` | string  | No       | Filter by listing status: `ACTIVE`, `INACTIVE`, or `LOST`. Input is case-insensitive. |
| `limit`  | integer | No       | Maximum rows to return.                          |
| `offset` | integer | No       | Rows to skip before returning results.           |

**Response**

| Field      | Type    | Description                                |
|------------|---------|--------------------------------------------|
| `ok`       | boolean | `true` on success.                         |
| `listings` | array   | Array of listing objects (same fields as `createlisting` response). |
| `total`    | integer | Total matching rows before pagination.     |

**Example Response**

```json
{
  "ok": true,
  "total": 12,
  "listings": [
    {
      "csku": "C_APPLE_IPHONE15_128_B",
      "ref_id": "EXT-12345",
      "make": "Apple",
      "model": "iPhone 15",
      "memory": "128",
      "quality": "B",
      "make_name": "Apple",
      "model_name": "iPhone 15",
      "units_sold": 3,
      "count": 18,
      "minimum_price": 425,
      "market_price": 460,
      "price_30day": 455,
      "volume_estimate": 120,
      "status": "ACTIVE",
      "min_price_to_beat": 425,
      "best_min_price": 420
    }
  ]
}
```

---

### 5.3 Orders

#### 5.3.1 `getorders` — Get Orders

Returns inventory-order rows for your domain.

**Input**

| Field          | Type    | Required | Description                                           |
|----------------|---------|----------|-------------------------------------------------------|
| `token`        | string  | Yes      | API token.                                            |
| `order_number` | string  | No       | Filter by order number. `order` is accepted as alias. |
| `csku`         | string  | No       | Filter by C-SKU.                                      |
| `buyer`        | string  | No       | Filter by buyer name/organization.                    |
| `ref_id`       | string  | No       | Filter by seller reference ID.                        |
| `imei`         | string  | No       | Filter by IMEI.                                       |
| `fulfilled`    | boolean | No       | `true` = only fully fulfilled rows. `false` = only unfulfilled. Omit for both. |
| `limit`        | integer | No       | Maximum rows to return.                               |
| `offset`       | integer | No       | Rows to skip.                                         |

**Response**

| Field    | Type    | Description                            |
|----------|---------|----------------------------------------|
| `ok`     | boolean | `true` on success.                     |
| `orders` | array   | Array of order row objects.            |
| `total`  | integer | Total matching rows before pagination. |

**Order row fields**

| Field           | Type    | Description                              |
|-----------------|---------|------------------------------------------|
| `order_number`  | string  | Inventory order identifier.              |
| `csku`          | string  | C-SKU.                                   |
| `make`          | string  | Manufacturer code.                       |
| `model`         | string  | Machine-oriented model identifier.       |
| `model_display` | string  | Human-facing model name.                 |
| `memory`        | string  | Device memory value.                     |
| `quality`       | string  | Single-letter quality code.              |
| `ref_id`        | string  | Seller reference ID.                     |
| `price`         | integer | Seller-side order row price.             |
| `imei`          | string  | Device IMEI (if fulfilled).              |
| `buyer`         | string  | Buyer name/organization.                 |
| `address`       | string  | Buyer shipping address.                  |
| `processed`     | string  | Created/processed timestamp display.     |
| `sold`          | string  | Sold timestamp display.                  |
| `ship_by`       | string  | Ship-by date display.                    |
| `fulfilled`     | boolean | Whether this order line is fully fulfilled. |

Timestamp fields are display strings, not guaranteed ISO 8601 values. `processed` is returned in `YYYY-MM-DD HH:MM GMT` format when a time is available; `sold` and `ship_by` are returned as date strings generated by the inventory shipping flow.

**Example Request**

```json
{
  "token": "your_api_token",
  "fulfilled": false,
  "limit": 10
}
```

**Example Response**

```json
{
  "ok": true,
  "total": 3,
  "orders": [
    {
      "order_number": "SO-10045",
      "csku": "C_APPLE_IPHONE15_128_B",
      "make": "Apple",
      "model": "iPhone 15",
      "model_display": "iPhone 15",
      "memory": "128",
      "quality": "B",
      "ref_id": "EXT-12345",
      "price": 450,
      "imei": "",
      "buyer": "PhoneHub Inc.",
      "address": "123 Main St, Suite 400, New York, NY 10001",
      "processed": "2025-01-10 09:00 GMT",
      "sold": "2025-01-10",
      "ship_by": "2025-01-13",
      "fulfilled": false
    }
  ]
}
```

When no IMEI is present, `imei` is returned as an empty string. `address` is returned as a single display string.

---

#### 5.3.2 `updateorder` — Fulfill & Ship an Order

Fulfills an order line by assigning an IMEI and optionally recording shipment tracking. This endpoint identifies the target order by `csku` + `order_number` — these are not mutable fields.

**Input**

| Field          | Type   | Required | Description                                          |
|----------------|--------|----------|------------------------------------------------------|
| `token`        | string | Yes      | API token.                                           |
| `csku`         | string | Yes      | C-SKU (identifies the order, not mutable).           |
| `order_number` | string | Yes      | Order number (identifies the order, not mutable).    |
| `ref_id`       | string | No       | New seller reference ID to save.                     |
| `imei`         | string | No       | IMEI to fulfill with. `IMEI` (uppercase) also accepted. |
| `tracking`     | string | No       | Tracking number. Requires `imei` to be set.          |
| `courier`      | string | No       | Courier slug (case-insensitive on input).            |

**Fulfillment behavior:**

- If `imei` is supplied and already fulfilled on a row in the order, that existing row is used.
- If `imei` is supplied and is new, the API fulfills one outstanding (unfulfilled) row in the order.
- If no `imei` is supplied, the API updates one outstanding row.
- For phone devices, IMEI validation and GSMA checks are applied.
- Shipping (via `tracking`) requires an IMEI on the same update.

Validation failures return HTTP `400` with an `error` message. Common messages include `Bad Request - Invalid IMEI.`, `Bad Request - Invalid serial number format.`, `Bad Request - Inventory line has no outstanding quantity.`, and GSMA-specific failure messages when a GSMA check fails.

**Accepted courier values:**

`ups`, `fedex`, `usps`, `dhl`, `canada-post`, `purolator`, `ups-freight`, `ups-mi`, `fedex-freight`, `fedex-crossborder`, `fedex-fims`, `dhl-sftp`, `Unknown`

Courier slugs follow the [AfterShip](https://www.aftership.com/) convention (lowercase). `Unknown` and `unknown` are both accepted and saved as `Unknown`.

**Response**

| Field             | Type    | Description                                         |
|-------------------|---------|-----------------------------------------------------|
| `ok`              | boolean | `true` on success.                                  |
| `order`           | object  | The updated order row.                              |

**Updated order row fields**

| Field             | Type    | Description                                         |
|-------------------|---------|-----------------------------------------------------|
| `csku`            | string  | C-SKU.                                              |
| `ref_id`          | string  | Seller reference ID.                                |
| `order_number`    | string  | Order number.                                       |
| `imei`            | string  | Assigned IMEI.                                      |
| `fulfilled`       | boolean | `true` when this call applied or recognized an IMEI fulfillment action. A ref/tracking-only update can return `false`. |
| `already_applied` | boolean | `true` if the IMEI was already on this order row.   |
| `shipped`         | boolean | `true` if shipment data was successfully recorded.  |
| `courier`         | string  | Courier slug (normalized).                          |
| `tracking`        | string  | Tracking number.                                    |

**Example Request — Fulfill and ship in one call:**

```json
{
  "token": "your_api_token",
  "csku": "C_APPLE_IPHONE15_128_B",
  "order_number": "SO-10045",
  "imei": "123456789012345",
  "tracking": "1Z999AA10123456784",
  "courier": "UPS"
}
```

**Example Response:**

```json
{
  "ok": true,
  "order": {
    "csku": "C_APPLE_IPHONE15_128_B",
    "ref_id": "EXT-12345",
    "order_number": "SO-10045",
    "imei": "123456789012345",
    "fulfilled": true,
    "already_applied": false,
    "shipped": true,
    "courier": "ups",
    "tracking": "1Z999AA10123456784"
  }
}
```

Note that `courier` is normalized to lowercase (`"UPS"` → `"ups"`), except for `"Unknown"` which retains its capitalization.

---

## 6 Appendix

### 6.1 Field Reference

A consolidated reference of all field names used across endpoints.

| Field              | Type    | Description                                                    |
|--------------------|---------|----------------------------------------------------------------|
| `token`            | string  | API token for the calling domain.                              |
| `csku`             | string  | Cellbie stock-keeping unit identifier.                         |
| `make`             | string  | Manufacturer code.                                             |
| `title`            | string  | Catalog title for a device family.                             |
| `model`            | string  | Machine-oriented model identifier.                             |
| `model_display`    | string  | Human-facing model display name (orders only).                 |
| `memory`           | string  | Device memory value.                                           |
| `quality`          | string  | Single-letter Cellbie quality code.                            |
| `count`            | integer | Seller listing quantity.                                       |
| `minimum_price`    | integer | Seller minimum listing price (whole dollars).                  |
| `market_price`     | integer | Current market price.                                          |
| `price_30day`      | integer | 30-day price reference.                                        |
| `volume_estimate`  | integer | Volume estimate for a listing row.                             |
| `best_min_price`   | integer | Best minimum price target.                                     |
| `min_price_to_beat`| integer | Current minimum price to beat the market.                      |
| `ref_id`           | string  | Seller reference ID.                                           |
| `make_name`        | string  | Human-facing make name (listings only).                        |
| `model_name`       | string  | Human-facing model name (listings only).                       |
| `units_sold`       | integer | Ordered but not yet fulfilled quantity (listings only).        |
| `status`           | string  | Listing market status: `ACTIVE`, `INACTIVE`, or `LOST`.        |
| `order_number`     | string  | Inventory order identifier.                                    |
| `buyer`            | string  | Buyer organization/name.                                       |
| `imei`             | string  | Device identifier for fulfillment.                             |
| `fulfilled`        | boolean | For `getorders`, whether the order line is fully fulfilled. For `updateorder`, whether this call applied or recognized an IMEI fulfillment action. |
| `price`            | integer | Seller-side order row price (whole dollars).                   |
| `address`          | string  | Buyer shipping address.                                        |
| `processed`        | string  | Created/processed timestamp display, usually `YYYY-MM-DD HH:MM GMT` when available. |
| `sold`             | string  | Sold timestamp display.                                        |
| `ship_by`          | string  | Ship-by date display.                                          |
| `tracking`         | string  | Shipment tracking number.                                      |
| `courier`          | string  | Courier slug (AfterShip convention).                           |
| `ok`               | boolean | Success indicator on all responses.                            |
| `feedback`         | string  | Human-readable success message (mutation endpoints).           |
| `warning`          | string  | Non-fatal warning message.                                     |
| `already_applied`  | boolean | `true` when IMEI was already on the target order row.          |
| `shipped`          | boolean | `true` when shipment data was recorded.                        |
| `total`            | integer | Total row count before pagination (collection endpoints).      |
| `limit`            | integer | Maximum rows to return.                                        |
| `offset`           | integer | Rows to skip before returning results.                         |

---

## 7 Additional Notes

The listings API does not expose cancellation, return, or dispute-management endpoints. Please contact your account manager or support@cellbie.com for business continuity.
