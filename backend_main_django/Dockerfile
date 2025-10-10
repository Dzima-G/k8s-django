FROM python:3.9-slim
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /webapp

COPY requirements.txt /webapp/

RUN apt update && apt install -y gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/* \
    && pip3 install -r requirements.txt

COPY src /webapp/src
WORKDIR /webapp/srcыф

RUN SECRET_KEY=empty python manage.py collectstatic --noinput

CMD ["gunicorn", "src.wsgi:application", "--bind", "0.0.0.0:8000"]

EXPOSE 8000