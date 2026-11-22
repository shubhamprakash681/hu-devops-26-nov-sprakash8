# Stage 1: Build & Dependencies
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

# Stage 2: Hardened Runtime
FROM node:20-alpine
WORKDIR /app
RUN apk upgrade --no-cache && rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx
COPY --from=build /app/node_modules ./node_modules
COPY . .
EXPOSE 8080
USER node
CMD ["node", "src/server.js"]
