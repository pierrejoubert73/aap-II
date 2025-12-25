# EVE Online Market API Guide

This document explains how to use the EVE Online ESI (EVE Swagger Interface) API to find past and current market rates for items, such as PLEX.

## Introduction

The EVE Online API, known as ESI (EVE Swagger Interface), provides a vast amount of data about the game universe. For trading and market analysis, three main steps are usually required:
1.  **Find the Item ID** (Type ID) and **Region ID** (where you want to check prices).
2.  **Get Market History** (for past rates/averages).
3.  **Get Current Market Orders** (for the current "live" order book).

All the endpoints mentioned here are **public** and do not require authentication (OAuth) to access.

## Prerequisites

You can access the API using any HTTP client. Examples below will use:
*   **curl**: A command-line tool for transferring data.
*   **Python**: Using the `requests` library.

## Step 1: Find IDs (Item and Region)

Before fetching market data, you need the numerical IDs for the item (e.g., "PLEX") and the region (e.g., "The Forge", where the main trade hub Jita is located).

**Endpoint:** `POST /universe/ids/`

### Example: Finding IDs for "PLEX" and "The Forge"

**curl:**
```bash
curl -X POST "https://esi.evetech.net/latest/universe/ids/?datasource=tranquility" \
     -H "Content-Type: application/json" \
     -d '["The Forge", "PLEX"]'
```

**Python:**
```python
import requests

url = "https://esi.evetech.net/latest/universe/ids/"
params = {"datasource": "tranquility"}
headers = {"Content-Type": "application/json"}
data = ["The Forge", "PLEX"]

response = requests.post(url, params=params, json=data, headers=headers)
print(response.json())
```

**Response Output (Example):**
```json
{
  "inventory_types": [
    {
      "id": 44992,
      "name": "PLEX"
    }
  ],
  "regions": [
    {
      "id": 10000002,
      "name": "The Forge"
    }
  ]
}
```
*Note: The response might include other categories (like characters or alliances) if names match. Look for `inventory_types` for items and `regions` for regions.*

*   **PLEX Type ID:** `44992`
*   **The Forge Region ID:** `10000002`

## Step 2: Get Market History (Past Rates)

To see past performance, daily averages, and volume, use the history endpoint.

**Endpoint:** `GET /markets/{region_id}/history/`

### Example: PLEX History in The Forge

**curl:**
```bash
curl -s "https://esi.evetech.net/latest/markets/10000002/history/?datasource=tranquility&type_id=44992"
```

**Python:**
```python
import requests

region_id = 10000002 # The Forge
type_id = 44992      # PLEX
url = f"https://esi.evetech.net/latest/markets/{region_id}/history/"
params = {
    "datasource": "tranquility",
    "type_id": type_id
}

response = requests.get(url, params=params)
data = response.json()

# Print the most recent entry
print(data[-1])
```

**Response Structure (List of objects):**
```json
[
  {
    "average": 5100000.5,
    "date": "2023-10-25",
    "highest": 5150000.0,
    "lowest": 5050000.0,
    "order_count": 1500,
    "volume": 2000000000
  },
  ...
]
```
*   **average**: The average price for the day.
*   **highest/lowest**: The price range for the day.
*   **volume**: The quantity of items traded.

## Step 3: Get Current Market Orders (Current Rates)

To see the current "Sell" (Ask) and "Buy" (Bid) orders.

**Endpoint:** `GET /markets/{region_id}/orders/`

You usually want to filter by `type_id` and order type (`buy` or `sell`). Note that the API returns *all* active orders for the region, so filtering is essential. However, the ESI endpoint `orders` *requires* you to handle pagination or it returns the first page of *all* orders in the region if `type_id` isn't supported as a direct filter on *some* old endpoints, but generally, for specific items, you might need to fetch orders and filter client-side or check for specific `type_id` support in the query parameters.

*Correction:* The standard `/markets/{region_id}/orders/` endpoint accepts a `type_id` parameter to filter results server-side.

### Example: Current PLEX Orders in The Forge

**curl:**
```bash
curl -s "https://esi.evetech.net/latest/markets/10000002/orders/?datasource=tranquility&order_type=all&type_id=44992"
```

**Python:**
```python
import requests

region_id = 10000002 # The Forge
type_id = 44992      # PLEX
url = f"https://esi.evetech.net/latest/markets/{region_id}/orders/"
params = {
    "datasource": "tranquility",
    "order_type": "all", # 'buy', 'sell', or 'all'
    "type_id": type_id
}

response = requests.get(url, params=params)
orders = response.json()

# Separate buy and sell orders
buy_orders = [o for o in orders if o['is_buy_order']]
sell_orders = [o for o in orders if not o['is_buy_order']]

# Sort to find best prices
# Best Sell (lowest price)
sell_orders.sort(key=lambda x: x['price'])
# Best Buy (highest price)
buy_orders.sort(key=lambda x: x['price'], reverse=True)

if sell_orders:
    print(f"Lowest Sell Price: {sell_orders[0]['price']} ISK")
if buy_orders:
    print(f"Highest Buy Price: {buy_orders[0]['price']} ISK")
```

**Response Object Key Fields:**
*   `price`: The cost per unit in ISK.
*   `volume_remain`: How many items are left in this order.
*   `is_buy_order`: `true` if it's a Buy order, `false` if it's a Sell order.
*   `location_id`: The station or structure where the order is located (Jita 4-4 is `60003760`).

## Summary

1.  Use `/universe/ids/` to map "PLEX" -> `44992` and "The Forge" -> `10000002`.
2.  Use `/markets/10000002/history/?type_id=44992` to see price trends over the last year.
3.  Use `/markets/10000002/orders/?type_id=44992` to see the current live market board.
