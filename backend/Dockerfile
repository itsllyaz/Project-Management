FROM node:18-alpine

# Install OpenSSL libraries natively inside Alpine Linux
RUN apk add --no-cache openssl openssl-dev

WORKDIR /app

COPY package*.json ./

# 1. Install dependencies but ignore the postinstall script for now
RUN npm install --ignore-scripts

# 2. Copy Prisma schema so Docker can see it
COPY prisma ./prisma/

# 3. Generate the OS-native Prisma client binaries 
RUN npx prisma generate

# 4. Copy the rest of your application code (including your scripts/ folder)
COPY . .

EXPOSE 5000
CMD ["npm", "run", "dev"]
