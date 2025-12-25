# EVE Online Market API Guide (ESI)

This guide explains how to use the **EVE Swagger Interface (ESI)** to access EVE Online's market data. You can use this to find current prices (orders) and historical market data for items like PLEX, minerals, ships, and more.

## Prerequisites

*   **HTTP Client:** You can use `curl` for testing or libraries like `requests` in Python.
*   **Base URL:** The root URL for the API is `https://esi.evetech.net`.
*   **Documentation:** [Official ESI Documentation (Swagger UI)](https://esi.evetech.net/ui/)

---

## Step 1: Find the Item Type ID

Before you can query market data, you need the `type_id` of the item (e.g., "PLEX").

**Endpoint:** `/v2/search/`
**Method:** `GET`

### Example: Finding the ID for "PLEX"

You need to search the `inventory_type` category.

**Request (curl):**
```bash
curl -X GET "https://esi.evetech.net/latest/search/?categories=inventory_type&search=PLEX&strict=true"
```

*   `categories=inventory_type`: Tells ESI we are looking for items.
*   `search=PLEX`: The name of the item.
*   `strict=true`: Exact match only (helps avoid getting PLEX related skins etc).

**Response:**
```json
{
  "inventory_type": [
    44992
  ]
}
```
The ID for **PLEX** is `44992`.

---

## Step 2: Get Market History (Past Rates)

To analyze trends (past rates), use the history endpoint. Market data is separated by **Region**. The most popular trade hub is **Jita**, located in the **The Forge** region.

*   **The Forge Region ID:** `10000002`
*   **PLEX Type ID:** `44992`

**Endpoint:** `/v1/markets/{region_id}/history/`
**Method:** `GET`

### Example: PLEX History in The Forge

**Request (curl):**
```bash
curl -X GET "https://esi.evetech.net/latest/markets/10000002/history/?type_id=44992"
```

**Response (Truncated):**
```json
[
  {
    "average": 4950000.5,
    "date": "2023-10-01",
    "highest": 5000000.0,
    "lowest": 4900000.0,
    "order_count": 1200,
    "volume": 50000000
  },
  ...
]
```
*   `average`: The average price for that day.
*   `highest` / `lowest`: The max/min price paid.
*   `volume`: Total quantity traded.
*   `order_count`: Number of orders executed.

---

## Step 3: Get Current Market Orders (Current Rates)

To execute trades or see the current "Sell" and "Buy" walls, fetch the active orders.

**Endpoint:** `/v1/markets/{region_id}/orders/`
**Method:** `GET`

### Example: Current PLEX Orders in The Forge

**Request (curl):**
```bash
curl -X GET "https://esi.evetech.net/latest/markets/10000002/orders/?order_type=all&type_id=44992"
```

*   `order_type=all`: Get both Buy and Sell orders. (Or use `buy` / `sell`).

**Response (Truncated):**
```json
[
  {
    "duration": 90,
    "is_buy_order": false,
    "issued": "2023-10-25T12:00:00Z",
    "location_id": 60003760,
    "min_volume": 1,
    "order_id": 6539485734,
    "price": 5100000.0,
    "range": "region",
    "system_id": 30000142,
    "type_id": 44992,
    "volume_remain": 100,
    "volume_total": 100
  },
  ...
]
```
*   `is_buy_order`: `true` if it's a Buy order (bid), `false` if it's a Sell order (ask).
*   `price`: The cost per unit.
*   `location_id`: `60003760` is Jita IV - Moon 4 - Caldari Navy Assembly Plant (the main hub).

---

## Python Script Example

Here is a simple Python script to fetch the current average price of PLEX in Jita.

```python
import requests

def get_plex_price():
    # 1. Constants
    REGION_THE_FORGE = 10000002
    TYPE_PLEX = 44992

    # 2. Get Market History (Last entry is usually yesterday's closed data)
    url = f"https://esi.evetech.net/latest/markets/{REGION_THE_FORGE}/history/"
    params = {"type_id": TYPE_PLEX}

    try:
        response = requests.get(url, params=params)
        response.raise_for_status()
        data = response.json()

        # Get the most recent day
        latest = data[-1]
        print(f"Date: {latest['date']}")
        print(f"Average Price: {latest['average']:,.2f} ISK")
        print(f"Volume Traded: {latest['volume']:,}")

    except Exception as e:
        print(f"Error fetching data: {e}")

if __name__ == "__main__":
    get_plex_price()
```

## Useful IDs

| Name | ID |
| :--- | :--- |
| **Regions** | |
| The Forge (Jita) | `10000002` |
| Domain (Amarr) | `10000043` |
| **Items** | |
| PLEX | `44992` |
| Tritanium | `34` |
| Veldspar | `1230` |

## Rate Limiting and Etiquette

The EVE Online API (ESI) has a rate limit.
*   **Error Limit:** If you generate too many errors (HTTP 4xx/5xx), you will be temporarily banned.
*   **Cache:** Respect the `Expires` header. Data like market history updates only once a day. Market orders update every 5 minutes. Do not spam requests for data that hasn't changed.
