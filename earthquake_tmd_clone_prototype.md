# EarthquakeTMD Clone — Prototype Boilerplate

นี่คือ repo prototype แบบ all-in-one ที่ผมสร้างให้อยู่ในเอกสารเดียว — มีทั้ง **backend (FastAPI)**, **mobile (Flutter skeleton)**, **infra (Terraform + gcloud scripts)**, และ **deployment + diagram** พร้อมคำสั่ง run จริง ๆ

> เปิดดูไฟล์แต่ละอันในเอกสารนี้ แล้วก็ `copy` ออกไปวางใน repo จริงได้เลย

---

## โครงสร้างไฟล์ที่รวมไว้ในเอกสารนี้

```
earthquake_tmd_clone_prototype/
├─ backend/
│  ├─ Dockerfile
│  ├─ requirements.txt
│  ├─ docker-compose.yml
│  ├─ .env.example
│  └─ app/
│     ├─ main.py
│     ├─ services/
│     │  ├─ earthquake.py
│     │  ├─ air_quality.py
│     │  └─ weather.py
│     └─ notifications/
│        └─ firebase_admin_init.py
├─ mobile/
│  ├─ pubspec.yaml
│  └─ lib/
│     ├─ main.dart
│     └─ screens/map_screen.dart
├─ infra/
│  ├─ gcloud_deploy.sh
│  └─ terraform/
│     ├─ main.tf
│     └─ variables.tf
└─ README.md
```

---

# 1) Backend — เอกสารไฟล์ (คัดลอกไปวางในโฟลเดอร์ backend/)

---

### backend/requirements.txt
```
fastapi
uvicorn[standard]
httpx
sqlalchemy
asyncpg
alembic
psycopg2-binary
databases
pydantic
redis
python-dotenv
apscheduler
geojson
requests
pandas
scikit-learn
firebase-admin
```

---

### backend/Dockerfile
```
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080", "--workers", "1"]
```

---

### backend/docker-compose.yml
```
version: "3.8"
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
  redis:
    image: redis:7
    ports:
      - "6379:6379"
  backend:
    build: .
    volumes:
      - ./:/app
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://dev:dev@db:5432/appdb
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - db
      - redis
```

---

### backend/.env.example
```
DATABASE_URL=postgresql://dev:dev@db:5432/appdb
REDIS_URL=redis://redis:6379/0
USGS_BASE=https://earthquake.usgs.gov/fdsnws/event/1/query
OPENWEATHER_API_KEY=your_openweather_api_key
OPENAQ_BASE=https://api.openaq.org/v2/latest
FIREBASE_SERVICE_ACCOUNT=./serviceAccount.json
FCM_TOPIC=all-users
```

---

### backend/app/main.py
```
from fastapi import FastAPI, BackgroundTasks
import httpx
import os
from dotenv import load_dotenv
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from app.services import earthquake, weather, air_quality
from app.notifications.firebase_admin_init import init_firebase, send_topic_notification

load_dotenv()

app = FastAPI(title="EarthquakeTMD Prototype")

@app.on_event("startup")
async def startup_event():
    init_firebase()
    scheduler = AsyncIOScheduler()
    # hourly PM2.5 job
    scheduler.add_job(func=air_quality.fetch_and_store_hourly, trigger='cron', minute=5)
    # daily weather job at 06:00
    scheduler.add_job(func=weather.fetch_and_store_daily, trigger='cron', hour=6, minute=0)
    scheduler.start()

@app.get('/health')
async def health():
    return {"status":"ok"}

@app.get('/api/earthquakes')
async def get_eqs(min_magnitude: float = 4.0):
    return await earthquake.fetch_usgs(min_magnitude=min_magnitude)

@app.get('/api/pm25')
async def get_pm25():
    return await air_quality.get_latest_all()

@app.get('/api/weather/today')
async def get_weather():
    return await weather.get_daily_summary_for_thailand()
```

---

### backend/app/services/earthquake.py
```
import httpx
import os
from urllib.parse import urlencode

USGS_BASE = os.getenv('USGS_BASE', 'https://earthquake.usgs.gov/fdsnws/event/1/query')

async def fetch_usgs(min_magnitude=4.0):
    # simplified: fetch last 30 days in Asia bounding box
    params = {
        'format': 'geojson',
        'minmagnitude': min_magnitude,
        'starttime': '2025-01-01',
        'minlatitude': -10,
        'maxlatitude': 60,
        'minlongitude': 60,
        'maxlongitude': 180,
        'limit': 200
    }
    url = USGS_BASE + '?' + urlencode(params)
    async with httpx.AsyncClient() as c:
        r = await c.get(url, timeout=30)
        r.raise_for_status()
        return r.json()
```

