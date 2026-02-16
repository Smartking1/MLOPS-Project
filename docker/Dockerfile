FROM python:3.10-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first for better caching
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Create a non-root user
RUN useradd -m -u 1000 appuser

# Create a directory for NLTK data that is writable by the non-root user
ENV NLTK_DATA=/app/nltk_data
RUN mkdir -p /app/nltk_data && \
    python -c "import nltk; nltk.download('stopwords', download_dir='/app/nltk_data', quiet=True)" && \
    chown -R appuser:appuser /app/nltk_data

# Copy the rest of the application code
COPY . .

# Create models and logs directories with correct permissions
RUN mkdir -p logs models && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Expose API port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run the application
CMD ["python", "-m", "uvicorn", "src.api.app:app", "--host", "0.0.0.0", "--port", "8000"]