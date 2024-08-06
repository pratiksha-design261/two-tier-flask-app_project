# Two Tier application execution steps

1. Clone code to local received from dev team
```sh
git clone <url>
```
2.  Create DockerFile
3.  Create new network
```sh
docker network create twotire
```
> as type of network is not mentioned it will generate with bridge as its default network

3.  Run available image for sql and also add network for communication between msql and flask, env variable as per application  and also add volume to persist data 
```sh
docker run -d --name mysql_data_with_vloume -v mysql_volume_1:/var/lib/mysql --network twotire -e MYSQL_DATABASE=devops -e MYSQL_ROOT_PASSWORD=root mysql:5.7
```
> [-d](): Run container in demon/background mode
>
> [--name mysql_data_with_vloume](): it sets the name for generated container
>
> [-v mysql_volume_1:/var/lib/mysql](): it will create mysql_volume_1 named volume and mount to  /var/lib/mysql location
>
> [--network twotire](): It will set user-generated bridge network for communication between db and flask
>
> [-e MYSQL_DATABASE=devops](): -e can be while setting env varibale -e <variable name>=<value>
>
> [mysql:5.7](): <image name>:tag


4. Build docker image for flask
```sh
docker build -t two_tire_app:latest .
```

5. Run docker image
```sh
docker run -d --network twotire --name two-tire-application -p 5000:5000 -e MYSQL_HOST=mysql_data_with_vloume -e MYSQL_USER=root -e MYSQL_PASSWORD=root -e MYSQL_DB=devops two_tire_app:latest
```
`Note`: the element explain above is not explained again 
>[-p 5000:5000](): It will expose port 5000 to run application 

6. EC2 UI , In the security group edit inbound rule for port 5000
7. run <ip address>:5000 in browser
8. To validate data entered in DB 
   A. use execute cmd to run MySQL in local
     ```sh
     docker exec -it mysql_data_with_vloume bash
     ```
    >[-it mysql_data_with_vloume]():add docker container name for mysql
 
     B. Switch bash CLI to Mysql
     ```sh
     mysql -u <uname> -p
     ```
     C. switch to our application database 
     ```sh
     use <database name>;
     ```
     D. List of tabels
     ```sh
     show tabels;
     ```
     E. Get all entered data from the tabel 
     ```sh
     select * from <tabel name>;
     ```


##### Steps to use docker-compose for the same project 
Step 1: Install docker-compose 
   ```sh
   sudo apt-get install docker-compose-v2
   ```
   `note`: if docker-compose older version is available uninstall it with below cmd 
   ```sh
   sudo apt purge docker-compose
   ```
Step 2: Create docker-compose.yml file 
 ```sh
version: '3.9'
services:

  flask-app:
    container_name: 'flask-python-application'
    build:
      context: .
    ports:
      - '5000:5000'
    environment:
      MYSQL_HOST: 'mysql-db'
      MYSQL_USER: 'root'
      MYSQL_PASSWORD: 'root'
      MYSQL_DB: 'devops'
    depends_on:
      mysql-db:
        condition: service_healthy

  mysql-db:
    container_name: 'mysql-db'
    image: 'mysql:5.7'
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 'root'
      MYSQL_DATABASE: 'devops'
    volumes:
      - type: volume
        source: mysql-volume-data
        target: /var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  mysql-volume-data:
```

Step 3: Run the docker-compose file
```sh
docker compose up -d
```
`note`: 1. if we want to make any changes in the docker-compose file previous running container must be stopped to reflect new changes 
```sh
docker compose down
```
`note`: 2. if any of container fail then check docker logs 
```sh
docker logs <container id>
```


# Multi-stage docker build 
1. It uses to optimize the image 
2. In it file can be divided into stages after stage 1, the image can be compressed and use in the next stage, generally in 1st stage we include the base image and all run cmd and then get its compressed version can be used in stage 2 which only has last CMD cmd 
For Ex.
```sh
#----------------------------stage 1 started -----------------------------------------
# Use an official Python runtime as the base image
FROM python:3.9-slim AS Backend_flask

# Set the working directory in the container
WORKDIR /app

# install required packages for system
RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y gcc default-libmysqlclient-dev pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Copy the requirements file into the container
COPY requirements.txt .

# Install app dependencies
RUN pip install mysqlclient
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY . .

#------------------------stage 2 started ------------------
#Used compressed version of stage 1
FROM python:3.9-slim

#Set working directory for stage 2
WORKDIR /app

#Copy libraries
COPY --from=Backend_flask /usr/local/lib/python3.9/site-packages/ /usr/local/lib/python3.11/site-packages/

#Copy Src code from stage 1
COPY --from=Backend_flask /app /app

# Specify the command to run your application
CMD ["python", "app.py"]
```

3. run the build cmd with dockerfile 


# Entrypoint VS CMD
>[Entrypoint](): its same as CMD command but parameters cannot be overwritten
>[CMD](): Parameters can be overwritten , if anyone can pass parameter while building image 

# RUN VS CMD
>[RUN](): This is used in the intermediate layer of image
>[CMD](): this is used in end of image , this generally use to run app file 

# Docker Scout
1. Docker scout is used for vulnerabilities scan for image


##### Steps:
1. Create a folder 
```sh
mkdir -p $HOME/.docker/scout
```
2. Install using curl cmd
```sh
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh
```
3. log in with docker hub credentials
goto: dockerhub -> Account Setting -> PAT -> (Give name and select access) and generate 
-> the 2 cmd will shown, with that login in shell

4. run docker scout cmd 
```sh
docker scout cves two-tier-flask:latest
```
`note`: to view scan results in short 
```sh
docker scout quickview two-tier-flask:latest
```

# Docker push and pull
1. Log in docker credentials with PAT
2. make sure image name is with dockerhub uname if not make another copy of image with desired name
```sh
docker image tag <image name>:<tag> <uname>/<image name>:<tag>
```
3. push image
```sh
docker push pratiksha1999/two-tier-flask:latest
```
4. pull image
```sh
docker pull pratiksha1999/stockmanager
```

