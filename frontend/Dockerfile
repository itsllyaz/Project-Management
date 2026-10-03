# Build stage
FROM node:20-alpine as build   # use Node 20 to match vite/react-router requirements
WORKDIR /app

# Copy only package.json + lock first
COPY package*.json ./
RUN npm install

# Copy the rest of the source
COPY . .

# Build the app
RUN npm run build

# Serve with nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
