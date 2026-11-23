# FastBite – Campus Food-Court Ordering System (Starter Code)

FastBite lets students and staff order food from a university food court. The backend is
a set of six Node.js (Express) microservices. All services store their data in MongoDB,
and every service has its **own database** on the same MongoDB server.

How to run, ship and repeat the deployment: see [`DEPLOYMENT.md`](DEPLOYMENT.md).

## Services

| Folder              | Port | Responsibility                                                    | Database              |
|---------------------|------|-------------------------------------------------------------------|-----------------------|
| `api-gateway`       | 3000 | Single entry point, forwards requests to the right service         | none                  |
| `menu-service`      | 3001 | Menu items (name, price, availability)                             | `fastbite_menu`       |
| `order-service`     | 3002 | Coordinates an order (menu, customer, inventory, loyalty points)   | `fastbite_orders`     |
| `inventory-service` | 3003 | Stock levels, refuses to oversell                                  | `fastbite_inventory`  |
| `customer-service`  | 3004 | Customer profile and loyalty points                                | `fastbite_customers`  |
| `payment-service`   | 3005 | Verifies the amount against the order and records the payment      | `fastbite_payments`   |

Each folder is an independent Node.js project (`package.json`, entry file `index.js`,
start command `npm start`).

## Configuration (environment variables)

Every service reads its settings from environment variables. Service addresses and the
MongoDB connection string have **no defaults**: a service started without them prints
`Missing required environment variable ...` and exits, so it can never fall back to
`localhost` by accident. `docker-compose.yml` sets all of them, and the database
password comes from a local `.env` file that is never committed (see `DEPLOYMENT.md`).

| Service             | Variable                | Value under docker compose                     |
|---------------------|-------------------------|------------------------------------------------|
| all services        | `PORT`                  | the service's port from the table above (also the default) |
| all except gateway  | `MONGO_URI`             | `mongodb://<user>:<password>@mongo:27017/<its database>?authSource=admin`, built from `.env` |
| menu, inventory, customer | `SEED_DATA`       | demo data is inserted when the database is empty; set `false` to disable |
| `api-gateway`       | `MENU_SERVICE_URL`      | `http://menu-service:3001`                     |
| `api-gateway`       | `ORDER_SERVICE_URL`     | `http://order-service:3002`                    |
| `api-gateway`       | `INVENTORY_SERVICE_URL` | `http://inventory-service:3003`                |
| `api-gateway`       | `CUSTOMER_SERVICE_URL`  | `http://customer-service:3004`                 |
| `api-gateway`       | `PAYMENT_SERVICE_URL`   | `http://payment-service:3005`                  |
| `order-service`     | `MENU_SERVICE_URL`      | `http://menu-service:3001`                     |
| `order-service`     | `INVENTORY_SERVICE_URL` | `http://inventory-service:3003`                |
| `order-service`     | `CUSTOMER_SERVICE_URL`  | `http://customer-service:3004`                 |
| `payment-service`   | `ORDER_SERVICE_URL`     | `http://order-service:3002`                    |

A service that cannot reach MongoDB at start-up prints an error and exits.

## Demo data (inserted automatically on first start)

* Menu / stock: `M-101` Zinger Burger Rs. 450 (stock 80), `M-102` Fries Rs. 200 (stock 120),
  `M-103` Cold Drink Rs. 100 (stock 200), `M-104` Chai Rs. 80 (stock 150)
* Customers: `C-1042` Hamza Ali (150 points), `C-1043` Ayesha Khan (20 points)
* Loyalty: 1 point for every Rs. 10 spent; a free drink is unlocked at 200 points.

## API (through the gateway, `http://localhost:3000`)

| Method | Route                       | Body / notes                                                          |
|--------|-----------------------------|-----------------------------------------------------------------------|
| GET    | `/menu`                     | available menu items                                                  |
| GET    | `/customers/:customerId`    | profile and loyalty points                                            |
| POST   | `/customers/update-points`  | `{ "customerId": "C-1042", "points": 65 }`                            |
| GET    | `/inventory`, `/inventory/:itemId` | stock levels                                                   |
| POST   | `/inventory/update`         | `{ "items": [ { "itemId": "M-101", "quantity": 1 } ] }`               |
| POST   | `/orders`                   | `{ "customerId": "C-1042", "items": [ { "itemId": "M-101", "quantity": 1 }, { "itemId": "M-102", "quantity": 1 } ] }` |
| GET    | `/orders`, `/orders/:orderId` | stored orders                                                       |
| POST   | `/payments`                 | `{ "orderId": "ORD-XXXXXXXX", "amount": 650 }`                        |
| GET    | `/payments`, `/payments/:orderId` | recorded payments                                               |

Every service also answers `GET /health`.
