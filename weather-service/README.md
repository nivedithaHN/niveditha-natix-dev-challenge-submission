# Resilient Weather Service API

- [View complete challenge](https://github.com/natix-io/dev-challenge/blob/main/weather-service/readme.md)

## Problem Overview

Design a resilient backend API to serve daily weather data for a user-specified city, despite using a rate-limited external weather API (100 calls/hour). 

Our service should:
- Minimize calls to the external weather API
- Handle API failures gracefully
- Serve ~100,000 daily active users across ~2,500 cities

## Solution Overview

### Key Strategies

1. Caching - Weather data per city is cached with a daily TTL to minimize calls to the rate-limited external API
2. Internal rate limiting - External API usage tracked using a global counter
3. Graceful Degradation - If the API fails or rate limit exceeds, return last known good data or fallback response
4. Improved Response Structure - Clear and friendly metadata for frontend rendering
5. Distributed Locking - To prevent "thundering herd" issues when multiple requests for the same uncached city arrive concurrently
6. Circuit Breaking - To gracefully handle external API failures and protect both our service and the external service from cascading issues

## Detailed Design & Answers

* [**Go to DESIGN.md for the full explanation**](DESIGN.md)
