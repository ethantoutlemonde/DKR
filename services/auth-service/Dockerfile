# ---- Base ----
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# ---- Development ----
FROM node:18-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3001
CMD ["npm", "run", "dev"]

# ---- Production ----
FROM node:18-alpine AS production
WORKDIR /app

# Creer un utilisateur non-root pour la securite
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=base /app/node_modules ./node_modules
COPY package*.json ./
COPY src ./src

# Changer le proprietaire des fichiers
RUN chown -R appuser:appgroup /app

USER appuser
EXPOSE 3001
CMD ["npm", "start"]
