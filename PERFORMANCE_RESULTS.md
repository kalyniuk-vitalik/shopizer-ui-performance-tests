# Performance Results - Shopizer UI E-commerce Flow

---

## Summary

| Goal | Parameter | Value |
|------|-----------|-------|
| Load Test stability | 5 users, 3600s | Stable, 0% errors |
| Transactions count | Total | 3564 |
| Avg response time | All transactions | 176 ms |
| Slowest transaction | Login | 699 ms avg |
| Fastest transaction | Open login page | 66 ms avg |
| Throughput | Overall | ~1 TPS |

---

## Load Test

**Load model:** 5 threads, ramp-up 60s, duration 3600s, loops infinite

### Transactions Response Time Aggregate

| Transaction | Throughput | Avg | Median | 90 pct | 95 pct | 99 pct | Errors |
|-------------|------------|-----|--------|--------|--------|--------|--------|
| Open login page | 0.20 req/s | 66 ms | 57 ms | 67 ms | 124 ms | 363 ms | 0% |
| Add to cart | 0.28 req/s | 70 ms | 18 ms | 179 ms | 182 ms | 244 ms | 0% |
| Update item | 0.22 req/s | 92 ms | 40 ms | 201 ms | 207 ms | 313 ms | 0% |
| Open category page | 0.30 req/s | 138 ms | 84 ms | 246 ms | 255 ms | 382 ms | 0% |
| Open product page | 0.30 req/s | 140 ms | 83 ms | 242 ms | 251 ms | 371 ms | 0% |
| Checkout | 0.23 req/s | 143 ms | 90 ms | 251 ms | 265 ms | 366 ms | 0% |
| Proceed to checkout | 0.22 req/s | 155 ms | 101 ms | 258 ms | 267 ms | 394 ms | 0% |
| Open home page | 0.23 req/s | 181 ms | 162 ms | 229 ms | 317 ms | 680 ms | 0% |
| Remove item | 0.21 req/s | 130 ms | 87 ms | 248 ms | 290 ms | 394 ms | 0% |
| Fill order fields | 0.23 req/s | 300 ms | 237 ms | 396 ms | 420 ms | 645 ms | 0% |
| Submit order | 0.23 req/s | 496 ms | 450 ms | 602 ms | 688 ms | 1230 ms | 0% |
| Login | 0.20 req/s | 699 ms | 688 ms | 733 ms | 837 ms | 1580 ms | 0% |

**Key findings:**
- System is **stable for 3600s** at 5 concurrent users with **0% errors**
- **Login (699ms avg)** is slowest, expected for session-based auth with full cookie handling
- **Submit order (496ms avg)** involves full checkout pipeline (cart, address, shipping, confirmation)
- **Page navigation** (category, product, home) stays under **200ms median**
- **90th percentile** under 733ms for all transactions, no long-tail outliers
- **No degradation** over time, response time graph shows flat consistent pattern throughout 3600s
