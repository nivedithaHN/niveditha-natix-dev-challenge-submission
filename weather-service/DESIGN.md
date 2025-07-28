# Resilient Weather Service API - Technical Design Document

This document provides a detailed technical design for the Resilient Weather Service API, outlining the architecture, key components, and strategies employed to meet the challenge requirements.

## 1. Context and Scope

The goal is to design a resilient backend API that efficiently serves daily weather data to a frontend, minimizing calls to a severely rate-limited external weather API while gracefully handling its failures.

**Key Constraints:**

* **External API Limit:** 100 requests per hour (2400 requests per 24 hours)
* **External API Response:** Detailed 24-hour weather data for the *current day* for a given city.
* **User Load:** ~100000 DAU across ~2500 cities.

User authentication and external API authentication are explicitly out of scope for this design.



## 2. Assumptions Made

* The most critical assumption is how to handle the discrepancy between supporting ~2500 cities and 2400 external API requests per day. If all cities are requested equally, around 100 cities will consistently serve stale or unavailable data due to the rate limit. The system prioritizes cities based on the order in which they are requested after their cache expires.

* Basic city name normalization (e.g., converting to lowercase, trimming whitespace) is performed to ensure consistent cache keys. However, complex normalization (e.g., mapping aliases like "NYC" to "New York City"), correction of misspellings, or geolocation-based resolution of city names is considered out of scope for this design

* A distributed caching system is assumed to be available, along with support for distributed locking and atomic counters to manage cache writes and enforce rate limits.

* The external API is assumed to return weather data that is valid for the entire UTC day (00:00 to 23:59). This allows for a fixed TTL and simplifies cache invalidation. If the API returned data based on each city’s local time, the cache logic would be significantly more complex.

* The external API is assumed to consistently return a complete 24 hour forecast, with hourly entries from 0 to 23. If this structure changes or becomes inconsistent, additional validation and normalization would be required.

## 3. Design Principles

1. Aggressive Caching: We'll prioritize serving data from a fast, in-memory cache to drastically reduce the number of calls to the external API. Since the external API provides "today's" weather, we can cache this data for nearly 24 hours. This strategy is essential for handling a high volume of user requests with a very limited external API budget.

2. Internal Rate Limiting: We'll implement a robust, global rate-limiting mechanism within our system to ensure we never exceed the external API's call limit. This acts as a strict gatekeeper, preventing us from overwhelming the third-party service and incurring penalties.

3. Graceful Degradation: When fresh data isn't available (due to external API failures, rate limiting, or an open circuit breaker), our system will prioritize providing some information over none. We'll serve stale cached data as a fallback whenever possible. If no data is available, we'll return a clear, user-friendly error message, ensuring the frontend can adapt appropriately.

4. Improved Response Structure:  Every response will include metadata such as data's source (e.g., "live," "cached," "stale") and its last_updated_utc timestamp. This transparency allows the frontend to provide more informed data to users. 

5. Distributed Locking: To prevent redundant external API calls for the same uncached city, we'll use distributed locks. When a request for data not in the cache comes in, only one instance of our service will attempt to fetch it, while other concurrent requests for the same city will wait for the result or serve stale data. This avoids a "thundering herd" problem.

6. Circuit Breaking: To handle external API failures gracefully, we'll implement a circuit breaker pattern. If the external API becomes unresponsive or starts returning too many errors, our circuit breaker will "open," immediately preventing further calls for a set period. This protects both our service from long timeouts and the external service from being hammered while it's struggling.



## 4. High-Level Architecture

The system will be composed of several interconnected components working in concert to handle high user load and the constraints of the external weather API.

```
User Frontend (Web/Mobile)
      ↓
Load Balancer / API Gateway
      ↓
Our Backend API Service (Multiple Instances)
      ↓
Distributed Cache (e.g., Redis)
      ↑
      ↕ (Cache Miss, Cache Write, Distributed Lock Management)
      ↓
Global Rate Limiter (e.g., Redis-backed)
      ↓
Circuit Breaker (for External API calls)
      ↓
External Weather API
```

