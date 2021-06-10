# Setting up Mockaroo Enterprise

Mockaroo can be installed in your private AWS cloud as a docker image.  [Contact support for pricing.](https://mockaroo.com/comments/new)

## Requirements

Mockaroo requires the following cloud services:

* Amazon RDS (Postgres)
* Amazon S3
* Amazone SES (optional)

Mockaroo also requires redis, which can be installed via the redis docker image.

## Docker Image

The Mockaroo docker image is distributed via Amazon's Elastic Container Eegistry (ECR).  The URI for the image is:

```
622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise
```

## Setup

Mockaroo provides two types of services:

* app - The web front-end
* worker - Data generation workers - When we need to generate large volumes of data quickly, this is what we'll scale

I suggest running at least 2 separate docker containers: one for the app and one for workers.

### Pulling the image from Amazon ECR

Once the Mockaroo Enterprise repo has been shared with your AWS account, you'll need to ensure that the IAM user that you'll be using to pull the image has the required permissions.  These can be granted by applying the following IAM policy to a role to which the user has been assigned:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:DescribeImages"
      ],
      "Resource": "arn:aws:ecr:us-west-2:622045361486:repository/mockaroo-enterprise"
    }
  ]
 }
```

Once the user has been given the rights above, you can pull the Mockaroo Enterprise image using the following command:

```
aws ecr get-login-password --region us-west-2 | docker login --password-stdin --username AWS 622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise:latest
```

This will authenticate against the ECR repo with docker CLI by injecting an ECR token into docker from the output of the AWS CLI.  You should see the following output:

```
Login Succeeded
```

Then, pull the docker image:

```
docker pull 622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise:latest
```

### Amazon ElastiCache

Mockaroo uses Redis for caching content and queuing data generation jobs.  If you're using AWS the easiest way to provide Redis to Mockaroo is to create a Redis cluster using Amazon ElasticCache.

* Be sure to create the cluster in the same VPC where the EC2 instances running Mockaroo will reside.
* You can use a very small instance type as Mockaroo does not send much traffic to Redis. For example, cache.t2.small.

As an alternative, you can also run Redis natively or using docker if you don't want to use ElastiCache.

To run Redis as a docker image:

```
docker run -d --name redis -p 6379:6379 redis
```

### Amazon RDS

Create a postgres database on Amazon RDS called "mockaroo".  Remember the username and password.  You'll need to configure those as environment variables later.

### Amazon S3 Bucket

Create an Amazon S3 bucket.  You'll configure the name as an environment variable later.
In order for Mockaroo to upload files to this bucket, you can either configure AWS_ACCESS_KEY and AWS_SECRET_KEY environment variables (See "App Container" below), or assign an IAM role to the EC2 instance(s) on which Mockaroo run that can write to the S3 bucket.  Here a guide that describes how to do this: [Enable S3 access from EC2 by IAM role](https://cloud-gc.readthedocs.io/en/latest/chapter03_advanced-tutorial/iam-role.html)

### Amazon SES for Email

Mockaroo sends emails when users need to reset their password or have a file ready to download. We recommend you use Amazon SES to send emails. To set up SES:

1. Under Identity Management > Domains, add the domain on which Mockaroo will be hosted.  You will later set this as the MOCKAROO_DOMAIN environment variables.
2. Under Identity Management > Emails, add a "no-reply@(your domain)" email address.
3. Under Email Sending > SMTP Settings, create your SMTP credentials.  You will use these to set the MAIL_HOST, MAIL_USERNAME, and MAIL_PASSWORD environment variables.

### App Container

To run the app and api services, the first step is to create an app.env file...

```
# You'll need to configure these with your own values:
REDIS_URL=redis://(your redis hostname):6379/
REDIS_WORKERS=(The number of data generation workers you are running.  Can be from 1 to 32)
DB_USERNAME=(database username)
DB_PASSWORD=(database password)
DB_HOSTNAME=(hostname of your amazon rds instance)
AWS_ACCESS_KEY=(your aws key - alternatively you can omit this and grant mockaroo access to your bucket via IAM)
AWS_SECRET_KEY=(your aws secret - alternatively you can omit this and grant mockaroo access to your bucket via IAM)
S3_BUCKET=(the name of the S3 bucket assigned to mockaroo)
MOCKAROO_ADMIN_EMAIL=(an email address where errors and daily reports should be sent)
MOCKAROO_DOMAIN=(the domain name on which your hosting mockaroo)  
MOCKAROO_MAIL_FROM=(the email address used when Mockaroo sends automated emails, defaults to "no-reply@{MOCKAROO_DOMAIN}")
MAIL_HOST=(your SES email host, typically something like "email-smtp.us-west-2.amazonaws.com")
MAIL_USERNAME=(your SES email username)
MAIL_PASSWORD=(your SES email password)

