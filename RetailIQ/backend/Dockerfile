# Pinned to 3.12 (not the 3.14 used in local dev) for broad prebuilt-wheel
# availability across fastapi/pandas/numpy/scikit-learn on the slim Linux image.
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

# Migrations run on every container start - safe/idempotent (alembic no-ops
# if already at head), so the DB schema is always in place before the API
# starts serving. Seeding is a deliberate separate step (see README) since
# it's a one-time operation, not something that should re-run on restart.
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000"]