### 4.1. Component Responsibilities and Flow

This section briefly describes the role of each architectural component and how a request generally flows through the system.

1.  **User Frontend (Web/Mobile):** The client application that users interact with. It sends requests to our backend API and consumes the structured JSON responses.

2.  **Load Balancer / API Gateway:** The entry point for all incoming requests. It distributes traffic across multiple instances of our Backend API Service, ensuring high availability and scalability.

3.  **Our Backend API Service (Multiple Instances):** The core application logic. It processes requests, orchestrates interactions with the cache, rate limiter, circuit breaker, and the external weather API. Running multiple instances provides resilience and handles concurrent user load.

4.  **Distributed Cache (e.g., Redis):** A high-speed, shared data store for caching weather data across all service instances. It's critical for minimizing external API calls and also supports distributed locking.

5.  **Global Rate Limiter (e.g., Redis-backed):** A centralized component that strictly enforces the external API's hourly call limit across all our backend service instances.

6.  **Circuit Breaker (for External API calls):** Protects our service from prolonged outages or unresponsiveness of the external weather API by preventing further calls when a failure threshold is met.

7.  **External Weather API:** The third-party service providing the raw daily weather data, which is subject to strict rate limits.

### 4.2. Request Flow Highlights

The typical flow for a weather data request is as follows:

1.  A user's frontend sends a request for weather data for a `city` to our **Load Balancer**.
2.  The Load Balancer routes the request to an available instance of our **Backend API Service**.
3.  The Backend API Service first checks the **Distributed Cache** for the requested city's weather data.
4.  **If cached data is found and valid:** The data is immediately returned to the frontend.
5.  **If cached data is missing or expired:**
    * The Backend API Service attempts to acquire a **Distributed Lock** for that city.
    * It then consults the **Global Rate Limiter** and checks the **Circuit Breaker**.
    * If all checks pass, the Backend API Service makes a request to the **External Weather API** (with retries on transient failures).
    * The fetched data is processed, stored in the **Distributed Cache**, and then returned to the frontend.
    * If any check fails (rate limit, circuit open, external API failure), the system gracefully degrades by serving stale cached data or an appropriate error message.

This architecture ensures that the system is scalable, fault-tolerant, and highly efficient in its use of the limited external API resources.



## 5. Detailed Component Design & Workflow

This section elaborates on the internal logic of our backend API and the role of each major component.

### 5.1. `GET /weather?city=CityName` Endpoint Workflow

This is the primary entry point for frontend requests.

**Input:** HTTP GET request with `city` query parameter (e.g., `GET /weather?city=London`).

**Output:** JSON response containing weather data or an error message.

**Internal Workflow Steps:**


1.  **Input Validation & Normalization:**
    * **Purpose:** Ensure the input is valid and consistent for reliable processing and cache key generation.
    * **Process:**
        * Validate `city` parameter: Check if it's present and not empty.
        * Normalize the city name: Convert to lowercase, trim whitespace. (As per assumptions, no complex alias mapping or geolocation is performed here).
    * **Output:** `normalized_city_name` (string).
    * **Failure Mode:** If validation fails, return a 400 Bad Request error.

2.  **Cache Lookup:**
    * **Purpose:** Serve data quickly and minimize external API calls.
    * **Process:** Query the `Distributed Cache` using `normalized_city_name` as the key. Check if the retrieved data exists and if its stored expiration timestamp is in the future.
    * **Cache Expiration Strategy:** The cache entry's TTL is set to expire at midnight UTC of the current day + a small buffer (e.g., 5 minutes). This ensures that for any given city, we only attempt to fetch fresh data once per UTC day.
    * **Output:** `cached_data` (JSON object with `payload`, `is_valid`, `last_updated_utc`, `expiration_timestamp`) or `null`.
    * **Success Path:** If `cached_data` is found and `is_valid` is true, return `create_success_response(cached_data.payload, "cached", cached_data.last_updated_utc)` and terminate the request.

