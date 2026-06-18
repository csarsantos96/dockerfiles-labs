FROM python:3.11-slim AS build

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY static ./static
COPY templates ./templates


FROM gcr.io/distroless/python3

WORKDIR /app

COPY --from=build /app /app
COPY --from=build /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages

ENV PYTHONPATH=/usr/local/lib/python3.11/site-packages \
    FLASK_APP=app.py

EXPOSE 5000

ENTRYPOINT ["python3", "-m", "flask", "run", "--host=0.0.0.0"]
