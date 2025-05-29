From node:18
WORKDIR /app
COPY external/package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "start"]
