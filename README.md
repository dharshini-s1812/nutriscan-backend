<div align="center">

# NutriScan

### AI-Powered Nutrition Tracking & Health Analysis Platform

Monitor your diet, analyze nutritional intake, and get personalized health insights. Snap a photo of your food and let AI do the logging.

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)](https://www.java.com/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.x-brightgreen?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Supabase-336791?logo=postgresql&logoColor=white)](https://supabase.com/)
[![JWT](https://img.shields.io/badge/Auth-JWT-black?logo=jsonwebtokens)](https://jwt.io/)
[![Vercel](https://img.shields.io/badge/Frontend-Vercel-000000?logo=vercel&logoColor=white)](https://vercel.com/)
[![Zoho Catalyst](https://img.shields.io/badge/Backend-Zoho%20Catalyst-E42527)](https://catalyst.zoho.com/)
[![Gemini API](https://img.shields.io/badge/AI-Gemini%20API-4285F4?logo=googlegemini&logoColor=white)](https://ai.google.dev/)

</div>

---

## Table of Contents

- [About the Project](#about-the-project)
- [Tech Stack](#tech-stack)
- [Main Features](#main-features)
- [Application Workflow](#application-workflow)
- [Database Design](#database-design)
- [Project Structure](#project-structure)
- [API Endpoints](#api-endpoints)
- [Installation](#installation)
- [Deployment](#deployment)
- [Future Enhancements](#future-enhancements)
- [Contributing](#contributing)

---

## About the Project

**NutriScan** is a full-stack nutrition tracking app. A **Spring Boot** REST API backend and a lightweight **HTML/CSS/JavaScript** frontend let users log meals, track calories and macronutrients, and photograph their food to have it identified and logged automatically. Food recognition uses **Google's Gemini API**, and nutrition data comes from the **USDA FoodData Central** database.

The project is split into two repositories:

| Repo | Contents |
|---|---|
| `nutriscan-backend` (this repo) | Spring Boot REST API |
| `nutriscan-frontend` | Static web app, hosted on Vercel |

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Frontend | HTML5, Tailwind CSS, JavaScript |
| Backend | Java 21, Spring Boot 4, Spring Security, JWT, Maven |
| Database | PostgreSQL (Supabase) |
| External APIs | Gemini API (food recognition), USDA FoodData Central, Open Food Facts (barcodes) |

**Hosting**

| Layer | Platform |
|---|---|
| Frontend | [Vercel](https://vercel.com/) |
| Backend | [Zoho Catalyst AppSail](https://catalyst.zoho.com/) |
| Database | [Supabase](https://supabase.com/) |

---

## Main Features

- User registration and login
- JWT authentication (tokens expire after 1 hour)
- User profile management and BMI calculation
- Daily meal logging and nutrition tracking
- Dashboard analytics and personalized health suggestions
- USDA food search with local caching
- AI food image detection (Gemini)
- Barcode scanning
- Responsive design

---

## Application Workflow

1. The user registers or logs in, then completes their profile. BMI is calculated automatically.
2. The user logs a meal by searching manually, scanning a barcode, or uploading a food photo.
3. The backend checks the local `Food` table first and falls back to the USDA API if the item isn't cached, then saves the result locally.
4. For photos, the backend first tries to read a barcode, then asks Gemini to identify the food, and hands the name to the same lookup.
5. The meal is stored in PostgreSQL and the dashboard and analytics update.

<details>
<summary><b>Food lookup flow</b></summary>

```text
Search food / identify image
        │
        ▼
Check local Food table
        │
        ├── Found ──────────► Return nutrition data
        │
        └── Not found
                │
                ▼
        USDA FoodData Central
                │
                ▼
        Save food locally
                │
                ▼
            Add meal ──► Dashboard & analytics updated
```

</details>

---

## Database Design

```mermaid
erDiagram
    UserInfo ||--o| UserDetails : "has"
    UserInfo ||--o{ UserMeal : "logs"
    UserDetails ||--o{ user_allergies : "has"

    UserInfo {
        bigint Id PK
        varchar username
        varchar password
        datetime created_at
    }

    UserDetails {
        bigint userId PK
        varchar name
        int age
        double height
        double weight
        varchar gender
        varchar goal
        bigint user_id FK
    }

    user_allergies {
        bigint id PK
        bigint user_id FK
        varchar allergy
    }

    UserMeal {
        bigint mealId PK
        bigint user_id FK
        varchar mealType
        varchar foodName
        varchar portionSize
        double calories
        double protein
        double carbs
        double fat
        datetime created_at
    }

    Food {
        bigint foodId PK
        varchar foodName
        varchar servingUnit
        double calories
        double protein
        double carbs
        double fat
    }
```

> The `Food` table is a local cache filled on demand from the USDA API, so repeat lookups are fast and don't hit external rate limits.

---

## Project Structure

```
src/main/java/d/FoodAnalyzer/
├── Controller/     REST controllers (Auth, Food, Meals, Dashboard, ...)
├── Service/        Business logic, Gemini, USDA and Open Food Facts clients
├── Repository/     Spring Data JPA repositories
├── Model/          JPA entities (UserInfo, UserDetails, Food, UserMeal, ...)
├── DTO/            Request and response objects
├── Util/           JWT utility
├── JwtFilter.java
└── SecurityConfig.java
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Register a new user |
| POST | `/auth/login` | Log in and receive a JWT |
| POST | `/preferences` | Save profile, goals and allergies |
| PUT | `/updatePreferences` | Update profile information |
| GET | `/profile` | Get profile details |
| GET | `/bmi` | Get the user's BMI |
| POST | `/addMeal` | Add a meal entry |
| GET | `/todayMeal` | Today's meals |
| GET | `/meals/history` | Meal history by date and meal type |
| DELETE | `/deleteMeal/{mealId}` | Delete a meal |
| GET | `/dashboard` | Dashboard summary |
| GET | `/analytics` | Nutrition analytics and trends |
| POST | `/api/food/identify` | Identify food (or barcode) from an uploaded image |
| GET | `/api/food/search?q=` | Search foods |

> All endpoints except `/auth/register` and `/auth/login` need the header `Authorization: Bearer <token>`.

---

## Installation

### Requirements

JDK 21, a PostgreSQL database, a [Gemini API key](https://ai.google.dev/) and a [USDA API key](https://fdc.nal.usda.gov/api-key-signup).

### 1. Clone and configure

```bash
git clone https://github.com/<your-username>/nutriscan-backend.git
cd nutriscan-backend
```

No secrets are stored in the repo. All settings come from environment variables (see `.env.example`):

| Variable | Description |
|---|---|
| `DB_URL` | JDBC URL, e.g. `jdbc:postgresql://HOST:5432/postgres?sslmode=require` |
| `DB_USERNAME` / `DB_PASSWORD` | Database credentials |
| `JWT_SECRET` | Signing key, at least 32 characters (the app won't start without it) |
| `GEMINI_API_KEY` | Google Gemini API key |
| `USDA_API_KEY` | USDA FoodData Central key |
| `X_ZOHO_CATALYST_LISTEN_PORT` | Set by Catalyst. Defaults to 8080 locally |

### 2. Run the backend

```powershell
# PowerShell
$env:DB_URL="jdbc:postgresql://localhost:5432/nutriscan"
$env:DB_USERNAME="postgres"
$env:DB_PASSWORD="your-password"
$env:JWT_SECRET="a-random-string-of-at-least-32-characters"
$env:GEMINI_API_KEY="your-key"
$env:USDA_API_KEY="your-key"
./mvnw spring-boot:run
```

```bash
# macOS / Linux
export DB_URL=... DB_USERNAME=... DB_PASSWORD=... JWT_SECRET=... GEMINI_API_KEY=... USDA_API_KEY=...
./mvnw spring-boot:run
```

The API runs at `http://localhost:8080`.

The app uses `ddl-auto=validate`, so it checks the schema but never changes it. Create or restore the tables before the first start.

### 3. Run the frontend

Clone `nutriscan-frontend`, set `config.js` to `window.NUTRISCAN_API_BASE_URL = "http://localhost:8080";`, and serve the folder with any static server (for example `npx serve .`).

### Migrating an existing database

```bash
pg_dump -h localhost -U postgres -d nutriscan --no-owner --no-privileges -Fc -f nutriscan.dump
pg_restore --no-owner --no-privileges --clean --if-exists -d "postgresql://USER:PASS@HOST:5432/postgres" nutriscan.dump
```

Compare row counts afterwards. Never commit the dump file.

---

## Deployment

### Backend on Zoho Catalyst AppSail

```bash
./mvnw clean package -DskipTests
npm i -g zcatalyst-cli
catalyst login
catalyst init          # AppSail, Catalyst-managed runtime, Java 21 / Java SE
```

Copy `app-config.example.json` to your AppSail folder as `app-config.json` (it is git-ignored), fill in the real values, put the JAR next to it, then run:

```bash
catalyst deploy appsail
```

### Frontend on Vercel

Import the `nutriscan-frontend` repo into Vercel with the **Other** preset and no build command.

### CORS note

Catalyst's gateway answers browser preflight (`OPTIONS`) requests itself, so calling the AppSail URL directly from another origin gets blocked even though Spring allows it. The frontend avoids this by proxying API calls through Vercel rewrites (`vercel.json`), so the browser only talks to its own origin.

---

## Future Enhancements

- Dark mode
- Native mobile app
- Smart notifications and meal reminders

---

## Contributing

Contributions, issues and feature requests are welcome.

1. Fork the project
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request