3.  **Acquire Distributed Lock (Cache Miss Path):**
    * **Purpose:** Prevent the "thundering herd" problem where multiple concurrent requests for the same uncached city hit the external API simultaneously.
    * **Process:** If `cached_data` is `null` (cache miss or expired), attempt to acquire a distributed lock for `normalized_city_name`. The lock has a short timeout (e.g., 10 seconds) to prevent deadlocks.
    * **Lock Acquired:** If successful, proceed to Step 4.
    * **Lock Not Acquired:** If another process holds the lock, wait for a short, random duration (e.g., 0.5-1 second) and then **re-check the cache (Step 2)**. This allows the other process to potentially populate the cache. If data is now in cache, serve it. If still not in cache, proceed to **Step 8 (Handle Fallback / Serve Unavailable Data)**.

4.  **Global Rate Limiter Check:**
    * **Purpose:** Strictly enforce the 100 requests per hour limit to the external API.
    * **Process:** Before making the HTTP call, check the `Global Rate Limiter`. This component (e.g., a Redis-backed counter) atomically tracks calls within the current hour.
    * **Allowed:** If `RATE_LIMITER.allow_request()` returns `TRUE`, proceed to Step 5.
    * **Denied:** If `RATE_LIMITER.allow_request()` returns `FALSE` (limit exceeded), release the distributed lock (if held) and proceed to **Step 8 (Handle Fallback / Serve Unavailable Data)**. Log a warning.

5.  **Circuit Breaker Check:**
    * **Purpose:** Protect our system and the external API from cascading failures during periods of external API unhealthiness.
    * **Process:** Check the state of the `Circuit Breaker` for the external weather API.
    * **Allowed:** If `CIRCUIT_BREAKER.allow_request()` returns `TRUE` (circuit is `CLOSED` or `HALF-OPEN`), proceed to Step 6.
    * **Denied:** If `CIRCUIT_BREAKER.allow_request()` returns `FALSE` (circuit is `OPEN`), release the distributed lock (if held) and proceed to **Step 8 (Handle Fallback / Serve Unavailable Data)**. Log a warning.

6.  **Call External API (with Retries):**
    * **Purpose:** Fetch fresh weather data from the external source.
    * **Process:** Execute the actual HTTP GET request to the external weather API. Implement exponential backoff retries (e.g., 3 attempts with increasing delays) for transient errors (e.g., network timeouts, 500, 502, 503 HTTP status codes, or 429 Too Many Requests).
    * **Circuit Breaker State Update:** Based on the success or failure of the external API call (after retries), update the `Circuit Breaker` state (e.g., `CIRCUIT_BREAKER.record_success()` or `CIRCUIT_BREAKER.record_failure()`).
    * **Output:** `raw_external_api_response` (JSON) on success, or throws an `Exception` on persistent failure.
    * **Success Path:** If `raw_external_api_response` is obtained successfully, proceed to Step 7.
    * **Failure Path:** If the external API call persistently fails (after retries), proceed to **Step 8 (Handle Fallback / Serve Unavailable Data)**.

7.  **Process & Cache External API Response (On Successful Fetch):**
    * **Purpose:** Transform raw data into our desired format and store it for future requests. This step is *only* reached if the external API call (Step 6) was successful.
    * **Process:**
        * Parse the `raw_external_api_response`.
        * **Data Transformation:** Convert the external API's format to our improved internal format (e.g., convert `"18°C"` string to `18.0` number, add `unit: "C"`).
        * Store this transformed, enriched data in the `Distributed Cache` with the calculated daily expiration (midnight UTC + buffer) to align with daily API refreshes, mitigate clock skew, and smooth refresh load..
        * Release the distributed lock.
        * Return the transformed data to the frontend using `create_success_response(transformed_data, "live", CURRENT_UTC_TIME())`.

