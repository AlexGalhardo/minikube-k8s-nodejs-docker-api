FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm i
COPY index.mjs index.mjs
CMD ["npm", "start"]