# In most cases you can leave these unchanged:
RAILS_ENV=production
RACK_ENV=production
DB_ADAPTER=postgresql
DB_NAME=mockaroo
DB_PORT=5432
MAIL_PORT=587
PORT=3001
MOCKAROO_QUICK_DOWNLOAD_LIMIT=10000
MOCKAROO_ENTERPRISE=true
MOCKAROO_ALLOW_ANONYMOUS_ACCESS=true
MOCKAROO_ALLOW_PASSWORD_AUTH=true
MOCKAROO_API_REQUEST_LIMIT=100000
MOCKAROO_API_RECORD_LIMIT=5000
MOCKAROO_DEFAULT_PLAN=Free
MOCKAROO_SERVE_STATIC_ASSETS=true
MOCKAROO_USE_SSL=true
MOCKAROO_DEFAULT_PLAN=Enterprise
REDIS_CLIENT_CONNECTIONS=16
REDIS_CONCURRENCY=20
REDIS_SERVER_CONNECTIONS=18
MOCKAROO_WORKERS=web=1
```
... then, run following to initialize the database ...

```
docker run --env-file app.env mockaroo/mockaroo-enterprise rake db:create && rake db:schema:load && rake db:seed
```

Finally, run the following to start the mockaroo web app on port 8080 (or any port you like):

```
docker run -d --name mockaroo --env-file app.env -p 8080:80 mockaroo/mockaroo-enterprise
```

### Worker Container

To run the data generation workers, copy app.env to a new file called worker.env and replace this:

```
MOCKAROO_WORKERS=web=1
```

with this:

```
MOCKAROO_WORKERS=worker0=1,worker1=1,worker2=1,worker3=1,worker4=1,worker5=1,worker6=1,worker7=1
```

This will give you 8 concurrent data generation processes.  You can add more by adding additional workers `worker8=1, worker9=1`, etc... to `MOCKAROO_WORKERS` and increasing the following:

```
REDIS_CLIENT_CONNECTIONS=(2 x #workers)
REDIS_CONCURRENCY=(2 x #workers + 4)
REDIS_SERVER_CONNECTIONS=(2 x #workers + 2)
```

To start the worker container, run:

```
docker run -d --name worker --env-file worker.env mockaroo/mockaroo-enterprise
```

## Web Server

In order to serve traffic from Mockaroo securely, you need to put a web server in front of Mockaroo. Here we offer two choices:

- NGINX
- AWS API Gateway

Before continuing, ensure that you have created DNS A records for the domain on which you want to host Mockaroo.  For example,
if the domain you want to use is mockaroo.my-enterprise.com, create A records for the following domains that point to the IP address
on which Mockaroo is hosted:

```
mockaroo.my-enterprise.com
api.mockaroo.my-enterprise.com
my.api.mockaroo.my-enterprise.com
```

### NGINX

To install NGINX on Ubuntu 18, run:

```
sudo apt update
sudo apt install nginx
```

To configure NGINX, create a file called nginx.conf with the following contents:

```
upstream mockaroo {
  server localhost:3000;
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  
  # You will need to change each occurrence of "mockaroo.my-enterprise.com" below to your custom domain for mockaroo
  server_name mockaroo.my-enterprise.com;
  server_name api.mockaroo.my-enterprise.com;
  server_name my.api.mockaroo.my-enterprise.com;
  
  proxy_set_header Host $http_host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_redirect off;
  proxy_send_timeout 1000s;   # disable timeout - protects against complicated schemas timing out before flushing
  proxy_read_timeout 1000s;   # disable timeout - protects against complicated schemas timing out before flushing
  proxy_buffering off;        # enabled response streaming
  client_max_body_size 20M;
  keepalive_timeout 10;

  location / {
    proxy_pass http://mockaroo;
  }
}
```

Then, link the site and reload nginx:

```
sudo ln -s $(pwd)/nginx.conf /etc/nginx/sites-enabled/mockaroo
sudo systemctl reload nginx
```

#### TLS

You can use certbot to provision a free TLS certificate for Mockaroo. Installation steps may differ depending on your OS.  For Ubuntu 18, run:

```
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```

Then, when prompted, generate a certificate for all of the sites listed.

To ensure that certs are automatically renewed every 90 days, add a cron task by running:

```
crontab -e
```

And then pasting the following:

```
43 6 * * * certbot renew --post-hook "sudo systemctl restart nginx"
```

### AWS API Gateway

API Gateway can be used in place of NGINX. You can also use Amazon Certificate Manager to provision the TLS certificate.  When configuring API Gateway, be sure to set up services for all 3 custom domains:

mockaroo.my-enterprise.com
api.mockaroo.my-enterprise.com
my.api.mockaroo.my-enterprise.com

And use a single catch-all route to forward requests on any method to the EC2 instance running Mockaroo:

/{proxy+} => /{proxy}

## Upgrades

When an upgrade is available, grab the latest mockaroo-enterprise docker image, then run:

```
docker run app.env mockaroo/mockaroo-enterprise:(version) rake db:migrate
```

Then, redeploy your app and worker containers

