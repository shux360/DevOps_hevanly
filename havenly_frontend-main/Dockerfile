FROM node:18-alpine

# Set the working directory inside the container
# WORKDIR /src/App.jsx
WORKDIR /app 

# Copy package.json and package-lock.json to the working directory

COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the application code
COPY . .

# Expose the port the app runs on
# EXPOSE 3000
EXPOSE 5173

# Command to run the application
CMD ["npm", "run", "dev"]