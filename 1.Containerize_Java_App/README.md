![project_diagram](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/diagram.png)

This project is a hands-on of conterizating an existing multi-tier application.  

Steps Taken to Containerize Existing App  
1.Find the right base images for Nginx,RabbitMQ,Tomcat,Memcached,and MySQL  
2.Customize Dockerfiles to build images  
3.Write Docker compose file to test the application  
4.Push images to Dockerhub once you have ensured everything working  

You can complete this project using Vagrant, but I chose to do this in AWS. This tutorial will based on hosting the application within an EC2.  

First, we would need to create an EC2, I used a t2.small(this was the recommanded as t2.micro might be too slow this project), Redhat Enterprise Linux 9. If you are not using a free tier account, I ran this EC2 somewhere between 6-10 hours it costed less than a dollar.  

![project_diagram](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/EC2.png)

You will also need to set up a security group but we'll get to that in the later section. For now let's just enable SSH (port 22).  

Once your EC2 is up an running, you can SSH into it from your terminal.Locate the path/directory where your SSH is located I use windows so from the Powershell I ran:

```ssh -i "<your-pem-file>.pem" ec2-user@<public-ip>.compute-1.amazonaws.com```

Once your in your EC2, you want to glone the Github Project Repo and checkout to the correct branch. I chose to do this under the /home directory, I ran:  

```git clone <URL>```  
```git checkout <branch_name>```  
![project_diagram](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/git_clone.png)

Alright so we're half way there.  

Now you will need to install the docker-engine. I highly recommand reading up/watching a video on what the docker engine is, and how it functions. You can follow this tutorial on how to install docker-engine here: ![docker-engine-installation](https://docs.docker.com/engine/install).

Confirm the installation with ```sudo systemctl status docker``` or ```docker run hello-world```. The second command will run the hello world image.  
![docker-confirm](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker-confirm.png)

We're almost done, now the last part before we can begin is setting up ap Dockerhub account and creating the repositories. We're going to be building three custome images so we would need to create repositories for those.  
 ![docker-hub-sign=up](https://docs.docker.com/docker-id/).
![docker-hub](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker_hub.png)  

Finally, we can start writing docker files.  

The first Dockerfile we will be writing is for the application image and here we will be using Tomcat to serve these files.In order to do this we need to build an artifact.  
To build an artifact we would need to run ```mvn install``` which relies on``openjdk``.For image we will write out two images, one to build the artifact and then second to copy the artifact to the image Tomcat.

The first part of this Dockerfile will use openjdk to build a new image ```build_image```  
It will run ```mvn install``` which will build an artifact, (you can try doing this manually to see how it works)
this artifact will generated into a directory called target, which we are going to copy into the default Tomcat directory which is where Tomcat expects to find default web application files. In this case we are replacing with our own application artifact.  


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
Here we have a mysql file that needs to be copied into the image.We need to set a password and databe name, for now I gave it a random value.  
The ADD argument here will copy out sql file into the directory ```docker-entrypoint.initdb/```, this directory information 
can be found on the dockerhub mysql image's page.  

```
FROM mysql:8.0.33

ENV MYSQL_ROOT_PASSWORD="12345"
ENV MYSQL_DATABSE = "accounts"

ADD db_backup.sql docker-entrypoint-initdb/db_backup.sql  

```  


The application will be listening on port 80 bindind to a container called vproapp on port 8080.  
![nginx_our_file](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/nginx_conf.png)
  
The last Dockerfile we will be building is for nginx. We will be using the image from dockerhub but we need to change the configuration for this. The default configuration lives in ```/etc/nginx/conf.d/default.conf```(this information can be located on dockerhub for the nginx image). We will erase this default file and replace it with ours.  


```
FROM nginx

RUN rm -f /etc/nginx/conf.d/default.conf

COPY nginvproapp.conf /etc/nginx/conf.d/vproapp.conf
```


Now, before we commit these files to dockerhub we need to test them. We will be writing out a docker compose file next.  


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

All information related to port numbers and passwords can be found form the source code in the directory ```/src/main/resources/application.properties```  
Image names should be the same as the repositories we created on Dockerhub

Memcached and RabbitMQ do not to be customized, configurations related to them can be stated in the configuration files.

These volumes are docker managed and can be found in ```var/lib/docker/volume``` in most cases, this will create persistent data. This important in case your container crashes or needs to be restarted. This will ensure that there is no data loss.

Finally, let's navigate to the directory where all our confiuration files live and run ```docker compose build```.  

This will build all images 

![build_images](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/build_images.png)

```docker compose up -d``` will not run our containers, we could run this without build the images before hand and it would have still worked

```docker ps ``` will give you the an output of all running containers.
![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/containers.png)

Time to validate wheater or not the application is working. This set up is build on an EC2, so we will need the public IP of the EC2 and the port that app is being exposed on in this case port 80(nginx). 

```http://<public ip>:80```

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/public.ip.png)

At this point this will most likely not work because we havent configured our secuirty group rules.

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/SG.png)

After editing the Secuirty Group rules you should have an application that is up and running.

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/website.png)

TA DA!!!

Time to push these to Dockerhub

```docker push <account_name>/<repository_name>``` , you might need to run ```docker login``` before being able to push the image to your repository.

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker_push.png)

![containers](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker_hub_images.png)

Nice,now we have an image that is public to everyone and can be used whenever they like!

THE END!






