---

### backend/app/services/air_quality.py
```
import httpx
import os

OPENAQ_BASE = os.getenv('OPENAQ_BASE', 'https://api.openaq.org/v2/latest')

async def fetch_and_store_hourly():
    # prototype: just fetch and return
    params = {'limit': 100, 'page': 1}
    async with httpx.AsyncClient() as c:
        r = await c.get(OPENAQ_BASE, params=params, timeout=30)
        r.raise_for_status()
        data = r.json()
        # TODO: store into DB and run threshold checks
        return data

async def get_latest_all():
    async with httpx.AsyncClient() as c:
        r = await c.get(OPENAQ_BASE, params={'limit':100}, timeout=20)
        r.raise_for_status()
        return r.json()
```

---

### backend/app/services/weather.py
```
import httpx
import os

OPENWEATHER_KEY = os.getenv('OPENWEATHER_API_KEY')
OPENWEATHER_URL = 'https://api.openweathermap.org/data/2.5/onecall'

async def fetch_and_store_daily():
    # prototype: fetch weather for a list of coordinates in Thailand sample
    coords = [ (13.7563,100.5018) ]  # Bangkok as example
    results = []
    async with httpx.AsyncClient() as c:
        for lat,lon in coords:
            params = {'lat':lat,'lon':lon,'exclude':'minutely,hourly','appid':OPENWEATHER_KEY,'units':'metric'}
            r = await c.get(OPENWEATHER_URL, params=params, timeout=20)
            r.raise_for_status()
            results.append(r.json())
    # TODO: store to DB
    return results

async def get_daily_summary_for_thailand():
    return await fetch_and_store_daily()
```

---

### backend/app/notifications/firebase_admin_init.py
```
import firebase_admin
from firebase_admin import credentials, messaging
import os

def init_firebase():
    sa = os.getenv('FIREBASE_SERVICE_ACCOUNT')
    if sa and os.path.exists(sa):
        cred = credentials.Certificate(sa)
        firebase_admin.initialize_app(cred)
    else:
        # skip init in local dev if not provided
        pass


def send_topic_notification(topic, title, body, data=None):
    try:
        message = messaging.Message(
            notification=messaging.Notification(title=title, body=body),
            topic=topic,
            data=data or {}
        )
        resp = messaging.send(message)
        return resp
    except Exception as e:
        print('FCM error', e)
        return None
```

---

# 2) Mobile — Flutter skeleton (คัดลอกไปโฟลเดอร์ mobile/)

### mobile/pubspec.yaml
```
name: earthquake_tmd_clone
description: A prototype Flutter app for Earthquake + Weather + AQ
version: 0.1.0

environment:
  sdk: ">=2.18.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter
  http: ^0.14.0
  provider: ^6.0.0
  flutter_map: ^4.0.0
  firebase_core: ^2.0.0
  firebase_messaging: ^14.0.0
  geolocator: ^9.0.0
  flutter_local_notifications: ^12.0.0

flutter:
  uses-material-design: true
```

---

### mobile/lib/main.dart
```
import 'package:flutter/material.dart';
import 'screens/map_screen.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';

Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // Handle background message
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'EarthquakeTMD Prototype',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: MapScreen(),
    );
  }
}
```

---

### mobile/lib/screens/map_screen.dart
```
import 'package:flutter/material.dart';
import 'package:flutter_map/flutter_map.dart';
import 'package:latlong2/latlong.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

class MapScreen extends StatefulWidget {
  @override
  _MapScreenState createState() => _MapScreenState();
}

class _MapScreenState extends State<MapScreen> {
  List markers = [];

  @override
  void initState() {
    super.initState();
    fetchEarthquakes();
  }

  fetchEarthquakes() async {
    try {
      final r = await http.get(Uri.parse('http://10.0.2.2:8080/api/earthquakes'));
      if (r.statusCode == 200) {
        final j = json.decode(r.body);
        final features = j['features'] as List;
        setState((){
          markers = features.map((f) => f['geometry']['coordinates']).toList();
        });
      }
    } catch (e) {
      print('err $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Earthquake map')),
      body: FlutterMap(
        options: MapOptions(center: LatLng(15,100), zoom: 4),
        children: [
          TileLayer(
            urlTemplate: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
            subdomains: ['a','b','c'],
          ),
          MarkerLayer(
            markers: markers.map((c) {
              // c is [lon, lat, depth]
              return Marker(
                width: 30,
                height: 30,
                point: LatLng(c[1], c[0]),
                builder: (ctx) => Icon(Icons.location_on, color: Colors.red),
              );
            }).toList(),
          )
        ],
      ),
    );
  }
}
```

