FROM node:24-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev
COPY app.js .
EXPOSE 3001
USER node
CMD ["node", "app.js"]
