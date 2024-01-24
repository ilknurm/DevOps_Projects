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

Now you will need to install the docker-engine. I highly recommand reading up/watching a video on what the docker engine is, and how it functions. You can follow this tutorial on how to install docker-engine here: ![docker-engine](https://docs.docker.com/engine/install).

Confirm the installation with ```sudo systemctl status docker``` or ```docker run hello-world```. The second command will run the hello world image.  
![docker-confirm](https://github.com/ilknurm/DevOps_Projects/blob/main/1.Containerize_Java_App/images/docker-confirm.png)

We're almost done, now the last part before we can begin is setting up ap Dockerhub account and creating the repositories. We're going to be building three custome images so we would need to create repositories for those.  
 ![docker-hub](https://docs.docker.com/docker-id/).
  ![docker-hub](https://docs.docker.com/docker_hub.png)  













