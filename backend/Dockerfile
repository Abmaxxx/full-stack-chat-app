FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

EXPOSE 5001

ENV NODE_ENV=production

CMD ["npm", "start"]
