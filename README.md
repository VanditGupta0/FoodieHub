# FoodieHub
 
**A food delivery platform where the database does the heavy lifting.** FoodieHub has a React front end and an Express.js API, but most of the business rules run inside PostgreSQL itself: triggers keep driver status, order status and order totals in sync, and indexes serve the order-tracking lookups.
 
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-PL%2FpgSQL-336791?logo=postgresql&logoColor=white)
![Express](https://img.shields.io/badge/Express.js-4-000000?logo=express&logoColor=white)
![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)
![Vite](https://img.shields.io/badge/Vite-5-646CFF?logo=vite&logoColor=white)
 
<!-- Add screenshots here, e.g.
![Ordering](docs/screenshots/order.png)
![Admin analytics dashboard](docs/screenshots/dashboard.png)
![Delivery tracking](docs/screenshots/tracking.png)
-->
 
---
 
## Features
 
**Customers**
- Browse restaurants and menus, with search and category filters
- Place an order with several items and a payment mode (cash, card, UPI, wallet)
- See their saved delivery addresses and their own orders
- Track their orders in a delivery view that shows restaurant and customer coordinates, the driver and delivery progress
**Admins**
- Analytics dashboard: total revenue, orders, customers, drivers, average rating, order-status breakdown, top 5 restaurants by revenue and top 5 items sold
- Delivery view across all orders
- Read-only views of customers, drivers, admins, ratings, payments, categories and addresses
The app shows different tabs depending on whether you sign in as a customer or an admin.
 
---
 
## Tech stack
 
| Layer | Technology |
|---|---|
| Database | PostgreSQL, PL/pgSQL (triggers, functions, a view) |
| API | Node.js, Express.js, `pg` (node-postgres) |
| Front end | React 18, Vite |
 
---
 
## Database design
 
**15 tables, 6 ENUM types, 9 indexes, 5 business-logic triggers**, plus `updated_at` triggers, a tracking view and helper functions. The full script is in [`backend/sql/init.sql`](backend/sql/init.sql).
 
```mermaid
erDiagram
    CUSTOMER ||--o{ CUSTOMER_ADDRESS : has
    CUSTOMER ||--o{ ORDER_DETAILS : places
    CUSTOMER ||--o{ PLACES : ""
    ORDER_DETAILS ||--o{ PLACES : ""
    CUSTOMER ||--o{ RATING : writes
    RESTAURANT ||--o{ RATING : receives
    ADMIN |o--o{ RATING : moderates
    RESTAURANT ||--o{ RESTAURANT_LOCATION : "located at"
    RESTAURANT ||--o{ CATEGORY : has
    CATEGORY ||--o{ MENU_ITEM : contains
    RESTAURANT ||--o{ ORDER_DETAILS : receives
    CUSTOMER_ADDRESS |o--o{ ORDER_DETAILS : "delivered to"
    ORDER_DETAILS ||--|{ ORDERS : "line items"
    MENU_ITEM ||--o{ ORDERS : ""
    ORDER_DETAILS ||--o| PAYMENT_DETAILS : "paid by"
    ORDER_DETAILS ||--o| DELIVERY_STATUS : "delivered by"
    DRIVER |o--o{ DELIVERY_STATUS : assigned
    DRIVER ||--o{ DRIVER_LOCATION : reports
    DELIVERY_STATUS |o--o{ DRIVER_LOCATION : ""
```
 
### ENUM types
`order_status_type`, `delivery_status_type`, `driver_status_type`, `payment_mode_type`, `payment_status_type`, `address_type`. These stop invalid states (for example a misspelled order status) from ever reaching the database.
 
### Triggers
 
| Trigger | Fires on | What it does |
|---|---|---|
| `trg_update_driver_status` | insert or status update on `DELIVERY_STATUS` | Marks the driver **busy** while a delivery is assigned, picked up or in transit, and **available** once it's delivered or failed |
| `trg_set_actual_delivery_time` | status update on `DELIVERY_STATUS` | Records the exact time a delivery is first marked delivered |
| `trg_sync_order_status` | status update on `DELIVERY_STATUS` | Copies delivery progress to the order: in transit → out for delivery, delivered → delivered, failed → cancelled |
| `trg_populate_places` | insert on `ORDER_DETAILS` | Adds the customer–order link to `PLACES` for every new order |
| `trg_update_order_amount` | insert, update or delete on `ORDERS` | Recalculates the order total from its line items |
 
Because these rules live in the database, the API never updates driver or order status by hand, and they can't fall out of sync if another client writes to the tables.
 
### Indexes
9 B-tree indexes on the most-used lookups (customer email, order, driver and delivery status, menu category, restaurant ratings). Two are composite:
- `(delivery_id, timestamp)` on `DRIVER_LOCATION`, which serves the "latest driver location for this delivery" query in order tracking (`ORDER BY timestamp DESC LIMIT 1`)
- `(driver_id, timestamp)` on `DRIVER_LOCATION`, for a driver's location history
### Placing an order
`POST /api/orders` runs as **one database transaction**:
1. It looks up each item's price in the database instead of trusting prices sent by the client.
2. It validates item codes and quantities.
3. It inserts the order, its line items and a pending payment.
4. If any step fails, the whole order is rolled back.
---
 
## API
 
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/health` | Health check |
| POST | `/api/login/customer` | Customer sign-in (email, password) |
| POST | `/api/login/admin` | Admin sign-in (name, password) |
| GET | `/api/restaurants` | List restaurants |
| GET | `/api/restaurants/:restaurantId/menu` | Menu for one restaurant |
| POST | `/api/orders` | Place an order (transactional) |
| GET | `/api/orders/:orderId/tracking` | Order, delivery and driver status plus latest driver location |
| GET | `/api/customers/:id/orders` | A customer's orders |
| GET | `/api/customers/:id/addresses` | A customer's addresses |
| GET | `/api/customers/:id/delivery-map` | Delivery view data for one customer |
| GET | `/api/delivery-map` | Delivery view data for all orders |
| GET | `/api/analytics` | Dashboard metrics (aggregate SQL) |
| GET / POST | `/api/ratings`, `/api/addresses` | Read and add ratings and addresses |
| GET | `/api/customers`, `/api/drivers`, `/api/admins`, `/api/payments`, `/api/categories` | Read-only listings |
 
---
 
## Getting started
 
**Prerequisites:** Node.js 18+ and PostgreSQL 14+.
 
### 1. Create the database
```sql
CREATE DATABASE restaurant_delivery;
```
 
### 2. Configure and seed the back end
```bash
cd backend
npm install
```
Create `backend/.env`:
```env
PORT=5000
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/restaurant_delivery
```
Load the schema and sample data, using either command:
```bash
node seed_runner.js
# or: psql -U postgres -d restaurant_delivery -f sql/init.sql
```
> ⚠️ `init.sql` drops and recreates every FoodieHub table, so only run it against a development database.
 
### 3. Start the API
```bash
npm run dev               # http://localhost:5000
```
 
### 4. Start the front end
```bash
cd ../frontend
npm install
npm run dev               # http://localhost:5173
```
 
---
 
## Project structure
```
FoodieHub/
├── backend/
│   ├── sql/init.sql        # schema, ENUMs, indexes, triggers, view, functions, sample data
│   ├── src/server.js       # Express API
│   ├── src/db.js           # PostgreSQL connection pool
│   └── seed_runner.js      # runs init.sql against DATABASE_URL
└── frontend/
    └── src/
        ├── App.jsx         # ordering, dashboard, delivery view, data tabs
        └── LoginPage.jsx   # customer / admin sign-in
```
 
---
 
## Known limitations and next steps
 
This started as a database design project, so the back end is not production-hardened yet:
 
- [ ] **Passwords are stored and compared as plain text.** Next step: hash them with bcrypt.
- [ ] **The API doesn't check who is calling it.** Roles only control which tabs the front end shows. Next step: issue a JWT at sign-in and add middleware that checks the role on each endpoint.
- [ ] **The delivery view isn't a live map yet.** It shows coordinates and progress, and no driver locations are seeded. Next step: render the locations with Leaflet and poll or stream driver positions.
- [ ] Add automated API and trigger tests
- [ ] Move the API base URL into a Vite environment variable, and deploy it
---
 
## Author
 
**Vandit Gupta** · [GitHub](https://github.com/VanditGupta0) · [LinkedIn](https://linkedin.com/in/vandit-gupta-7537502a5)
 
