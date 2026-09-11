## 📦 Profast | A Parcel Delivery Service

[Live app](https://profast-eae10.web.app/) · [Frontend Repo](https://github.com/achibhossengit/profast-client)

ProFast is a full-stack parcel delivery management system where users can send parcels, track delivery status in real time, and make secure payments. Riders can earn money by delivering parcels, while admins manage users, riders, and delivery operations across Bangladesh.

This repository is the **backend API**. It handles business logic, REST endpoints, authentication, payments, and data storage.

## ✨ Key Features

- REST APIs: Users, riders, parcels, payments, and warehouse coverage.
- Authentication: Firebase Admin verifies ID tokens on protected routes.
- RBAC: Role checks for Admin, Rider, and User on sensitive endpoints.
- Parcel Workflow: Create, assign, update status, track, and conditionally delete parcels.
- Rider Applications: Users apply; admins accept or reject.
- Secure Payments: Stripe PaymentIntents and payment history.
- Rider Earnings: Calculated from assigned collect and deliver work.

## 🧑‍💻 Roles & Permissions

| Role         | Responsibilities                                                                |
| ------------ | ------------------------------------------------------------------------------- |
| **Admin**    | Manage all users, handle rider applications, assign riders, view parcel history |
| **Rider**    | View assigned parcels, update delivery statuses, view earnings                  |
| **User**     | Send parcels, make payments, view payment history                               |
| **All Role** | Update Profile, Change Password, Change Account Email                           |

## 🚀 Tech Stack

| Category       | Technology                 |
| -------------- | -------------------------- |
| Runtime        | Node.js, Express 5         |
| Database       | MongoDB (`ProFastDB`)      |
| Authentication | Firebase Admin             |
| Payments       | Stripe                     |
| Deployment     | Vercel                     |

## 🛠️ Installation & Setup

```bash
git clone https://github.com/achibhossengit/profast-backend.git
cd profast-backend
npm install
```

Create a `.env` file in the project root:

```env
PORT=5000
DB_USER=<Your database username>
DB_PASSWORD=<Your database password>
STRIPE_SECRET_KEY=<Your Stripe secret key>
FB_SERVICE_KEY=<Your Firebase service account key (Base64)>
```

`FB_SERVICE_KEY` is the Firebase service-account JSON encoded as Base64.

```bash
node index.js
```

The API runs at [http://localhost:5000](http://localhost:5000). For local reload, use `npx nodemon index.js`.

## 📁 Project Structure

```text
config/          MongoDB and Firebase Admin setup
controllers/     Business logic
middleware/      Token verification and role guards
routes/          REST route maps
services/        Stripe helpers
index.js         Server entry
vercel.json      Vercel deployment
```

## ⚙️ Workflows

### 1. Registration & Login

- The client authenticates with Firebase, then calls `POST /users`.
- A new email creates a MongoDB user with `role="user"`. An existing email updates `lastLoggedIn`.
- Protected routes verify the Firebase ID token and load the role from MongoDB.

### 2. Rider Application

- A user submits `POST /riders/applications`.
- The application is stored in the `riders` collection (region, district, NID, bike info).
- Admin accepts or rejects with `PATCH /riders/applications/:email/:accept`.
- Accept sets `role` to `rider` and copies rider details onto the user document.

### 3. Parcel Delivery

#### Send Parcel

- User creates a parcel → `delivery_status='pending'`, `payment_status='unpaid'`.
- After Stripe payment → `payment_status='paid'`.
- Update and delete are allowed only while the parcel is pending and unpaid.

#### Assign Rider (Admin)

- Admin assigns a rider based on the sender district.
- Parcel is updated with `assigned_to_collect` and status → `collecting`.

#### Rider Updates Status

- Rider collects → `collecting` → `collected`.

**Same District:**

- `collected` → `delivering` → `delivered` (same rider).

**Different District:**

- `collected` → `sendWarehouse` → Admin assigns a delivery rider → `delivering` → `delivered`.

#### Tracking

- Parcel status is read by ID through `GET /parcels/:id`.

### 4. Rider Earnings

- Collect: 35% of parcel cost when `assigned_to_collect`.
- Deliver: 35% of parcel cost when `assigned_to_deliver`.

## 🗃️ Database

MongoDB database: **`ProFastDB`**

| Collection   | Purpose                                      |
| ------------ | -------------------------------------------- |
| `users`      | Profiles, roles, and rider details           |
| `riders`     | Pending rider applications                   |
| `parcels`    | Parcel data, status, and assignments         |
| `payments`   | Stripe payment history                       |
| `warehouses` | Coverage regions, districts, and cities      |

## 🔐 Authentication

Protected routes expect:

```http
Authorization: Bearer <Firebase ID token>
```

| Guard                  | Rule                          |
| ---------------------- | ----------------------------- |
| `verifyFirebaseToken`  | Valid Firebase ID token       |
| `verifyAdmin`          | Role must be `admin`          |
| `verifyRider`          | Role must be `rider`          |
| `verifyUser`           | Role must be `user`           |

## 🔌 API Reference

### Users — `/users`

| Method   | Path              | Access | Description                    |
| -------- | ----------------- | ------ | ------------------------------ |
| `POST`   | `/users`          | token  | Create user or refresh login   |
| `GET`    | `/users/role`     | token  | Current user role              |
| `GET`    | `/users/profile`  | token  | Current user profile           |
| `PUT`    | `/users/profile`  | token  | Update profile                 |
| `PATCH`  | `/users/email`    | token  | Change account email           |
| `GET`    | `/users`          | admin  | Paginated user list            |
| `GET`    | `/users/:email`   | admin  | User by email                  |
| `DELETE` | `/users/:email`   | admin  | Delete user                    |

### Riders — `/riders`

| Method   | Path                                | Access | Description              |
| -------- | ----------------------------------- | ------ | ------------------------ |
| `POST`   | `/riders/applications`              | user   | Submit application       |
| `GET`    | `/riders/applications/:email`       | token  | Get application          |
| `PUT`    | `/riders/applications/:email`       | token  | Update own application   |
| `DELETE` | `/riders/applications/:email`       | token  | Delete own application   |
| `GET`    | `/riders/applications`              | admin  | List applications        |
| `PATCH`  | `/riders/applications/:email/:accept` | admin | Accept or reject (`true` / `false`) |
| `GET`    | `/riders/my-earnings`               | rider  | Earnings summary         |

### Parcels — `/parcels`

| Method   | Path                              | Access | Description                    |
| -------- | --------------------------------- | ------ | ------------------------------ |
| `GET`    | `/parcels/status-count`           | public | Counts by delivery status      |
| `GET`    | `/parcels`                        | token  | Role-scoped parcel list        |
| `GET`    | `/parcels/:id`                    | token  | Parcel by ID                   |
| `POST`   | `/parcels`                        | token  | Create parcel                  |
| `PUT`    | `/parcels/:id`                    | token  | Update if pending and unpaid   |
| `PATCH`  | `/parcels/:id/assign/:rider_email`| admin  | Assign collector or deliverer  |
| `PATCH`  | `/parcels/:id/status`             | rider  | Advance delivery status        |
| `DELETE` | `/parcels/:id`                    | token  | Delete if pending and unpaid   |

### Payments — `/payments`

| Method | Path                      | Access | Description                         |
| ------ | ------------------------- | ------ | ----------------------------------- |
| `POST` | `/payments/create-intent` | token  | Create Stripe PaymentIntent         |
| `POST` | `/payments`               | token  | Save payment and mark parcel paid   |
| `GET`  | `/payments`               | token  | Payment history                     |

### Warehouses — `/warehouseColl`

| Method | Path             | Access | Description              |
| ------ | ---------------- | ------ | ------------------------ |
| `GET`  | `/warehouseColl` | public | Coverage dataset         |
