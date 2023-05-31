# Use the latest Node.js LTS (Long-Term Support) version as the base image
FROM node:lts

# Set the working directory inside the container
WORKDIR /app

# Copy the Node.js notebook to the container
COPY stateManagement.nnb /app/

# Install nteract globally
RUN npm install -g nteract rxjs

# Expose a port for nteract server
EXPOSE 8888

# Set the command to execute the nteract server with the notebook
CMD ["nteract", "stateManagement.nnb"]
