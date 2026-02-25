# Use a lightweight Nginx image
FROM nginx:alpine

# 🛠️ Patch the libpng vulnerability caught by Trivy
RUN apk upgrade --no-cache libpng

# Remove the default Nginx welcome page
RUN rm -rf /usr/share/nginx/html/*

# Copy the application's static content
COPY . /usr/share/nginx/html/

# Expose port 80 for HTTP traffic
EXPOSE 80

# Command to start Nginx in the foreground
CMD ["nginx", "-g", "daemon off;"]