> **หมายเหตุ:** ใน Android emulator ใช้ `http://10.0.2.2:8080` เพื่อเรียก backend บนเครื่อง dev

---

# 3) Infra — gcloud + Terraform (คัดลอกไป infra/)

---

### infra/gcloud_deploy.sh
```
#!/bin/bash
PROJECT_ID=$1
IMAGE_NAME=gcr.io/${PROJECT_ID}/eq-backend:latest

# build and push
gcloud builds submit --tag ${IMAGE_NAME}

# deploy cloud run
gcloud run deploy eq-backend \
  --image ${IMAGE_NAME} \
  --platform managed \
  --region asia-southeast1 \
  --allow-unauthenticated

echo "Deployed to Cloud Run: eq-backend"
```

---

### infra/terraform/main.tf (minimal skeleton)
```
provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_cloud_run_service" "backend" {
  name     = "eq-backend"
  location = var.region

  template {
    spec {
      containers {
        image = var.container_image
        env {
          name  = "DATABASE_URL"
          value = var.database_url
        }
      }
    }
  }

  traffics {
    percent         = 100
    latest_revision = true
  }
}
```

### infra/terraform/variables.tf
```
variable "project_id" {}
variable "region" { default = "asia-southeast1" }
variable "container_image" {}
variable "database_url" {}
```

---

# 4) Architecture diagram (ASCII) + Deployment plan step-by-step

```
[USGS / OpenAQ / OpenWeather] --> [Cloud Run FastAPI] --> [Cloud SQL(Postgres) + Redis]
                                            |
                                            --> [Cloud Scheduler] (trigger fetch jobs)
                                            --> [FCM] (push notifications)
                                            --> [Mobile App Flutter]
```

## Deployment plan (short)
1. เตรียม GCP project + enable APIs: Cloud Run, Cloud Build, Cloud SQL, IAM, Cloud Scheduler
2. สร้าง Cloud SQL Postgres instance
3. ตั้งค่า service account และ secret (Firebase service account) ใน Secret Manager
4. `gcloud builds submit` & `gcloud run deploy` (หรือใช้ infra/gcloud_deploy.sh)
5. ตั้ง Cloud Scheduler job ให้เรียก endpoint ของ Cloud Run ทุกชั่วโมง/วัน
6. ตั้งค่า Firebase (FCM) และอัปโหลด `serviceAccount.json` ไปเก็บใน Secret Manager
7. ในมือถือ config FCM keys และทดสอบการรับ push

---

# 5) README.md (short)

```
# EarthquakeTMD Prototype

This repo contains a minimal prototype for an Earthquake + Weather + AQ app.

## Quickstart (local)

### Backend
1. cd backend
2. copy .env.example -> .env and edit
3. docker compose up --build
4. open http://localhost:8080/health

### Mobile
1. cd mobile
2. flutter pub get
3. flutter run

## Deploy (gcloud)
./infra/gcloud_deploy.sh <GCP_PROJECT_ID>
```

---

# 6) Next steps & recommended TODOs
- เพิ่ม DB schema + migrations (alembic)
- เก็บ raw data เข้า Postgres และเขียน query ใน PostGIS ถ้าต้องการ spatial
- เขียน rule-based alert engine + de-duplication
- ตั้ง monitoring (Sentry, Cloud Monitoring)
- ทำ CI (GitHub Actions) + Fastlane for mobile builds

---

ถ้าอยากให้ผมแปลงเอกสารนี้เป็น **zip** ดาวน์โหลดได้เลย หรือจะให้ผมแยกเป็น repo บน GitHub แบบตัวอย่าง (สร้าง repo skeleton + push) บอกมาได้เลย — ผมจะจัดให้ครบในข้อความเดียวต่อไป ✨

