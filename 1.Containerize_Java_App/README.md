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
To build an artifact we would need to run ```mvn install``` which relies on``openjdk``.For image we will write out two images, one to build the artifact and then second to copy the artifact to the image Tomcat  
```
FROM openjdk:11 as build_image
RUN yum update && install maven -y
RUN git clone https://github.com/devopshydclub/vprofile-project
RUN cd vprofile-project && git checkout docker && mvn install

FROM tomcat:9-jre11
RUN rm -rf /usr/local/tomcat/webapps*
COPY --from=build_image vprofile-project/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh","run]

```  
The first part of this Dockerfile will use openjdk to build a new image ```build_image```  
It will run ```mvn install``` which will build an artifact, (you can try doing this manually to see how it works)
this artifact will generated into a directory called target, which we are going to copy into the default Tomcat directory which is where Tomcat expects to find default web application files. In this case we are replacing with our own application artifact.  

Now, let's build the database. For this section we will be using MySQL. 
![docker-hub](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/app_docker.png)
Here we have a mysql file that needs to be copied into the image

```FROM mysql:8.0.33

ENV MYSQL_ROOT_PASSWORD="12345"
ENV MYSQL_DATABSE = "accounts"

ADD db_backup.sql docker-entrypoint-initdb/db_backup.sql ```

We need to set a password and databe name, for now I gave it a random value.  
The ADD argument here will copy out sql file into the directory ```docker-entrypoint.initdb/```, this directory information 
Lecture thumbnail
can be found on the dockerhub mysql image's page.

The last Dockerfile we will be building is for nginx. We will be using the image from dockerhub but we need to change the configuration for this. The default configuration lives in ```/etc/nginx/conf.d/default.conf```(this information can be located on dockerhub for the nginx image). We will erase this default file and replace it with ours.
![nginx_our_file](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/nginx_conf.png)

The application will be listening on port 80 bindind to a container called vproapp on port 8080.

```
FROM nginx

RUN rm -f /etc/nginx/conf.d/default.conf

COPY nginvproapp.conf /etc/nginx/conf.d/vproapp.conf

```

Now, before we commit these files to dockerhub we need to test them. We will be writing out a docker compose file next.
























