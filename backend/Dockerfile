FROM node:lts-alpine AS base
WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY src ./src

EXPOSE 3001

CMD ["npm", "start"]
