# ---- Build Stage ----
FROM node:22.22.0-alpine3.22 AS builder

# Set the working directory in the container to /console
WORKDIR /console

# Create the pnpm directory and set the PNPM_HOME environment variable
RUN mkdir -p ~/.pnpm
ENV PNPM_HOME=/root/.pnpm

# Set environment
ARG ENVIRONMENT=production

# Attribution
ARG DISABLE_FLEETBASE_ATTRIBUTION=false
ENV DISABLE_FLEETBASE_ATTRIBUTION=$DISABLE_FLEETBASE_ATTRIBUTION

# Add the pnpm global bin to the PATH
ENV PATH=/root/.pnpm/bin:$PATH

# Copy package metadata first for better dependency caching
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./

# Copy over .npmrc if applicable
COPY .npmr[c] ./

# Install pnpm with retries for transient registry/network interruptions.
# ember-cli is already a project devDependency and is invoked from node_modules
# by pnpm build, so installing a second global copy is unnecessary.
RUN npm config set fetch-retries 5 \
    && npm config set fetch-retry-mintimeout 20000 \
    && npm config set fetch-retry-maxtimeout 120000 \
    && npm config set fetch-timeout 600000 \
    && npm install -g pnpm@11.0.9 --no-audit --no-fund --prefer-online

# Install git
RUN apk update && apk add git openssh-client

# Trust GitHub's RSA host key
RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> /root/.ssh/known_hosts

# Install app dependencies
RUN pnpm install --frozen-lockfile

# Copy the console directory contents into the container at /console
COPY . .

# Build the application
RUN pnpm build --environment $ENVIRONMENT

# ---- Serve Stage ----
FROM nginx:alpine

# Copy the built app to our served directory
COPY --from=builder /console/dist /usr/share/nginx/html

# Expose the port nginx is bound to
EXPOSE 4200

# Use custom nginx.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Start Nginx server
CMD ["nginx", "-g", "daemon off;"]
