# ---------- Stage 1: Build the React app ----------
FROM node:18 AS builder

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm install postcss@8.4.21 postcss-safe-parser@6.0.0 --legacy-peer-deps
RUN npm install
RUN npm install chokidar@3.5.3

COPY . .

ENV NODE_OPTIONS=--openssl-legacy-provider

RUN npm run build

# ---------- Stage 2: Serve with nginx ----------
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
