# Node ka lightweight base image
FROM node:18-alpine

# Container ke andar working directory
WORKDIR /app

# Pehle lock + package files copy karo (caching ke liye)
COPY package*.json ./

# Production-grade, reproducible install (dev dependencies skip)
RUN npm ci --omit=dev

# Baaki saara code copy karo
COPY . .

# Non-root user par chalao (security best practice)
USER node

# Backend port 3001 par chalta hai
EXPOSE 3001

# App start command
CMD ["node", "index.js"]