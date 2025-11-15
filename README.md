# Bike & Car Rental (Django REST Framework)

A production-ready web application for renting vehicles (cars and bikes) built with Django 4.2, Django REST Framework, PostgreSQL, and Stripe Checkout integration.

## Features

- **User Authentication** — Email/password registration, login, logout, profile management
- **Vehicle Management** — List, search, filter by type/seats/price/location
- **Booking System** — Date availability checking, prevent double-booking, booking status tracking (pending/confirmed/cancelled/completed)
- **Payments** — Stripe Checkout integration with webhook support for payment confirmation
- **Admin Dashboard** — Manage vehicles, bookings, payments; export bookings as CSV; confirm/cancel bookings
- **API** — RESTful endpoints with token and session authentication
- **Testing** — 10+ unit tests covering availability, booking creation, and webhook handling

## Tech Stack

- **Backend**: Python 3.10+, Django 4.2, Django REST Framework 3.14
- **Database**: PostgreSQL (production), SQLite (local development)
- **Payments**: Stripe (Checkout + Webhook)
- **Image Handling**: Pillow (optional; requires Python 3.10+ for wheel support)
- **Other**: python-dotenv, dj-database-url, django-environ

## Quick Start (Local Development)

### 1. Clone and Set Up Virtual Environment

```powershell
cd "C:\Users\aicha\OneDrive\Desktop\Project(bike or car rental)"
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 2. Install Dependencies

```powershell
python -m pip install --upgrade pip
pip install -r requirements.txt
```

**Note**: If Pillow fails to install on Python 3.13, it's commented out in `requirements.txt`. To use images, switch to Python 3.10 or 3.11 and reinstall.

### 3. Configure Environment Variables

```powershell
copy .env.example .env
```

Edit `.env` with your values:

```
SECRET_KEY=your-random-secret-key-here
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=sqlite:///db.sqlite3
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

For local dev, you can use the defaults. For production, use:
- A secure `SECRET_KEY` (e.g., generate with `python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"`)
- `DEBUG=False`
- A PostgreSQL `DATABASE_URL`
- Real Stripe test keys from https://dashboard.stripe.com/test/apikeys

### 4. Create Database and Run Migrations

```powershell
python manage.py migrate
python manage.py createsuperuser
python manage.py seed_demo_data
```

The `seed_demo_data` command creates:
- Admin user: `admin` / `password` (change this after first login!)
- 2 demo vehicles (a car and a bike)

### 5. Start Development Server

```powershell
python manage.py runserver
```

Open http://127.0.0.1:8000/ in your browser. You should see the API home page with endpoint links.

## API Endpoints

### Public Endpoints

**GET** `/api/vehicles/`
- List all active vehicles with optional filters
- Query params: `type` (car|bike), `seats`, `min_price`, `max_price`, `city`
- Example: `/api/vehicles/?type=car&seats=4&min_price=30&max_price=100`

**GET** `/api/vehicles/{id}/`
- Retrieve a single vehicle with images

### Protected Endpoints (Requires Authentication)

**POST** `/api/auth/token/`
- Obtain auth token
- Body: `{"username": "...", "password": "..."}`
- Response: `{"token": "..."}`

**POST** `/api/bookings/`
- Create a booking (user authenticated, required)
- Body: `{"vehicle": 1, "start_date": "2025-12-01", "end_date": "2025-12-03"}`
- Returns: `{"id": 1, "vehicle": 1, "start_date": "2025-12-01", "end_date": "2025-12-03", "total_price": "100.00", "status": "pending", ...}`
- Validations:
  - start_date <= end_date
  - Vehicle must be available (no overlapping confirmed/pending bookings)
  - total_price = (days * price_per_day) + deposit

**GET** `/api/bookings/my/`
- List user's bookings (authenticated)

**POST** `/api/payments/create-checkout/`
- Create Stripe Checkout session for a booking (authenticated)
- Body: `{"booking_id": 1}`
- Returns: `{"sessionId": "sess_..."}`
- Redirect user to Stripe Checkout with this sessionId

