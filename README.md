# VPS Docker Lab 🚀

A microservices-based automation platform that transforms a VPS into a centralized control system using Docker.

---

## 🧩 Architecture

* API Gateway (FastAPI)
* Telegram Bot Service
* Crawler Service
* MySQL & PostgreSQL
* Redis (cache / message broker)
* Nginx (reverse proxy)

---

## ⚙️ Setup

```bash
git clone https://github.com/your-username/vps-docker-lab.git
cd vps-docker-lab

cp .env.example .env
docker-compose up --build
```

---

## 🌐 Services

| Service         | Endpoint                  |
| --------------- | ------------------------- |
| API Gateway     | http://localhost:8000     |
| Bot (via Nginx) | http://localhost/bot/     |
| Crawler         | http://localhost/crawler/ |

---

## 🔁 Example Flow

1. User sends command to Telegram Bot
2. Bot communicates with API Gateway
3. Gateway triggers Crawler Service
4. Data stored in database
5. Response returned to user

---

## 📂 Project Structure

```
.
├── docker-compose.yml
├── nginx/
├── services/
│   ├── bot-telegram/
│   └── tool-crawler/
├── data/
│   ├── db/
│   └── logs/
```

---

## 📌 Notes

* All services run via Docker
* Logs are stored in `./data/logs`
* Redis is used for caching and messaging
* Environment variables are managed via `.env`

---

## 🚀 Future Improvements

* Add API authentication (JWT)
* Add monitoring (Prometheus + Grafana)
* Add task queue (Celery / Redis Queue)
* CI/CD pipeline
