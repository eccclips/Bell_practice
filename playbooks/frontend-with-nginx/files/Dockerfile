FROM node:18.18.0 AS builder
WORKDIR /app
# COPY package*.json ./
COPY . .
RUN npm install && npm run build
 
FROM nginx:latest
COPY --from=builder /app/dist /etc/nginx/html/front
#надо потестить на вмке и на хосте пооткрывать и узнать с каким работает
COPY --from=builder /app/dist /etc/nginx/html/              
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY ssl.conf.j2 /etc/nginx/conf.d/ssl.conf
COPY test.local.crt /etc/ssl/test.local/test.local.crt
COPY test.local.key /etc/ssl/test.local/test.local.key
EXPOSE 80