**POST** `/api/webhooks/stripe/`
- Stripe webhook endpoint (no auth required)
- Stripe will POST here when payment completes
- On `checkout.session.completed`, booking is marked `confirmed` and payment marked `paid`

### Admin Endpoints

- **GET/POST** `/admin/` — Django admin panel (superuser only)
  - Manage vehicles, bookings, payments, reviews
  - Actions: confirm booking, export bookings to CSV
  - Change booking status manually if needed

## Testing

Run the test suite:

```powershell
python manage.py test
```

Tests cover:
- Availability checking (no bookings, overlap detection, adjacent days)
- Booking creation (price calculation, validation)
- API endpoints (authentication, filtering, pagination)
- Stripe webhook handling (mocked)

Expected output: 10+ tests passing.

## Stripe Webhook Testing (Local)

To test Stripe webhooks locally, use the Stripe CLI:

1. Download and install Stripe CLI: https://stripe.com/docs/stripe-cli
2. Login to your Stripe account:
   ```powershell
   stripe login
   ```
3. Forward webhooks to your local endpoint:
   ```powershell
   stripe listen --forward-to http://localhost:8000/api/webhooks/stripe/
   ```
4. In another terminal, trigger a test event or complete a checkout:
   ```powershell
   stripe trigger checkout.session.completed
   ```

The webhook should update your booking to `confirmed` status in the database.

## Booking Availability Rules

**Overlap Detection**: A new booking conflicts if:
```
existing.start_date <= new.end_date AND existing.end_date >= new.start_date
```

**End Date Inclusive**: The end_date is inclusive (guest leaves on end_date).
- Booking from Jan 10–15 blocks Jan 16 (next booking can start Jan 16).
- If you want Jan 15 to be the last day of the first booking, the next booking must start Jan 16 or later.

## Models

### Vehicle
- `id` (PK)
- `owner` (FK to User, nullable — for future host/owner features)
- `type` (car | bike)
- `title`, `description`, `make`, `model`, `year`, `seats`
- `price_per_day`, `deposit`
- `location_city`, `location_state`
- `is_active` (bool)
- `created_at` (timestamp)

### Booking
- `id` (PK)
- `user` (FK to User)
- `vehicle` (FK to Vehicle)
- `start_date`, `end_date` (dates, inclusive)
- `total_price` (calculated: days * price_per_day + deposit)
- `status` (pending | confirmed | cancelled | completed)
- `created_at` (timestamp)

### Payment
- `id` (PK)
- `booking` (1:1 to Booking)
- `amount` (decimal)
- `stripe_payment_intent` (CharField for Stripe PI ID)
- `paid` (bool)
- `paid_at` (timestamp, nullable)

### Review (optional)
- `id` (PK)
- `booking` (1:1 to Booking)
- `rating` (int 1–5)
- `comment` (text)
- `created_at` (timestamp)

### VehicleImage
- `id` (PK)
- `vehicle` (FK to Vehicle, cascade delete)
- `image` (ImageField, optional — requires Pillow)
- `alt_text` (CharField)

## Deployment

### Environment Variables Required

Create a `.env` file or set these in your host's environment:

```
SECRET_KEY                  # Django secret (generate a secure one)
DEBUG                       # False for production
ALLOWED_HOSTS               # Comma-separated domain list
DATABASE_URL                # PostgreSQL connection string
STRIPE_SECRET_KEY           # Stripe secret key (sk_test_... or sk_live_...)
STRIPE_WEBHOOK_SECRET       # Stripe webhook signing secret (whsec_...)
REDIS_URL                   # (optional) Redis connection for background jobs
```

### Option A: Deploy to Render

1. Push your repo to GitHub.
2. Create a new Web Service on Render.com linked to your repo.
3. Add these environment variables in Render's settings.
4. Set Build Command: `pip install -r requirements.txt && python manage.py migrate`
5. Set Start Command: `gunicorn config.wsgi`
6. Render will auto-redeploy on push.

### Option B: Deploy to Heroku

1. Install Heroku CLI: https://devcenter.heroku.com/articles/heroku-cli
2. Create app: `heroku create your-app-name`
3. Add PostgreSQL addon: `heroku addons:create heroku-postgresql:mini`
4. Set environment variables:
   ```powershell
   heroku config:set SECRET_KEY="your-secret" DEBUG=False STRIPE_SECRET_KEY="sk_..." STRIPE_WEBHOOK_SECRET="whsec_..."
   ```
