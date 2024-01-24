![project_diagram](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/diagram.png)  

Here you will find: [Project Source](https://www.udemy.com/course/devopsprojects/)  
This project involves hands-on containerization of an existing multi-tier application.  

Steps Taken to Containerize Existing App:  
1. Find the right base images for Nginx, RabbitMQ, Tomcat, Memcached, and MySQL.  
2. Customize Dockerfiles to build images.  
3. Write a Docker-compose file to test the application.  
4. Push images to Docker Hub once you have ensured everything is working.  

You can complete this project using Vagrant, but I chose to do it in AWS. This tutorial is based on hosting the application within an EC2 instance.  

First, we need to create an EC2 instance. I used a t2.small (this was recommended, as t2.micro might be too slow for this project) with Red Hat Enterprise Linux 9. If you are not using a free-tier account, I ran this EC2 for somewhere between 6-10 hours, and it cost less than a dollar.  

![project_diagram](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/EC2.png)  

We will also need to set up a security group, but we'll get to that in the later section. For now, let's just enable SSH (port 22).  

Once our EC2 is up and running, we can SSH into it from your terminal. Locate the path/directory where your SSH key is located. I use Windows, so from PowerShell, I ran:  


```ssh -i "<your-pem-file>.pem" ec2-user@<public-ip>.compute-1.amazonaws.com```  

Once you're in your EC2, you want to clone the GitHub Project Repo and check out to the correct branch. I chose to do this under the /home directory. I ran:  

 ```git clone <URL>```    
```git checkout <branch_name>```    
![project_diagram](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/git_clone.png)  

Alright, so we're halfway there.  

Now we will need to install the Docker Engine. I highly recommend reading up or watching a video on what the Docker Engine is and how it functions. You can follow this tutorial on how to install the Docker Engine here:  
 [docker-engine-installation](https://docs.docker.com/engine/install).  

Confirm the installation with ```sudo systemctl status docker``` or ```docker run hello-world```. The second command will run the hello world image.  
![docker-confirm](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker-confirm.png)  

We're almost done; now the last part before we can begin is setting up a Docker Hub account and creating the repositories. We're going to be building three custom images, so we would need to create repositories for those.  
 
 [docker-hub-sign=up](https://docs.docker.com/docker-id/).  
![docker-hub](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker_hub.png)    

Finally, we can start writing Dockerfiles.  
  
The first Dockerfile we will be writing is for the application image, and here we will be using Tomcat to serve these files. In order to do this, we need to build an artifact.  
To build an artifact, we would need to run ```mvn install```, which relies on ```openjdk```. For the image, we will write out two stages, one to build the artifact and the second to copy the artifact to the Tomcat image.  

The first part of this Dockerfile will use openjdk to build a new image named `build_image`.  
It will run ```mvn install```, which will build an artifact (you can try doing this manually to see how it works).  
This artifact will be generated into a directory called `target`, which we are going to copy into the default Tomcat directory, where Tomcat expects to find default web application files. In this case, we are replacing it with our application artifact.  


```
FROM openjdk:11 as build_image  
RUN apt update && apt install maven -y (we use apt because openjdk is based on debian)  
RUN git clone https://github.com/devopshydclub/vprofile-project  
RUN cd vprofile-project && git checkout docker && mvn install  

FROM tomcat:9-jre11  
RUN rm -rf /usr/local/tomcat/webapps*  
COPY --from=build_image vprofile-project/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war  
EXPOSE 8080  
CMD ["catalina.sh","run]  

```  

Now, let's build the database. For this section we will be using MySQL.   
![docker-hub](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/app_docker.png)  

Here we have a MySQL file that needs to be copied into the image. We need to set a password and a database name; for now, I gave them random values.  
The ADD argument here will copy our SQL file into the directory ```docker-entrypoint.initdb/```. We can obtain this information from  [Docker Hub-mysql image](https://hub.docker.com/_/mysql) under ```Initializing a fresh instance```

```
FROM mysql:8.0.33  

ENV MYSQL_ROOT_PASSWORD="12345"  
ENV MYSQL_DATABSE = "accounts"  

ADD db_backup.sql docker-entrypoint-initdb/db_backup.sql    

```  


The application will be listening on port 80, binding to a container called 'vproapp' on port 8080   
![nginx_our_file](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/nginx_conf.png)

The last Dockerfile we will be building is for Nginx. We will be using the image from Docker Hub, but we need to change the configuration for this. The default configuration lives in ```/etc/nginx/conf.d/default.conf``` [Docker Hub-Nginx](https://hub.docker.com/_/nginx). We will erase this default file and replace it with our own.


```
FROM nginx  

RUN rm -f /etc/nginx/conf.d/default.conf  

COPY nginvproapp.conf /etc/nginx/conf.d/vproapp.conf  
```


Now, before we commit these files to Docker Hub, we need to test them. Next, we will be writing a Docker Compose file.  


```
version: '3.8'

services:
  vprodb:
    build:
      context: ./Docker-files/db
    image: ilknurw/vprofiledb
    container_name: vprodb
    ports:
      - "3306:3306"
    volumes:
      - vprodbdata:/var/lib/mysql 
    environment:
      - MYSQL_ROOT_PASS=vprodbpass


  vprocache01:       
    image: memcached
    ports:
      - "11211:11211"

  vpromq01:
    image: rabbitmq
    ports:
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest

  vproapp:
    build:
      context: ./Docker-files/app
    image: ilknurw/vprofileapp
    container_name: vproapp
    ports:
      - "8080:8080"
    volumes:
      - vproappdata:/usr/local/tomcat/webapps

  vproweb:
    build:
      context: ./Docker-files/web
    image: ilknurw/vprofileweb
    container_name: vproweb
    ports:
      - "80:80"

volumes:
  vprodbdata: {}
  vproappdata: {}
```  

![docker-compose](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/app-properties.png)  

All information related to port numbers and passwords can be found in the source code in the directory ```/src/main/resources/application.properties```. Image names should be the same as the repositories we created on Docker Hub.  

Memcached and RabbitMQ do not need to be customized; configurations related to them can be stated in the configuration files.  

These volumes are Docker-managed and can be found in var/lib/docker/volume. In most cases, this will create persistent data. This is important in case your container crashes or needs to be restarted, ensuring that there is no data loss.  

Finally, let's navigate to the directory where all our configuration files live and run ```docker compose build```.
![build_images](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/build_images.png)  

```docker-compose up -d``` will run our containers. We could run this without building the images beforehand, and it would still work.  

```docker ps ``` will give you the an output of all running containers.  
![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/contianers.png)  

Time to validate whether or not the application is working. This setup is built on an EC2, so we will need the public IP of the EC2 and the port that the app is being exposed on—in this case, port 80 (Nginx)  


```http://<public ip>:80```  

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/public.ip.png)  

At this point, this will most likely not work because we haven't configured our security group rules.  

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/SG.png)  

After editing the Security Group rules, you should have an application that is up and running.  

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/website.png)  

Ta-da!  
  
Time to push these to Dockerhub.  

```docker push <account_name>/<repository_name>``` , you might need to run ```docker login``` before being able to push the image to your repository.  

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker_push.png)  

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker_hub_images.png)  


Nice! Now we have an image that is public to everyone and can be used whenever they like!  

THE END  





















