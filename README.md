

# Introduction
In this README, you will get to know how I was able to deploy my Trading-Application. The trading-app ([https://github.com/saud-aslam/trading-app](https://github.com/saud-aslam/trading-app)) is an online stock trading simulation REST API which can be used to create an account which would allow account holder to buy and sell stocks from Investor Exchange i.e IEX. Traders can withdraw money and/or deposit money into their account. They can also view latest quotes of any stock directly from this application. This REST API can be used by front-end developers, mobile-app developers, and traders. The architecture used here is based on microservices concept which is implemented using SpringBoot, IEX API and PSQL database. The SpringBoot 
(https://github.com/davidmiquelf/trading-app)) to AWS cloud. First a manual deployment with Docker, then an automated one with Elastic Beanstalk and Jenkins.

The trading-app simulates an API for posting, browsing, buying, and selling market quotes. It runs on EC2 micro instances that each have access to the same PostgreSQL RDS (Amazon relational database service).

# Docker

With Docker, each EC2 instance pulls the docker images from Dockerhub or Github, then creates and runs the containers. They originally ran psql locally, but in later iterations they simply connected to the RDS.


There are two Docker files, one in the base folder of trading-app, and the other in the psql folder.

## trading-app

The app is dockerized by the following lines from  `/dll/run_docker_app.sh`:

```
docker build -t trading-app .

docker run -d \
--restart unless-stopped \
-e "PSQL_URL=$PSQL_URL" \
-e "PSQL_USER=$PSQL_USER" \
-e "PSQL_PASSWORD=$PSQL_PASSWORD" \
-e "IEX_PUB_TOKEN=$IEX_PUB_TOKEN" \
--name trading-app \
--network trading-net \
-p 8080:8080 -t trading-app

```

This command creates a trading-app image from the Dockerfile, then runs the container with specific environment variables.

## jrvs-psql

The psql database is dockerized by similar lines in the script  `/dll/run_docker_psql.sh`:

```
cd ../psql

docker build -t jrvs-psql .

docker run --name jrvs-psql \
--restart unless-stopped \
-e "POSTGRES_PASSWORD=$PSQL_PASSWORD" \
-e POSTGRES_DB=jrvstrading \
-e "POSTGRES_USER=$PSQL_USER" \
--network trading-net \
-d -p 5432:5432 jrvs-psql

```

The Dockerfile tells the image to inject the commands in  `/psql/sql_dll/schema.sql`  after creating the jrvstrading database, so the database is fully ready after being dockerized.

# AWS Cloud

I did a number of small experiments on AWS to deploy my project to the cloud, here is a summary of what I did:

-   Now that I had docker images and scripts to dockerize my application, I could quickly set up my application on an Amazon EC2 instance.
-   I created an image of that instance so that I could launch copies of it from a launch template.
-   Rather than running jrvs-psql on each instance, I set up a single instance to run jrvs-psql, and allowed the other instances in the VPC (virtual private cloud) to access it.
-   Just to see if I could, I deleted my psql instance and used Amazon RDS to run an auto-generated psql database.
-   I set up a load balancer and an autoscaling group to automatically launch/terminate instances depending on server load.



# Jenkins and Elastic Beanstalk

The problem with the above approach is that it took a while to set it up, and updating my project way too time consuming. If I wanted to use a newer version of my app, I basically needed to log in to each instance and pull the latest docker image.  
Luckliy, Elastic Beanstalk (EB) can fully automate the process. After setting up an EB project with the desired environment variables and port forwards, I can simply upload a jar file of the latest version of my trading-app to have it run on all the automatically generated instances. Then, whenever I want to update my project I simply upload a new jar file.

But there are two problems to consider:

1.  In a production environment I don't want to upload a jar file to find out it crashed all my servers.
2.  I don't want every developer logging in to AWS and uploading their own jar files onto the same project.

The first problem is easy: I created two EB projects -- tradingApp-dev and tradingApp-prod -- that way I could test my new jar files on the dev servers first.

For the second problem, I used Jenkins: I made a new EC2 instance to host a Jenkins server behind an NGINX reverse proxy. I set up Jenkins to listen to the project's GitHub repo, pull new commits, build new jar files, then push them to EB. I set it up to listen to the dev branch for tradingApp-dev, and the master branch for tradingApp-prod.


  <img src="src/assets/images/docker.png" alt="docker"></p>
    <img src="src/assets/images/trading-aws.png" alt="aws"></p>

  <img src="src/assets/images/Jenkins.png" alt="jenkins"></p>

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkxMTYzNjU0MiwyMDY4MjMxOTM3LC0zOT
QzMTc4MTBdfQ==
-->