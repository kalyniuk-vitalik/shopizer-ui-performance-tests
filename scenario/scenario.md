# UI E-commerce Flow — Test Scenario

Full end-to-end user journey through the Shopizer e-commerce application: from opening the home page, through login, browsing categories, adding products to cart, manipulating cart items, and completing checkout with order submission.

The scenario covers 7 transactions: **Open home page → Login → Open category page → Open product page → Add to cart → Checkout → Submit order**.

Login runs on a configurable percentage of iterations (default 30%) via a Throughput Controller, the rest of the sessions stay anonymous, which reflects realistic traffic distribution. Adding products to cart is driven by a While loop that runs until the cart reaches a configurable target quantity (`PRODUCTS_TO_ADD_COUNT`), with a stock check before each add to skip out-of-stock items. Before submitting the order, a Weighted Switch Controller randomly either removes an item or updates its quantity (configurable weights, default 25/75), adding variance to the cart state before checkout.

User credentials are generated in a setUp Thread Group before the main test starts and written to `users_creds.csv` via Groovy + JMeter's `FileServer` API, making the path portable across local and CI environments. The CSV is recycled during the run, so any number of virtual users can share a fixed credential pool.

All test parameters (thread count, ramp-up, duration, loop count, products to add, whether to regenerate users) are externalized via `-J` properties, making the plan fully controllable from CI without touching the JMX.

Metrics are streamed in real time to InfluxDB via a Backend Listener with event tags (`USERS`, `RAMPUP`, `DURATION`), which allows Grafana to annotate test boundaries and compare runs across different load profiles in the Comparison dashboard.

## Screenshots

[Full test plan tree with all transactions and controllers](screenshots/jmx-full-tree.png)

[setUp Thread Group: conditional user registration, CSV generation via Groovy](screenshots/jmx-setup-thread-group.png)
