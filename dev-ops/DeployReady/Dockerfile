FROM node:22-bookworm-slim

ENV NODE_ENV=production
ENV PORT=3000

WORKDIR /usr/src/app

COPY app/package*.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY --chown=node:node app/index.js ./

USER node

EXPOSE 3000

CMD ["node", "index.js"]
