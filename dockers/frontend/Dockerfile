FROM nginx:1.25-alpine

EXPOSE 80

# Overwrite Nginx's default landing page with a dummy string
RUN echo "<h1>Welcome to the VoteApp Frontend!</h1>" > /usr/share/nginx/html/index.html

CMD ["nginx", "-g", "daemon off;"]