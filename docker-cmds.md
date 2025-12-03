### Download an image from Docker Hub
```docker pull <image_name>```
### Example: docker pull nginx

### List all images stored locally
```docker images```

### Build an image from a Dockerfile in the current directory (.)
```docker build -t <my_image_name> .```

### Delete an image
```docker rmi <image_id_or_name>```

### Delete all dangling (unused/untagged) images
```docker image prune```

### Create and start a container (detached mode, port mapped)
docker run -d -p 8080:80 --name my-web-server nginx

### List RUNNING containers
docker ps

### List ALL containers (running and stopped)
docker ps -a

### Lifecycle management
docker stop <container_id>
docker start <container_id>
docker rm <container_id>   ### Must be stopped first

### View logs (essential for debugging)
docker logs -f <container_id>

### Open a shell inside a running container
docker exec -it <container_id> bash
### Note: Use 'sh' for Alpine Linux images

### Create a persistent volume
docker volume create <volume_name>

### List volumes
docker volume ls

### Inspect volume (find physical location on host)
docker volume inspect <volume_name>

### Delete a volume (Data will be lost)
docker volume rm <volume_name>

### Run container with volume attached
### Syntax: -v <volume_name>:<container_path>
docker run -d -v my_data:/var/lib/mysql mysql

### Create a custom bridge network
docker network create <network_name>

### List networks
docker network ls

### View network details (subnet, connected IPs)
docker network inspect <network_name>

### Connect/Disconnect running containers
docker network connect <network_name> <container_name>
docker network disconnect <network_name> <container_name>

### Run container attached to specific network
docker run -d --net <network_name> nginx

### Removes stopped containers, unused networks, and dangling images
docker system prune

### Include unused volumes in the cleanup
docker system prune --volumes

### Pulls image if missing, runs it, prints text, and exits
docker run hello-world

### 1. Create a custom network
docker network create tutorial-net

### 2. Run Server (attached to network)
docker run -d --name my-web-server --network tutorial-net nginx

### 3. Run Client (curl) to test connection
### Note: We use the container name 'my-web-server' as the URL
docker run --rm --network tutorial-net curlimages/curl http://my-web-server

### 1. Create volume
docker volume create my-website-data

### 2. Run container with mounted volume
docker run -d -p 8080:80 -v my-website-data:/usr/share/nginx/html --name persistent-web nginx

### 3. Modify data inside the volume
docker exec persistent-web bash -c "echo '<h1>Persistence Works!</h1>' > /usr/share/nginx/html/index.html"

### 4. Delete the container (Force remove)
docker rm -f persistent-web

### 5. Create NEW container with SAME volume
docker run -d -p 8080:80 -v my-website-data:/usr/share/nginx/html --name new-web nginx

### 6. Verify data persists
curl localhost:8080

### Mounts volume AND current directory, then tars the data
docker run --rm \
  -v my-website-data:/source_data \
  -v $(pwd):/backup_dir \
  ubuntu \
  tar cvf /backup_dir/backup.tar -C /source_data .

  ### 1. Destroy data
docker volume rm my-website-data

### 2. Create fresh volume
docker volume create restored-volume

### 3. Extract tar into new volume
docker run --rm \
  -v restored-volume:/target_data \
  -v $(pwd):/backup_dir \
  ubuntu \
  bash -c "tar xvf /backup_dir/backup.tar -C /target_data"

### 4. Verify content
docker run --rm -v restored-volume:/data ubuntu cat /data/index.html

docker rm -f my-web-server new-web
docker network rm tutorial-net
docker volume rm restored-volume
rm backup.tar
