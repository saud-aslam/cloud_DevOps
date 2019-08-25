

# Introduction
In this README, you will get to know how I was able to deploy my Trading-Application on Amazon Cloud Server. The trading-app ([https://github.com/saud-aslam/trading-app](https://github.com/saud-aslam/trading-app)) is an online stock trading simulation REST API which can be used to create an account which would allow account holder to buy and sell stocks from Investor Exchange i.e IEX. Traders can withdraw money and/or deposit money into their account. They can also view latest quotes of any stock directly from this application. This REST API can be used by front-end developers, mobile-app developers, and traders. The architecture used here is based on microservices concept which is implemented using SpringBoot, IEX API and PSQL database. The SpringBoot controller embeds Apache Tomcat which binds our application with HTTP requests.

The deployment of this application to cloud was done on Amazon Web Services (AWS). But before that we dockerized our trading-application so that we can load our application containers on remote servers (EC2 instances). We, then, did manual deployment on AWS cloud. After manually deploying our application, we used Elastic Beanstalk to automatically manage our application on Cloud. Lastly, we used CI/CD pipeline through Jenkins, to further automate our deployment process.


# Dockerize Trading Application

The aim was to deploy the application on EC2 instances by running docker containers of our application on it. Since our application also runs Postgresql database, we need an extra container in addition of the parent application container. Both of these containers are independent to each other but we need them to communicate. For that a network bridge is created in docker. 
#### DockerFiles
There are two Dockerfiles each for each containers. These dockerfiles have commands in them which pulls docker images from DockerHub and creates local images in our docker and then when we run our local images, we get our containers. The trading-app dockerfile loads two images from DockerHub: ***Maven*** image which is used to build and package the source code by following pom.xml; ***openjdk:8-alpine*** which is use to run our java application; ***postgres*** which is used to run postgres database and  also build all tables based on the schema which is provided under *trading_ddl* folder.






## Commands to dockerize trading-app

```
sudo docker network create --driver bridge trading-net .

```
```
sudo docker build -t trading-app .

```
```
sudo docker run \
-e "PSQL_URL=jdbc:postgresql://psql:5432/jrvstrading" \
-e "PSQL_USER=postgres" \
-e 'PSQL_PASSWORD=password' \
-e "IEX_PUB_TOKEN=YOUR_TOKEN" \
--network trading-net \
-p 8080:8080 -t trading-app

```

## Commands to dockerize jrvs-psql


```
cd psql

sudo docker build -t jrvs-psql .

sudo docker run --name jrvs-psql \
-e "POSTGRES_PASSWORD=$PSQL_PASSWORD" \
-e POSTGRES_DB=jrvstrading \
-e "POSTGRES_USER=$PSQL_USER" \
--network trading-net \
-d -p 5432:5432 jrvs-psql

```

This completes the process of dockerizing our application. The diagram below shows the process.
 <p align="center">
 <img src="src/assets/images/docker.png" alt="docker"></p>
  


# Manual Deployment on AWS cloud

To deploy the application on AWS cloud, I did some configuration on Amazon Web services platform as follows:
- Provison a EC2 instance.
- Create a security group with TCP 8080 protocol.
- Launch the instance.

Once the instance is running, it was time to configure the instance to run the trading-app. To do it :
- SSH to the ec2 server : `ssh -i ~/.ssh/pem/you_pem_key ec2-user@ec2_pub_ip`. Note: pem keys would be prompted to be generated when creating an instance. You need to store that key in .ssh directory of your local machine.
- Install docker 
- Start and Enable docker
- Install Git and Clone git trading-app repo : `git clone https://[https://github.com/saud-aslam/trading-app`
- docker build and run to dockerize the app and mentioned below.
-ssh to EC2 public ip from the laptop's browser as like this: http://{EC2_pubic_ip}:8080/swagger-ui.html

The below diagram shows how the the ec2 server resides the docker containers.
<p align="center">
 <img src="src/assets/images/docker1.png" alt="aws"></p>

So far in the manual deployment we used our local database which is running on EC2 instance. Now we went a little advance in the approach and rather than running local psql container in the instance, we used AWS database service i.e RDS. This allows us to run our trading-app container on ec2 instances and database container on a separate private subnet database server. Further we also created autoscaling group and a load balancer which would automatically scale-up and scale-down the instances based on the  http request volume on Load balancer.

The architecture is shown below.
<p align="center">
<img src="src/assets/images/trading-aws.png" alt="aws"></p>

# Elastic Beanstalk and Jenkins

The manual deployment of our trading-app which is performed above has a problem i.e It is time consuming to manually deploy the application on every instances whenever there is an update in our code. It will not be fun to docker build the application component on every instances.

This problem is solved by using Elastic Beanstalk (EB). By using this service of AWS, one can fully rely on EB to manage and upload the latest version of the code in all the running instances. All we need to do is to *mvn package* our source code and upload the jar/zip file. We also created two environment, namely, Development and Production.

Going further, we did more automation as we wanted to get rid of maven packaging and uploading our application jar/zip file every time there is a change in the code. To achieve this, we created a CI/CD pipeline by using Jenkins. Basically a Jenkinfiles is needed by Jenkins which manages this pipeline process. Whenever there is a new commit in the github repository, the jenkin server sitting in a newly created EC2 instance



In the last method all components had to be configured manually and every code update had to be deployed manually on every running instance, which is a lot of work. AWS provids the service Elastic Beanstalk to make this process easier. Elastic Beanstalk requires one setup and then it automatically handles the deployment, from capacity provisioning, load balancing, auto-scaling to application health monitoring. When there are changes in the code, the new code can be uploaded once and EB takes care of updating every EC2 instance.

The need to manually upload new code can be eliminated by using a CI/CD (Continuous Integration/Continuous Deployment) pipeline. In this project Jenkins was used to created such a pipeline. Jenkins monitors the trading_app GitHub repository. Whenever there is a new commit it will automatically compile, package, and deploy the new code to all running EC2 instances.

Jenkins requires a  [Jenkinsfiles](https://github.com/MiriamEA/trading_app/blob/master/Jenkinsfile)  that specifies how to handle the new code. The stage 'Build' packages the code using maven. The two stages 'Deploy-prod' and 'Deploy-dev' call script to deploy the code. The argument specifies which EB environment to use. This depends on the GitHub branch that was compiled. The actual deployment is done by the  [eb_deploy script](https://github.com/MiriamEA/trading_app/blob/master/scripts/eb_deploy.sh)  using the Elastic Beanstalk Command Line Interface.

This diagram shows the architecture of the deployment with EB and Jenkins.

 <p align="center">
<img src="src/assets/images/Jenkins.png" alt="jenkins" width="800" height="800"></p>


The problem with the above approach is that it took a while to set it up, and updating my project way too time consuming. If I wanted to use a newer version of my app, I basically needed to log in to each instance and pull the latest docker image.  
Luckliy, Elastic Beanstalk (EB) can fully automate the process. After setting up an EB project with the desired environment variables and port forwards, I can simply upload a jar file of the latest version of my trading-app to have it run on all the automatically generated instances. Then, whenever I want to update my project I simply upload a new jar file.

But there are two problems to consider:

1.  In a production environment I don't want to upload a jar file to find out it crashed all my servers.
2.  I don't want every developer logging in to AWS and uploading their own jar files onto the same project.

The first problem is easy: I created two EB projects -- tradingApp-dev and tradingApp-prod -- that way I could test my new jar files on the dev servers first.

For the second problem, I used Jenkins: I made a new EC2 instance to host a Jenkins server behind an NGINX reverse proxy. I set up Jenkins to listen to the project's GitHub repo, pull new commits, build new jar files, then push them to EB. I set it up to listen to the dev branch for tradingApp-dev, and the master branch for tradingApp-prod.


 


<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNDc2NDk4ODgsOTcxNzcxMjUxLDE4OD
I0MzI4MDUsLTIxNjkwNDU1NywtMTM0MDk0OTE0NCwxNDIxMDI4
ODAxLC00OTcxNTkzMjksMTYzMDc0MjIwLDQ3NDMxOTE5NCwtMz
A1MDE3OTgwLDE4MjcwMTM4MTEsLTE2MTc2MTg4MjIsMjA2ODIz
MTkzNywtMzk0MzE3ODEwXX0=
-->