5. Deploy: `git push heroku main`
6. Run migrations: `heroku run python manage.py migrate`

### Option C: Docker (DigitalOcean or Local)

A `docker-compose.yml` is included for local Postgres + Redis setup:

```powershell
docker-compose up --build
docker-compose exec web python manage.py migrate
docker-compose exec web python manage.py createsuperuser
docker-compose exec web python manage.py seed_demo_data
```

Access at http://localhost:8000.

## Production Checklist

- [ ] Set `DEBUG=False` in `.env`
- [ ] Use a strong, randomly generated `SECRET_KEY`
- [ ] Use PostgreSQL (not SQLite)
- [ ] Set `ALLOWED_HOSTS` to your domain(s)
- [ ] Enable HTTPS only (set `SECURE_SSL_REDIRECT=True`)
- [ ] Configure static files and media uploads (S3 or DigitalOcean Spaces)
- [ ] Add Stripe live keys (when ready)
- [ ] Set up Stripe webhook in your Stripe dashboard pointing to `https://yourdomain.com/api/webhooks/stripe/`
- [ ] Run `collectstatic` to gather static files
- [ ] Use a production WSGI server (gunicorn, uWSGI)

## File Structure

```
project-root/
├── config/                  # Django project settings
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── __init__.py
├── rentals/                 # Main app
│   ├── models.py            # Vehicle, Booking, Payment, Review, VehicleImage
│   ├── views.py             # DRF views & Stripe webhook
│   ├── serializers.py       # DRF serializers
│   ├── urls.py              # API URL routing
│   ├── admin.py             # Django admin registration & actions
│   ├── utils.py             # is_vehicle_available() function
│   ├── tests.py             # 10+ unit tests
│   ├── apps.py
│   ├── migrations/          # Database migrations
│   ├── management/
│   │   └── commands/
│   │       └── seed_demo_data.py  # Demo data seeding
│   └── __init__.py
├── manage.py                # Django CLI
├── requirements.txt         # Python dependencies
├── Procfile                 # Heroku/Render deployment config
├── docker-compose.yml       # Local Docker Compose setup
├── .env.example             # Example environment variables (no secrets)
└── README.md                # This file
```

## Known Limitations & Future Improvements

1. **Image Uploads** — Pillow is optional; requires Python 3.10+. Enable by switching Python versions and installing Pillow.
2. **Frontend** — Currently API-only. Future: add Tailwind/Bootstrap templates for vehicle listing, booking form, user dashboard.
3. **Email Notifications** — Bookings don't send confirmation emails yet. Future: add Celery + Redis for async email tasks.
4. **Owner Dashboard** — Vehicles can have an owner, but owner features not yet implemented.
5. **Reviews** — Review model exists but no endpoints to create/list reviews yet.
6. **Cancellation Policy** — No refund logic; bookings can be cancelled manually via admin.
7. **Maps** — No location map integration; location_city/state are text fields only.

## Troubleshooting

**ModuleNotFoundError: No module named 'django'**
- Ensure venv is activated: `.\.venv\Scripts\Activate.ps1`
- Reinstall: `pip install -r requirements.txt`

**No such table: rentals_vehicle**
- Migrations haven't run: `python manage.py migrate`

**Pillow installation fails**
- Upgrade to Python 3.10+ or comment out Pillow in `requirements.txt`

**Stripe webhook not triggering**
- Ensure `STRIPE_WEBHOOK_SECRET` is correct in `.env`
- Use `stripe listen` command to forward events to `http://localhost:8000/api/webhooks/stripe/`

## Contributing

Fork, branch, and submit pull requests. Ensure tests pass: `python manage.py test`

## License

MIT (or your choice)

---

**Questions?** Check the API endpoints at http://127.0.0.1:8000/ or review the code in `rentals/` folder.
#   b i k e - r e n t a l  
 #   b i k e - r e n t a l  
 #   b i k e - r e n t a l  
 