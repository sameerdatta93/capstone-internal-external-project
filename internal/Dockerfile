From node:18
WORKDIR /app
COPY internal/package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "start"]
