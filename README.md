version: '3.8'

services:
  # ========== DATABASES ==========
  mysql:
    image: mysql:8.0
    container_name: lab-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      TZ: ${TZ}
    volumes:
      - ./data/db/mysql:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - lab-network

  postgres:
    image: postgres:15-alpine
    container_name: lab-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      TZ: ${TZ}
    volumes:
      - ./data/db/postgres:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - lab-network

  redis:
    image: redis:7-alpine
    container_name: lab-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - ./data/db/redis:/data
    ports:
      - "6379:6379"
    networks:
      - lab-network

  # ========== REVERSE PROXY ==========
  nginx:
    image: nginx:alpine
    container_name: lab-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./data/logs/nginx:/var/log/nginx
    depends_on:
      - bot-telegram
      - tool-crawler
    networks:
      - lab-network

  # ========== SERVICES ==========
  bot-telegram:
    build: ./services/bot-telegram
    container_name: lab-bot-telegram
    restart: unless-stopped
    environment:
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      TELEGRAM_CHAT_ID: ${TELEGRAM_CHAT_ID}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      TZ: ${TZ}
    volumes:
      - ./data/logs/bot-telegram:/app/logs
    depends_on:
      - redis
    networks:
      - lab-network

  tool-crawler:
    build: ./services/tool-crawler
    container_name: lab-tool-crawler
    restart: unless-stopped
    environment:
      MYSQL_HOST: mysql
      MYSQL_PORT: 3306
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      TZ: ${TZ}
    volumes:
      - ./data/logs/tool-crawler:/app/logs
    depends_on:
      - mysql
    networks:
      - lab-network

networks:
  lab-network:
    driver: bridge
    server {
    listen 80;
    server_name localhost;

    # Bot Telegram (nếu có webhook hoặc health check)
    location /bot/ {
        proxy_pass http://bot-telegram:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Tool Crawler (ví dụ có API xem kết quả)
    location /crawler/ {
        proxy_pass http://tool-crawler:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Trang mặc định
    location / {
        return 200 "VPS Lab - All systems operational\n";
        add_header Content-Type text/plain;
    }
}
FROM python:3.11-slim

WORKDIR /app

# Cài đặt dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy code
COPY main.py .

# Tạo thư mục logs
RUN mkdir -p /app/logs

# Mở port 5000 để health check (tùy chọn)
EXPOSE 5000

CMD ["python", "main.py"]
#!/usr/bin/env python3
import os
import logging
import asyncio
from datetime import datetime

from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes
import redis

# Cấu hình logging
log_dir = "/app/logs"
os.makedirs(log_dir, exist_ok=True)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(os.path.join(log_dir, "bot.log")),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# Biến môi trường
TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")  # Có thể dùng để gửi thông báo

# Kết nối Redis
redis_client = redis.Redis(
    host=os.getenv("REDIS_HOST", "localhost"),
    port=int(os.getenv("REDIS_PORT", 6379)),
    password=os.getenv("REDIS_PASSWORD"),
    decode_responses=True
)

# Command handlers
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🤖 Bot Telegram Lab đã sẵn sàng!")

async def ping(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🏓 Pong!")

async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        redis_client.ping()
        redis_status = "✅ Redis OK"
    except Exception as e:
        redis_status = f"❌ Redis lỗi: {e}"
    
    msg = f"🖥 Trạng thái hệ thống:\n- {redis_status}\n- Thời gian: {datetime.now()}"
    await update.message.reply_text(msg)

async def echo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    await update.message.reply_text(f"Bạn nói: {text}")

def main():
    if not TOKEN:
        logger.error("Thiếu TELEGRAM_BOT_TOKEN trong biến môi trường!")
        return
    
    app = Application.builder().token(TOKEN).build()
    
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("ping", ping))
    app.add_handler(CommandHandler("status", status))
    # Echo tất cả tin nhắn không phải command
    from telegram.ext import MessageHandler, filters
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, echo))
    
    logger.info("Bot đang khởi động...")
    app.run_polling()

if __name__ == "__main__":
    # Khởi động một web server đơn giản trên port 5000 để health check
    from http.server import HTTPServer, BaseHTTPRequestHandler
    import threading
    
    class HealthHandler(BaseHTTPRequestHandler):
        def do_GET(self):
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"Bot is running")
    
    def run_health_server():
        server = HTTPServer(('0.0.0.0', 5000), HealthHandler)
        server.serve_forever()
    
    threading.Thread(target=run_health_server, daemon=True).start()
    
    main()
