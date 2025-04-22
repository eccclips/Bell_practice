FROM node:18.18.0 AS builder
WORKDIR /app
# COPY package*.json ./
COPY . .
RUN npm install && npm run build
 
FROM nginx:latest
COPY --from=builder /app/dist /etc/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80