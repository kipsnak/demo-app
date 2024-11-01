#
# Builder
#
FROM docker.io/node:lts-alpine AS builder
LABEL stage="builder"

# prepare workdir
RUN mkdir -p /app
WORKDIR /app

# install dependencies
COPY package*.json .
RUN npm install --omit dev

# build app
COPY . .
RUN npm run build

#
# Runtime
#
FROM docker.io/nginx:alpine
LABEL stage="runtime"

COPY --from=builder /app/build /usr/share/nginx/html/