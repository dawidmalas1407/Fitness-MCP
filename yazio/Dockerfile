FROM node:20-slim
RUN npm config set fetch-retries 6 && npm config set fetch-retry-maxtimeout 180000 && npm install -g supergateway@4.2.0 yazio-mcp@0.0.14
EXPOSE 8001
CMD ["supergateway", "--stdio", "yazio-mcp", \
     "--outputTransport", "streamableHttp", \
     "--port", "8001", "--streamableHttpPath", "/yazio/mcp"]
