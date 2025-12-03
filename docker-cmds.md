# Download an image from Docker Hub
docker pull <image_name>
# Example: docker pull nginx

# List all images stored locally on your machine
docker images

# Build an image from a Dockerfile in the current directory (.)
# -t assigns a name (tag) to the image
docker build -t <my_image_name> .

# Delete an image from your disk
docker rmi <image_id_or_name>

# Delete all unused images (dangling images)
docker image prune



# Create and start a container
# -d: Run in background (detached)
# -p: Map ports (HostPort:ContainerPort)
# --name: Assign a custom name
docker run -d -p 8080:80 --name my-web-server nginx

# List currently RUNNING containers
docker ps

# List ALL containers (running and stopped)
docker ps -a

# Stop a running container
docker stop <container_id_or_name>

# Start a stopped container
docker start <container_id_or_name>

# Delete a container (must be stopped first)
docker rm <container_id_or_name>

# View the output/logs of a container (Critical for debugging)
docker logs <container_id_or_name>

# Follow the logs in real-time (like tail -f)
docker logs -f <container_id_or_name>

# Open a command prompt (shell) inside a running container
docker exec -it <container_id_or_name> bash
# Note: Use 'sh' instead of 'bash' if the image is Alpine Linux






# Create a new volume
docker volume create <volume_name>

# List all volumes
docker volume ls

# View volume details (find where data is stored on host)
docker volume inspect <volume_name>

# Delete a volume (WARNING: This deletes the data)
docker volume rm <volume_name>

# Usage Example: Running a container with a volume attached
# Syntax: -v <volume_name>:<path_inside_container>
docker run -d -v my_data:/var/lib/mysql mysql






# Create a custom bridge network
docker network create <network_name>

# List all networks
docker network ls

# View network details (subnet, connected containers, IPs)
docker network inspect <network_name>

# Connect a running container to a specific network
docker network connect <network_name> <container_name>

# Disconnect a container from a network
docker network disconnect <network_name> <container_name>

# Delete a network
docker network rm <network_name>

# Usage Example: Run a container inside a specific network
docker run -d --net <network_name> nginx






# Removes:
# 1. Stopped containers
# 2. Unused networks
# 3. Dangling images (images with no name/tag)
# 4. Build cache
docker system prune

# To also include unused volumes, add the --volumes flag:
docker system prune --volumes
