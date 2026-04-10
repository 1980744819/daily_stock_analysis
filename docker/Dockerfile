# ===================================
# A股自选股智能分析系统 - Docker 镜像
# ===================================
# 多阶段构建：前端打包 + 后端运行

FROM 192.168.1.7:30002/base/node:20-slim AS web-builder

WORKDIR /app/apps/dsa-web

# 配置 npm 镜像加速
RUN npm config set registry https://registry.npmmirror.com

COPY apps/dsa-web/package.json apps/dsa-web/package-lock.json ./
RUN npm ci

COPY apps/dsa-web/ ./
RUN npm run build

# Pin to bookworm: wkhtmltopdf was removed from Debian testing (2025)
FROM 192.168.1.7:30002/base/python:3.11-slim-bookworm

WORKDIR /app

# 设置时区为上海
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 配置 pip 镜像加速
RUN pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple && \
    pip config set global.trusted-host pypi.tuna.tsinghua.edu.cn

# 配置代理
ENV HTTP_PROXY=http://192.168.1.7:7890
ENV HTTPS_PROXY=http://192.168.1.7:7890
ENV http_proxy=http://192.168.1.7:7890
ENV https_proxy=http://192.168.1.7:7890
ENV NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,.svc,.cluster.local
ENV no_proxy=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12,.svc,.cluster.local

# 配置 apt 镜像加速（清华或阿里云）
RUN echo 'deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware' > /etc/apt/sources.list && \
    echo 'deb http://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware' >> /etc/apt/sources.list && \
    echo 'deb http://mirrors.tuna.tsinghua.edu.cn/debian-security/ bookworm-security main contrib non-free non-free-firmware' >> /etc/apt/sources.list && \
    rm -f /etc/apt/sources.list.d/*.list

# 安装系统依赖（wkhtmltopdf 含 wkhtmltoimage，用于 Markdown 转图片 Issue #289）
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    curl \
    wkhtmltopdf \
    fontconfig \
    libjpeg62-turbo \
    libxrender1 \
    libxext6 \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装 Python 依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY *.py ./
COPY api/ ./api/
COPY data_provider/ ./data_provider/
COPY bot/ ./bot/
COPY patch/ ./patch/
COPY src/ ./src/
COPY strategies/ ./strategies/
COPY --from=web-builder /app/static ./static/

# 创建数据目录
RUN mkdir -p /app/data /app/logs /app/reports

# 设置环境变量默认值
ENV PYTHONUNBUFFERED=1
ENV LOG_DIR=/app/logs
ENV DATABASE_PATH=/app/data/stock_analysis.db
# Web/API service
ENV WEBUI_HOST=0.0.0.0
ENV API_PORT=8000

# 暴露 API 端口
EXPOSE 8000

# 数据卷（持久化数据）
VOLUME ["/app/data", "/app/logs", "/app/reports"]

# 健康检查（FastAPI 模式）
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/api/health || curl -f http://localhost:8000/health \
    || python -c "import sys; sys.exit(0)"

# 默认命令（可被覆盖）
CMD ["python", "main.py", "--schedule"]
