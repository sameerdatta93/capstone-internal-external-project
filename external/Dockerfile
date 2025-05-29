From node:18
WORKDIR /app
COPY external/package*.json ./
RUN npm install
COPY external/. ./
CMD ["npm", "start"]
