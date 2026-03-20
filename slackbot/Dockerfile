FROM python:3.13-slim-bookworm

LABEL org.opencontainers.image.title="Kagent Slack Bot"
LABEL org.opencontainers.image.description="Slack bot for Kagent AI agents with HITL approval support"

RUN groupadd --gid 1000 kagent && \
    useradd --uid 1000 --gid kagent --shell /bin/bash --create-home kagent

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    rm -rf /root/.cache /tmp/* /var/tmp/*
COPY --chown=kagent:kagent slack_bot.py .

USER 1000
CMD ["python", "-u", "slack_bot.py"]
