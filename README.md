# Apex Learning — Full-stack course store

React + Vite frontend, Express API, MongoDB persistence, JWT registration/login, cart, and Stripe-hosted Checkout. Course catalogue data and prices are loaded from MongoDB; seed the sample courses before starting the app.

> This is a development starter, not a deployed live store. You must configure your own MongoDB and Stripe test credentials and verify the complete test payment flow before production.

## Requirements

- Node.js 20+ and npm
- MongoDB running locally **or** a MongoDB Atlas cluster
- Stripe account in test mode
- Stripe CLI for local webhook testing
- Two terminals (frontend and backend)

## Folder layout

```text
apex-course-fullstack/
├── frontend/
│   ├── .env.example        # VITE_API_URL goes here
│   └── src/
│       ├── App.jsx         # screens, register/login, course list, cart, checkout return
│       ├── api.js          # shared fetch helper + JWT header
│       ├── main.jsx
│       └── styles.css
└── backend/
    ├── .env.example        # MongoDB, JWT, Stripe secrets go here
    ├── scripts/seedCourses.js
    └── src/
        ├── server.js
        ├── config/db.js
        ├── middleware/auth.js
        ├── models/         # User, Course, Cart, Order
        └── routes/         # auth, courses, cart, checkout, orders, webhook
```

## 1. Set up MongoDB

### Option A: Local MongoDB

Start MongoDB locally. The sample connection URI is:

`mongodb://127.0.0.1:27017/apex_learning`

### Option B: MongoDB Atlas

1. Create a cluster and a database user in Atlas.
2. In **Network Access**, allow your development IP address.
3. Choose **Connect → Drivers** and copy the connection URI.
4. Put it in `backend/.env` as `MONGODB_URI`. Replace the username, password and database name with your own values. URL-encode special characters in the password.

No manually created collection IDs are required. Mongoose creates the collections, and the seed script inserts the sample courses.

## 2. Configure and start the backend

In terminal 1, from this project directory:

```bash
cd backend
cp .env.example .env
npm install
```

Open `backend/.env` and set these values:

```dotenv
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://127.0.0.1:27017/apex_learning
JWT_SECRET=REPLACE_WITH_A_RANDOM_SECRET
JWT_EXPIRES_IN=7d
CLIENT_URL=http://localhost:5173
STRIPE_SECRET_KEY=sk_test_YOUR_STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET=whsec_REPLACE_AFTER_STRIPE_LISTEN
CURRENCY=inr
```

Generate a JWT secret in a separate terminal with:

```bash
openssl rand -hex 32
```

Copy the generated output into `JWT_SECRET`. Do not use the example placeholder in a real environment.

### Where to get the Stripe key

1. Sign in to the Stripe Dashboard: https://dashboard.stripe.com/test/apikeys
2. Enable **test mode**.
3. Copy the **Secret key** beginning with `sk_test_` into `backend/.env` as `STRIPE_SECRET_KEY`.
4. This project uses Stripe's hosted Checkout URL, so a Stripe publishable key is not needed in the frontend.
5. Keep all `sk_...` and `whsec_...` values on the backend only. Never add them to a `VITE_...` variable or commit `.env` to Git.

Seed the demo courses and start the API:

```bash
npm run seed
npm run dev
```

Expected API health check: http://localhost:5000/api/health — it should return `{"status":"ok"}`.

## 3. Configure and start the frontend

Open terminal 2 at the project root:

```bash
cd frontend
cp .env.example .env
npm install
```

Set `frontend/.env` to:

```dotenv
VITE_API_URL=http://localhost:5000/api
```

**This is the API base URL.** If your backend runs on another port or hosted domain, update this value to the public API base URL ending in `/api`, then restart Vite.

Start the frontend:

```bash
npm run dev
```

Open the URL Vite prints, usually http://localhost:5173.

## 4. Register, login, cart and checkout test

1. Make sure MongoDB and the backend are running, and `/api/health` responds.
2. Make sure the frontend `VITE_API_URL` points at that backend.
3. Open Courses and choose a course. Add to cart will ask you to register/sign in if you're logged out.
4. Register with a name, valid email and password of at least 8 characters. The server hashes the password with bcrypt and returns a JWT.
5. Add the course, open Cart and proceed to checkout.
6. Complete payment on Stripe Checkout using a Stripe test card from https://docs.stripe.com/testing. Use the expiry/CVC values listed in Stripe's current test documentation.
7. The order is only marked paid after a valid Stripe webhook is received. The return page polls the backend for status.