8.  **Handle Fallback / Serve Unavailable Data:**
    * **Purpose:** Ensure graceful degradation and provide the best possible user experience when fresh data is unavailable due to rate limits, external API failures, or circuit breaker being open. This is the common entry point for all failure/denial scenarios.
    * **Process:**
        * Release the distributed lock (if held).
        * **Serve Stale Data:** Attempt to retrieve *any* data for `normalized_city_name` from the `Distributed Cache`, even if it's expired (`get_stale` method). If found, serve this stale data, but include `data_source: "stale"` and a warning message in the `error_info` field. This provides *some* weather information to the user.
        * **Fallback Response:** If no cached data (not even stale) is available, return a generic "Weather data temporarily unavailable" response to the frontend, populating the `error_info` field with `status: "unavailable"` and a relevant error `code` and `message`.
        * Log the specific failure reason (e.g., "External API down," "Rate limit hit," "Circuit Open").


### 5.2. Component Deep Dive

This section provides more specific details on the implementation and internal workings of each core component.

#### Distributed Cache (e.g., Redis)

* **Role:** The primary mechanism to reduce external API calls.
* **Implementation Details:**
    * **Keys:** `weather:city:<normalized_city_name>`.
    * **Values:** Stores the serialized JSON of our improved weather response object, along with metadata like `last_updated_utc` and `expiration_timestamp`.
    * **TTL:** Set to midnight UTC of the current day + 5 minutes. This ensures daily refresh attempts.
    * **Stale Data Retrieval:** Must support retrieving data even if its TTL has expired, for graceful degradation.

#### Global Rate Limiter (e.g., Redis-backed Token Bucket)

* **Role:** Strictly enforce the 100 requests per hour limit across all backend service instances.
* **Implementation Details:**
    * **Mechanism:** Uses a shared counter in Redis. When a backend instance wants to make an external call, it attempts to atomically increment a counter key (e.g., `api_calls_hourly:<current_hour_utc>`).
    * **Hourly Reset:** The counter key is set with an expiration of 1 hour (or slightly more) to automatically reset the count.
    * **Logic:** If the incremented count is within the `EXTERNAL_API_HOURLY_LIMIT`, the request is allowed; otherwise, it's denied.

#### Circuit Breaker (for External API)

* **Role:** Protects our system from a failing external API.
* **Implementation Details:**
    * **States:** `CLOSED` (normal), `OPEN` (blocking calls due to failures), `HALF-OPEN` (allowing test calls after a timeout).
    * **Failure Threshold:** A configurable number of consecutive failures (e.g., 5 failures) or a failure rate (e.g., 50% failures within a 10-second window) triggers the `OPEN` state.
    * **Reset Timeout:** A configurable duration (e.g., 60 seconds) after which the circuit transitions from `OPEN` to `HALF-OPEN`.
    * **Monitoring:** Records successes and failures to update its internal state.

#### Distributed Locking

