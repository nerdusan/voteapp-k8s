FROM python:3.11-slim

# Set a non-root working directory
WORKDIR /app

# Expose the port matching your deployment YAML
EXPOSE 8080

# Create a self-contained python script that serves a 200 OK text block on /ok
RUN echo 'from http.server import BaseHTTPRequestHandler, HTTPServer\n\
class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):\n\
    def do_GET(self):\n\
        if self.path == "/ok":\n\
            self.send_response(200)\n\
            self.send_header("Content-type", "text/plain")\n\
            self.end_headers()\n\
            self.wfile.write(b"OK")\n\
        else:\n\
            self.send_response(200)\n\
            self.send_header("Content-type", "text/html")\n\
            self.end_headers()\n\
            self.wfile.write(b"Hello from the true app layer!")\n\
\n\
server = HTTPServer(("0.0.0.0", 8080), SimpleHTTPRequestHandler)\n\
print("Server running on port 8080...")\n\
server.serve_forever()' > server.py

# Run the container entirely as a non-root user
USER 10001

CMD ["python", "server.py"]