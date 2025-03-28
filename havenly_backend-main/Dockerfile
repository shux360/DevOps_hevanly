FROM node:18-alpine

WORKDIR /app

# Copy all necessary files
COPY . .  

# Ensure package.json exists before running npm install
RUN npm install -g nodemon
RUN ls -l /app && npm install

EXPOSE 3001

CMD ["npm", "start"]
