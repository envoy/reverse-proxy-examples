# About

This example uses `nginx` to proxy http requests from an ec2 insance on a public subnet (network) 
to another ec2 instance on a private subnet. 

*upsteam host*  ec2 instance running service nginx will proxy connections too
*proxy host*  ec2 instance running nginx


### files

```
├── Dockerfile # docker image used to run nginx
├── docker-compose.yml # docker container runtime settings
├── nginx.conf # nginx stanza to proxy requests to an upstream http service
└── nginx.env # enviroment variable used to template nginx.conf
```


# Prerequistes

- AWS creditianls & aws cli installed 
- VPC & subnets provisioned in an aws region


# Installing nginx on ec2

### Create an ec2 instance

Generate an ssh key in aws
```
aws ec2 create-key-pair --key-name http-nginx --query 'KeyMaterial' --output text > ~/.ssh/example.pem
```

Create a security group
```
aws ec2 create-security-group --group-name nginx-http  --description "nginx-http" --vpc-id <your vpc id here>
```

Add ingress for ssh
```
aws ec2 authorize-security-group-ingress --group-id <nginx-http sg id>  --protocol tcp --port 22 --cidr $(curl -s  ifconfig.io)/32
```

Deploy an ec2 instance
```
aws ec2 run-instances --image-id ami-0076b30a832c25ac4 --count 1 --instance-type t2.micro --key-name http-nginx --security-group-ids <nginx-http >  --subnet-id subnet-78da0953
```

Setup sg ingress from proxy host to upstream host
```
aws ec2 authorize-security-group-ingress --group-id <upsteam host sg-id>   --protocol tcp --port 80 --source-group <proxy host sg-id> 
```



### Install example nginx proxy on ec2

Copy over nginx/conf and files
```
scp -r  -i "~/.ssh/example.pem" http_ec2   ubuntu@ec2-54-205-29-206.compute-1.amazonaws.com:/home/ubuntu/
```

SSH onto host and install dependencies
```
sudo apt-get update && sudo apt-get install -y docker docker-compose netcat
```

Confirm the network route is establised between proxy host and upstream host
```
nc -vz <upsteam host private ip> 80
```

navigate to your nginx conf settings
```
cd /home/ubuntu/http_ec2 
```

seed  enviroment variables (nginx.env)
```
tee -a nginx.env << END
HTTP_SERVICE_PORT=80
HTTP_SERVICE=<upsteam host private ip>
END
```

start the nginx container
```
sudo docker-compose up -d
sudo docker ps # confirm the container is running
sudo docker logs nginx-http-reverse-proxy # check nginx logs
```

use an http client/downloader to test your proxy
```
wget localhost:80
```

### notes

Your `nginx.conf` may differ based on the version you run, your upstream service's setup and several other factor.
( ie Some http servers may require certain request headers be appended to nginx) 


For an http service listening on port `80` you may need to omit a port suffix `$HTTP_SERVICE_PORT`
```
    location / {
       proxy_pass http://$HTTP_SERVICE;
    }
 ```

For an http service listening on port an arbitrary port (8080) you may need to include a port suffix `$HTTP_SERVICE_PORT`
```
    location / {
       proxy_pass http://$HTTP_SERVICE:$HTTP_SERVICE_PORT;
       # proxy_pass http://172.31.53.136:8080
    }
 ```


### versions used
- 20.04.1-Ubuntu
- Docker version 20.10.12
- docker-compose version 1.25.0


# Resources
- https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2.html
- https://cloud-images.ubuntu.com/locator/ec2/
