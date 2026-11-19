# Deploying NutriScan (FoodAnalyzer) on Zoho Catalyst

Architecture
- Backend (Spring Boot JAR)  -> Catalyst AppSail (Java 21, Java SE)
- Frontend (src/FoodScanner)  -> Catalyst Web Client Hosting
- Database                    -> PostgreSQL stays PostgreSQL (hosted, e.g. Supabase/Neon).
  Catalyst Data Store is NOT Postgres, so JPA can't use it without a rewrite.

## 1. Migrate your local data (do this first)
Your local DB: nutriscan on localhost:5432. Tables: food, user_details, user_info, user_meal
(names may differ slightly - pg_dump copies whatever exists).

    # dump (custom format)
    pg_dump -h localhost -U postgres -d nutriscan --no-owner --no-privileges -Fc -f nutriscan.dump

    # create a hosted Postgres (Supabase -> Project Settings -> Database -> connection string)
    # then restore into it
    pg_restore --no-owner --no-privileges --clean --if-exists \
      -d "postgresql://USER:PASS@HOST:5432/postgres" nutriscan.dump

    # verify row counts match
    psql "postgresql://USER:PASS@HOST:5432/postgres" -c "select count(*) from user_info;"

Keep nutriscan.dump as your backup. Don't point the app at the hosted DB until counts match.
Use the same major PostgreSQL version as local if possible.

## 2. Build the JAR
    ./mvnw clean package -DskipTests
    mkdir -p deploy && cp target/FoodAnalyzer-0.0.1-SNAPSHOT.jar deploy/

## 3. Create the Catalyst project + AppSail
    npm i -g zcatalyst-cli
    catalyst login
    catalyst init            # pick/create project; select AppSail (+ Client)
      - Catalyst-Managed Runtime
      - build path: ./deploy
      - stack: Java 21, platform: Java SE

Edit appsail's app-config.json:
    {
      "command": "java -jar FoodAnalyzer-0.0.1-SNAPSHOT.jar",
      "stack": "java21",
      "platform": "javase",
      "memory": 1024,
      "env_variables": {
        "DB_URL": "jdbc:postgresql://HOST:5432/postgres?sslmode=require",
        "DB_USERNAME": "...",
        "DB_PASSWORD": "...",
        "GEMINI_API_KEY": "...",
        "USDA_API_KEY": "...",
        "JWT_SECRET": "<random string, 32+ chars>",
        "DDL_AUTO": "update"
      }
    }
(Memory: Spring Boot + JPA needs roughly 512MB-1GB; raise if it fails to start.)

    catalyst deploy appsail

Note the AppSail endpoint URL it prints.
The app reads Catalyst's port from X_ZOHO_CATALYST_LISTEN_PORT (already wired in application.properties).

## 4. Frontend
Edit src/FoodScanner/config.js:
    window.NUTRISCAN_API_BASE_URL = "https://<your-appsail-url>";
Copy src/FoodScanner/* into the project's client/ folder (needs client-package.json
with "homepage": "index.html"), then:
    catalyst deploy client

## 5. Test
- Open the client URL, log in with an EXISTING user (proves migrated data + password hashes work).
- Check meal history shows your old meals.
- Scan/identify a food (proves Gemini/USDA keys).

## Important notes
- JWT_SECRET changed from the hardcoded value, so existing login tokens are invalidated; users just log in again.
  Passwords are BCrypt-hashed in the DB, so they keep working.
- Your old Gemini/USDA keys and JWT secret were in the repo (.git included). If it's on GitHub, rotate them.
- Free/low plans: AppSail may cold-start; first request can be slow.
