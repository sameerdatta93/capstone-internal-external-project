From node:18
WORKDIR /app
COPY internal/package*.json ./
RUN npm install
COPY internal/. ./
CMD ["npm", "start"]