## 5. Set up Stripe webhook for local testing

Keep the backend running. In terminal 3, install and sign into Stripe CLI (instructions: https://docs.stripe.com/stripe-cli), then run:

```bash
stripe login
stripe listen --forward-to localhost:5000/api/payments/webhook
```

The CLI prints a webhook signing secret beginning with `whsec_`. Copy that exact value into `backend/.env` as `STRIPE_WEBHOOK_SECRET`, replacing the placeholder, then restart the backend. The CLI secret can change between listening sessions; update it whenever needed.

For production, create a webhook endpoint in Stripe Dashboard pointing to:

`https://YOUR_BACKEND_DOMAIN/api/payments/webhook`

Configure the signing secret shown for that endpoint in your production backend environment. Never expose it in frontend code.

## 6. Environment variables — exact locations

| Value | Put it here | Where it comes from |
|---|---|---|
| `VITE_API_URL` | `frontend/.env` | Your backend base URL, e.g. `http://localhost:5000/api` |
| `MONGODB_URI` | `backend/.env` | Local MongoDB URI or Atlas connection URI |
| `JWT_SECRET` | `backend/.env` | Generate using `openssl rand -hex 32` |
| `STRIPE_SECRET_KEY` | `backend/.env` | Stripe Dashboard test secret key, starts `sk_test_` |
| `STRIPE_WEBHOOK_SECRET` | `backend/.env` | Stripe CLI output `whsec_...` for local testing, or webhook endpoint secret in production |
| `CLIENT_URL` | `backend/.env` | Frontend origin; local value `http://localhost:5173` |
| `PORT` | `backend/.env` | Backend port; default `5000` |
| `CURRENCY` | `backend/.env` | `inr` for these sample prices, only if enabled/supported by your Stripe account |

You do **not** need to create a Stripe product ID or price ID manually. Checkout line items are created from trusted course prices in MongoDB. Course slugs are seeded by `npm run seed`.

## API reference

All API paths below are appended to the base URL (default `http://localhost:5000/api`). Protected endpoints need `Authorization: Bearer <JWT>`.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/health` | No | API health check |
| POST | `/auth/register` | No | Register `{name,email,password}` |
| POST | `/auth/login` | No | Login `{email,password}` |
| GET | `/auth/me` | JWT | Validate session / current user |
| GET | `/courses` | No | List active courses from MongoDB |
| GET | `/courses/:slug` | No | Course detail |
| GET | `/cart` | JWT | Current user's cart |
| POST | `/cart/items` | JWT | Add `{courseSlug}` |
| DELETE | `/cart/items/:slug` | JWT | Remove course |
| DELETE | `/cart` | JWT | Clear cart |
| POST | `/checkout` | JWT | Create pending order and Stripe Checkout Session |
| GET | `/orders` | JWT | Current user's orders |
| GET | `/orders/:id` | JWT | One of the current user's orders |
| GET | `/orders/payment-session/:sessionId` | JWT | Check current user's payment/order status |
| POST | `/payments/webhook` | Stripe signature | Stripe payment event receiver |

## Data models

- `User`: name, normalized unique email, bcrypt password hash.
- `Course`: slug, title, category, mode, grade, duration, description, price in paise, active flag.
- `Cart`: one cart per user with course references.
- `Order`: user, immutable item/price snapshots, currency, total, Stripe session/payment IDs, status and paid timestamp.

## Before production

- Deploy frontend and backend to HTTPS-capable hosting; set `VITE_API_URL` to the deployed API URL and `CLIENT_URL` to the deployed frontend origin.
- Set production environment variables in the host dashboard, not in committed files.
- Use live Stripe keys only after test payments and webhook verification work. Check Stripe account eligibility and supported currencies/payment methods for India.
- Add email verification, password reset, order/course entitlement granting, refund handling, admin course management, webhook event logging, rate limits for all sensitive endpoints, monitoring and backups.
- For a commercial site, consider HttpOnly secure cookies instead of localStorage JWT storage, plus CSRF protections.
