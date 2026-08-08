# Copyright The OpenTelemetry Authors
# SPDX-License-Identifier: Apache-2.0

FROM node:22 AS builder

WORKDIR /app

COPY ./package*.json ./

RUN npm ci

COPY . ./

RUN npm run build

# -----------------------------------------------------------------------------

FROM node:22-alpine

WORKDIR /app

COPY ./package*.json ./

#RUN npm ci --only=production
RUN npm ci --omit=dev

COPY --from=builder /app/src/instrumentation.ts ./instrumentation.ts
COPY --from=builder /app/next.config.mjs ./next.config.mjs

COPY --from=builder /app/.next ./.next

EXPOSE 4000

CMD ["npm", "start"]
