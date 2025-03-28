FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install -g nodemon
RUN npm install

COPY . .

CMD ["npm", "start"]