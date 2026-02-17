FROM node:20-alpine AS build

WORKDIR /app
COPY . .

RUN npm ci \
    && npx @angular/cli build --configuration production

FROM caddy:2-alpine

WORKDIR /app

COPY --from=build /app/dist/microcrm/browser /usr/share/caddy
COPY Caddyfile /etc/caddy/Caddyfile

EXPOSE 80
EXPOSE 443
