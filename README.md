## Deploying Docker App to AWS ECS

Watch the video!
[![Watch Step by Step on how to Deploying Docker App to AWS ECS](https://img.youtube.com/vi/YOUTUBE_VIDEO_ID_HERE/0.jpg)](https://youtu.be/vxrO7Vs4EPA)


In this POC, let's deploy a docker App (dotnet6) to AWS using ECS.
First what we need is:
1. Create a dotnet project
2. Create a dockerfile
3. Create an image and container in Docket to test.
4. Get our credentials from AWS
5. Send our Image to ECR
6. Create a Task Definition
7. Create a Cluster on ECS
8. Attach our Task to the Cluster
9. Open the port on EC2 - Security Group
10. Voilà


### Creating a dotnet6 project

After you have [dotnet](https://dotnet.microsoft.com/en-us/download) install in your machine.
Go to the folder where you want to create your project.
Run in terminal:
``` 
dotnet new webapi -n SimpleApi
cd SimpleApi/
dotnet run
```
 
 If everything is ok, you should be able to call the endpoint: localhost:[port]/WeatherForecast

 This is all that we need here. In this POC we want to focus on deployment, not on coding.

 ### Create a dockerfile

 Now we need to create a dockerfile that gives ECS the recipe of what we want to install in our container.

 The dockerfile:
 ```json
 #Build Stage

FROM mcr.microsoft.com/dotnet/sdk:6.0-focal AS Build
WORKDIR /source
COPY . .
RUN dotnet restore "SimpleApi.csproj" --disable-parallel
RUN dotnet publish "SimpleApi.csproj" -c release -o /app --no-restore

#Serve Stage
FROM mcr.microsoft.com/dotnet/sdk:6.0-focal 
WORKDIR /app
COPY --from=build /app ./

EXPOSE 5000

ENTRYPOINT [ "dotnet", "SimpleApi.dll" ]
 ```

We have our dockerfile but before we deploy to ECS, let's first build our docker container locally and see if everything is working well. 

``` 

//Running docker command to create the image
docker build --rm -t devotts/simpleapi:latest .

//To check if the image was created correctly
docker image ls | grep simpleapi

//Run the container
docker run --rm -p 5000:5000 devotts/simpleapi 
```

You will see that it will not work. This happen because ASP.NET Core applications only listen on the loopback interface as [this article](https://georgestocker.com/2017/01/31/fix-for-asp-net-core-docker-service-not-being-exposed-on-host/) from Geroge Stocker explains. 

So, in your `Program.cs` add this line:
```C#
    builder.WebHost.UseKestrel(serverOptions =>
    {    
        serverOptions.ListenAnyIP(5000);
    });
```

Now, build your image again and run it.
If everything is ok, you should be able to call the endpoint: localhost:5000/WeatherForecast

### Deploying Image to ECR Repository

(Before we go ahead, make sure you have AWS Cli installed and the [credentials configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)).

First, let's loggin in to ECR AWS to save our image in the ECR repository. (Elastic Container Registry is like dockerhub, where we host our images)

```bash

aws ecr get-login-password --region [AwsRegion] | docker login --username AWS --password-stdin [AwsAccountId].dkr.ecr.[AwsRegion].amazonaws.com
```

In [AWS ECR](https://us-east-1.console.aws.amazon.com/ecr/create-repository?region=us-east-1) create a new ECR Repository.
For this example I created a repository called `manual-test`. Copy the URI of the created repository.

Now we will push the docker image to ECR Repository

```bash
#tag 
docker tag poc-dev/simpleapi:latest [AccountId].dkr.ecr.us-east-1.amazonaws.com/manual-test:latest

#push
docker push [AccountId].dkr.ecr.us-east-1.amazonaws.com/manual-test:latest
```

In `AWS > ECR > Repositories > manual-test` you will be able to see a new Image tag called "latest".

### Creating the ECS

For us to create the ECS, hosting configuration using EC2, attach to the ECR repository and the container we will do it all at AWS Console. 
Follow the instructions on the video. 

[watch the video](https://youtu.be/vxrO7Vs4EPA)

In a nutshell we will:
1. [Create a Cluster](https://us-east-1.console.aws.amazon.com/ecs/home?region=us-east-1#/clusters/create/new)
2. [Create a Task Definition](https://us-east-1.console.aws.amazon.com/ecs/home?region=us-east-1#/taskDefinitions)
    2.1. Add a container getting the URI from the image `manual-test` stored at ECR
3. Goes back in your cluster and at Tasks tab click in *Run new Task* and associated your cluster with the Task that you just created.
4. Goes to your EC2 instance attached to yout Cluster, get the Public IPv4 DNS url and voilà, now you are able to access your API here -> http://[some-name].compute-1.amazonaws.com:8888/WeatherForecast

## Tips

Tip 1: If you choose EC2 as your hosting type in ECS, yout have to pay attention with CPU and Memory.
So, for example, when you create a Task make sure that the total memory of this tasks not exceed the memory available of the EC2 instance. Otherwise you will get an error like this: `Reasons : ["RESOURCE:MEMORY"] `.

Tip 2: If you choose EC2 as your hosting type in ECS, make sure you open the port 8888 in "EC2 > Security Groups > your-security-group > Edit inbound rules"

### Bonus: Let's automatize all of this proccess with  CloudFormation?

In the next video, I'll show how to code all of this with CloudFormation and CodeBuild. Stay tune.