* **Role:** Prevents redundant external API calls for the same city during cache misses.
* **Implementation Details:**
    * **Mechanism:** Uses atomic operations on the Distributed Cache (e.g., Redis's `SET key value NX PX timeout` command). `NX` ensures the key is set only if it doesn't already exist (acquiring the lock), and `PX timeout` ensures the lock automatically expires to prevent deadlocks.
    * **Release:** The lock is released after the data fetch and cache update are complete.



## 6. Moderating (Minimizing) External API Calls

This section details how the design directly addresses the goal of minimizing calls to the rate-limited external weather API:

* **Aggressive Daily Caching:** This is the most impactful strategy. By caching the 24-hour weather data for an entire UTC day for each city, we ensure that once data for a city is fetched, all subsequent requests for that city within the same day are served from the cache, consuming zero external API calls. Given 100000 DAU across 2500 cities, a high cache hit rate is essential for sustainability.
* **Global Rate Limiting:** The dedicated Rate Limiter component acts as a strict gatekeeper, ensuring that the total number of calls made to the external API across *all* our backend service instances never exceeds the 100 requests per hour limit. This prevents accidental over-usage and potential blacklisting.
* **Distributed Locking:** By preventing multiple concurrent fetches for the same city during a cache miss, distributed locking ensures that a single cache miss for a city results in only *one* external API call, not many. This avoids unnecessary redundant calls during peak demand for a newly requested or expired city.
* **Smart Cache Expiration:** Expiring cache entries precisely at midnight UTC (plus a buffer)  ensures that we only attempt to refresh data once per day per city, aligning with the external API's "today's weather" data model.



## 7. Handling API Failures Gracefully

This section explains how the system is designed to handle failures from the external weather API without completely breaking the user experience:

* **Circuit Breaker:** This is the primary mechanism for handling external API unhealthiness. When the external API starts failing frequently, the circuit breaker "opens," preventing our service from sending further requests to it. This protects the external API from being overloaded and prevents our service from blocking on slow or failing responses.
* **Retries with Exponential Backoff:** For transient network issues or temporary external API glitches (e.g., 5xx errors, 429 Too Many Requests), our system attempts to retry the external API call with increasing delays. This increases the chance of success for intermittent problems.
* **Serving Stale Data:** In scenarios where fresh data cannot be obtained (due to external API failure, rate limit, or open circuit), the system will attempt to serve the most recently available data from the cache, even if it's expired. This provides *some* weather information to the user, which is generally preferred over a blank screen or a hard error. The response clearly indicates `data_source: "stale"`.
* **Clear Error Messaging:** If no data (fresh or stale) can be provided, the API returns a clear, user-friendly error message (`status: "unavailable"`, `error_info` object) to the frontend, allowing the UI to inform the user appropriately rather than crashing.


## 8. Improved Response Object Design

The example response provided is minimal. A more effective response for a frontend/UI would include additional metadata, more detailed weather parameters, and clear status indicators to enhance the user experience and provide richer data for display.

```json
{
  "city": "New York",
  "country_code": "US", // ISO 3166-1 alpha-2 code for country
  "latitude": 40.7128,
  "longitude": -74.0060,
  "timezone_offset_seconds": -14400, // Offset from UTC in seconds (e.g., -4 hours for EDT)
  "last_updated_utc": "2025-07-27T14:30:00Z", // When this data was last successfully fetched by our service
  "data_source": "live", // "live" | "cached" | "stale" | "fallback" - Indicates data freshness/origin
  "status": "success", // "success" | "unavailable" - Overall status of the request
  "weather_forecast": [
    {
      "hour": 0, // Hour of the day (0-23)
      "time_utc": "2025-07-27T00:00:00Z", // Full UTC timestamp for precision
      "temperature": {
        "value": 18.0, 
        "unit": "C", 
        "feels_like": 17.5
      },
      "condition": "Clear", // Human-readable condition string
      "condition_code": "CLEAR_SKY_DAY", // Standardized code for UI logic/icons (e.g., for mapping to specific images/SVGs)
      "humidity_percent": 65,
      "wind_speed_kph": 15.2, 
      "wind_direction_degrees": 270, 
      "pressure_hpa": 1012.5,
      "precipitation_chance_percent": 5,
      "uv_index": 7
    },
    {
      "hour": 1,
      "time_utc": "2025-07-27T01:00:00Z",
      "temperature": {
        "value": 17.0,
        "unit": "C",
        "feels_like": 16.0
      },
      "condition": "Clear",
      "condition_code": "CLEAR_SKY_NIGHT",
      "humidity_percent": 68,
      "wind_speed_kph": 12.0,
      "wind_direction_degrees": 260,
      "pressure_hpa": 1013.0,
      "precipitation_chance_percent": 0,
      "uv_index": 0
    }
    // ... up to hour 23
  ],
  "daily_summary": { // Useful for displaying a quick overview of the day without parsing the full forecast
    "min_temperature": {"value": 15.0, "unit": "C"},
    "max_temperature": {"value": 25.0, "unit": "C"},
    "overall_condition": "Partly Cloudy",
    "overall_condition_code": "PARTLY_CLOUDY",
    "sunrise_utc": "2025-07-27T05:30:00Z",
    "sunset_utc": "2025-07-27T20:15:00Z"
  },
  "error_info": { // Only present if 'status' is not "success"
    "code": "WEATHER_UNAVAILABLE", // Machine-readable error code for frontend logic
    "message": "Weather data temporarily unavailable for this city. Please try again later.", // User-friendly message for display
    "details": "External API rate limit exceeded or external service is experiencing issues. Stale data might be served if available." // Technical details for debugging/logging
  }
}
```