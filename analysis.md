# PXL Vision's Case Study
The Case Study has been split into three parts:
## The First Part
### Dockerfile check,
The following `Dockerfile`, has four issues.
```yaml
FROM  python:3
COPY  magic_ball.py  /
RUN  pip  install  flask
CMD  [  "python",  "./magic_ball.py"  ]
```

 1. Using python:3
 After reviewing the code provided in the app I found that it is using this code snippet which is used in python 2 and will give an error in python3.
  ```python
 raw_input() 
 ``` 
 ```python
 print  "It is certain."
  ```
 so we are using python:2 and alpine which is a light version to reduce the image size.
 
 2. There is no working directory. However, no error will show up but it is recommended to use WORKDIR.
 3. The  COPY layer is missing the correct directory which should be "./"  so you can copy the script in the target directory.
 4. Installing flask is not required which is not needed in the project.
 

### The correct Dockerfile should be the following:

```docker
#Base Image
FROM  python:2.7-alpine
#Set the working directory
WORKDIR  /app
#Copy the application's files
COPY  magic_ball.py  .
#Run the Script
ENTRYPOINT  [  "python"  ]
CMD  [ "magic_ball.py"  ]
```
Then build the image using the command 
``docker build -t pxl:latest . `` considering the image called pxl with latest tag.
Then run the image using the following command:
``docker run -it --name pxl pxl:latest ``
then using "-it" so it will run the script in the interactive mode and the result will be 
``  Ask the Magic 8-ball a question: (Press Enter/Return to exit)``
*******************
## The second task:
### Add Nginx reverse proxy and serve on HTTPS:
I'm providing two solutions for this task. Namely, using Nginx docker container and using Nginx-Manager.

### First Solution:
The first solution is to make a docker container that serves as a Reverse Proxy using the following steps:
>For demonstration  I created a server on AWS it is an EC2 instance with the DNS name 
>- **ec2-3-145-217-255.us-east-2.compute.amazonaws.com** 
>- And has the following directory structure 
>- Docker-compose directory that has the WordPress composer file.
>- Proxy directory that has `includes` and `ssl` directories, `default.conf`, `Dockerfile` and proxy's `docker-composer.yml`.

1- we need to set up and configure a reverse proxy container. This requires creating multiple files and subdirectories, which should all be stored inside the **proxy** directory.
```bash
mkdir proxy
cd proxy
```
2- Configure the Dockerfile:
``sudo nano Dockerfile``
then the file should contain the following:
```
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf
COPY ./includes/ /etc/nginx/includes/
COPY ./ssl/ /etc/ssl/certs/nginx/
```
The Dockerfile is based on the Nginx image. It also copies a number of files from the local machine:
-   the default configuration for the proxy service
-   proxy, SSL configurations, and certificates

3- Configure the `default.conf` File
``sudo nano default.conf``
```
# wordpress config.

server {
listen 80;
server_name ec2-3-145-217-255.us-east-2.compute.amazonaws.com;
return 301 https://$host$request_uri;
}
server {
listen 443 ssl default_server;
server_name ec2-3-145-217-255.us-east-2.compute.amazonaws.com;
# Path for SSL config/key/certificate
ssl_certificate /etc/ssl/certs/nginx/pxl.crt;
ssl_certificate_key /etc/ssl/certs/nginx/pxl.key;
include /etc/nginx/includes/ssl.conf; 

location / {
include /etc/nginx/includes/proxy.conf;
proxy_pass http://docker-compose_wordpress_1;
}
}
```
4- Then make a directory called `ssl` then copy your certificates and the key
OR make new certificates with the following steps:
``cd ssl``
``touch pxltest.key pxltest.crt ``
Then, use OpenSSL to generate keys and certificates for your web services. For the WordPress web service (pxltest), run the command:
``sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout pxl.key -out pxl.crt``

5- Edit the Proxy and SSL Configuration:
 A- Exit out of the `ssl` subdirectory and back into `proxy` dir. To do so, use the command:
```bash
cd ..
mkdir includes
cd includes
touch proxy.conf  ssl.conf
```
B- Next, open the  `proxy.conf` file: 
``sudo nano proxy.conf``
C- Add the following configuration:
```
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_buffering off; proxy_request_buffering off; 
proxy_http_version 1.1; proxy_intercept_errors on;
```
D- Save and exit `proxy.conf` . 
F- Next, Open the `ssl.conf` file:
``sudo nano ssl.conf``
G- Add the following lines in the file:
```
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHAECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
ssl_prefer_server_ciphers on;
```
7- Create `docker-compose.yml` for nginx-proxy 
```
version: '3'
services:
  proxy:
   build: .
   networks:
    - wordpress
   ports:
    - "80:80"
    - "443:443"  
networks:
   wordpress:
     external:
     #wordpress docker-compose
      name: docker-compose_default
```

6- Open your browser and type 
https://ec2-3-145-217-255.us-east-2.compute.amazonaws.com/
you should have the initial WordPress setup after you run the docker-compose for the WordPress that is discussed in the next step.

******
### Second Solution:
Using the Nginx Proxy Manager with the following steps:
1- Create a directory called Nginx-manager and add docker-compose.yml file and the following: 
```docker
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```
2- Bring your stack up by running ``docker-compose up -d `` .
3- Log in to the Admin UI : [http://127.0.0.1:81](http://127.0.0.1:81/) .
4- Use the default Credentials:
```
Email:    admin@example.com
Password: changeme
```
Immediately after logging in with this default user, you will be asked to modify your details and change your password.
5- Then open the Hosts tab and add a new Proxy host and the details for that 
Domain names with pxltest.com and the schema for the forward IP and port number which will be 80 in our wordpress app. then you can enable `Block Common Exploits` for more security.
Next, open the SSL tab and add your signed self Certificates or request a new SSL Certificate from `Let's Encrypt`.

### My preferred solution:
I prefer using the second one (Nginx Manager) as It is easier to set up and manage and gives many other options. 


### Now the second part of the task is to move the sensitive data out of `docker-compose.yml` and store them in a more secure way.
This will use the environment file `.env` and save into the server not in the docker-compose file like the following: 
```docker-compose
version: '3.3'
services:
  db:
    image: mysql:5.7
    volumes:
      - ./db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${Mysql_Root_Password}
      MYSQL_DATABASE: ${Mysql_Database}
      MYSQL_USER: ${Mysql_User}
      MYSQL_PASSWORD: ${Mysql_Password}  
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
     - "80"
    restart: always
    volumes:
       - ./html:/var/www/html
    environment:
       WORDPRESS_DB_HOST: ${Wordpress_DB_Host}
       WORDPRESS_DB_USER: ${Wordpress_DB_User}
       WORDPRESS_DB_PASSWORD: ${Wordpress_DB_Password}
       WORDPRESS_DB_NAME: ${Wordpress_DB_Name}
```
The `.env` file will be saved only on the servers that run the Docker-compose :
```env
# MySQL Environment
Mysql_Password=somewordpress
Mysql_Database=wordpress
Mysql_User=wordpress
Mysql_Root_Password=wordpress  
#wordpress Environment
Wordpress_DB_Host=db:3306
Wordpress_DB_User=wordpress
Wordpress_DB_Password=wordpress
Wordpress_DB_Name=wordpress
```
#####  Run your Docker compose with the following command.
``docker-compose --env-file .env up -d ``

#### Open your browser and type 
https://ec2-3-145-217-255.us-east-2.compute.amazonaws.com/

*************

## The third task :
1- The `.github` folder :
On Github, folder  `.github`  is just a convention folder used to place Github-related stuff inside it. Github handles some of these files even when you place them at the root of your project (_such as  `CONTRIBUTING.md`,  `CODE_OF_CONDUCT.md`  etc). 
Some of the most used files in the  `.github`  folder:
-   `CODE_OF_CONDUCT.md`  -> How to engage in community and how to behave yourself.
-   `CONTRIBUTING.md`  -> How to contribute to the repo (_making pull request, setting development environment..._)
-   `LICENSE.md`  - A software license tells others what they can and can't do with your source code  (_You should place this at the root of your project since GitHub ignores it in the  `.github`  folder. You can find this file while browsing other Git hosting services such as GitLab, Bitbucket etc.)
-   `FUNDING.yml`  -> Supporting a project
-   `ISSUE_TEMPLATE`  -> Folder that contains templates of possible issues user can use to open issue (_such as if the issue is related to documentation, if it's a bug, or if the user wants a new feature, etc_).
-   `PULL_REQUEST_TEMPLATE.md`  -> How to make a pull request to project
-   `stale.yml`  -> Probot configuration to close stale issues. There are many other apps on Github Marketplace that place their configurations inside the `.github`  folder because they are related to GitHub specifically.
-   `SECURITY.md`  -> How to responsibly report a security vulnerability in the project
-   `workflows`  -> Configuration folder containing yaml files for GitHub Actions
-   `CODEOWNERS`  -> Pull request reviewer rules. More info  [here](https://docs.github.com/en/github/creating-cloning-and-archiving-repositories/about-code-owners).
-   `dependabot.yml`  -> Configuration options for dependency updates

#####  The workflow directory 
A workflow is a configurable automated process that will run one or more jobs. Workflows are defined by a YAML file checked into your repository and will run when triggered by an event in your repository, or they can be triggered manually, or at a defined schedule.
In our case study, there is a file called `cicd.yml` in the workflow directory 
which has the jobs that will be applied depending on the trigger that we put inside the file. 
Let's go to the file and explain these steps;

 - ``name: Build Docker Images`` ⇒ display the name of our workflow and
   it is optional.
 - This is the section where we define the Events  :
  Every time that an event occurs, we can trigger a certain workflow.
  in our case the workflow will run when a pull request is made to development and main branches in the repository:
 ```  on:   pull_request:
       branches:
        - development
        - main     
```
 - Jobs section is the part that gets executed whenever our events happen.
 - ``build_docker_images:`` ⇒ Our job name.
 - ``runs-on: ubuntu-latest``⇒ Define the type of machine to run the job on. The machine can be either a GitHub-hosted runner or a self-hosted runner.
 - ``steps:`` ⇒ the sequence of process that will run.
 
 - First step is called checkout which check out the code in the repository.
 **actions/checkout@v2** path in GitHub is where pre-created or predefined actions are hosted so all can use it. (https://github.com/actions)
  ```
name: Checkout
uses: actions/checkout@v2
```
 - ``name: Docker meta`` ⇒ GitHub Action to extract metadata from Git reference and GitHub events. This action is particularly useful if used with [Docker Build Push] action to tag and label Docker images.
 ```
 name: Docker meta
id: docker_meta
uses: crazy-max/ghaction-docker-meta@v1
with:
images: ghcr.io/awesomecompany/mlflow
tag-sha: true
```
 - ``name: Set up QEM`` ⇒ GitHub Action to install [QEMU](https://github.com/qemu/qemu) static binaries.
 QEMU is a generic and open source machine & userspace emulator and virtualizer.
QEMU is capable of emulating a complete machine in software without any need for hardware virtualization support. By using dynamic translation, it achieves very good performance.
```
name: Set up QEMU
uses: docker/setup-qemu-action@v1
```
 - After building our image we have to push the created image into the docker registry. so first we have to login into the registry using this action:
 ```
 name: Login to GitHub Container Registry
uses: docker/login-action@v1
with:
registry: ghcr.io
username: ${{ github.repository_owner }}
password: ${{ secrets.CR_PAT }} 
```
 - Then Build and push our image using this action 
 Build docker images using docker file located in the following path `./mlflow/Dockerfile`
 and tag it using the output from docker meta action.
 ```
 name: Build and push MLFlow image
id: mlflow
uses: docker/build-push-action@v2
with:
context: ./
file: ./mlflow/Dockerfile
push: true
tags: ${{ steps.docker_meta.outputs.tags }}
```
### Conclusion:
This is a pipeline that is used to automate the development workflow. 
It is triggered when a pull request happened on the repository and it runs on the Ubuntu agent and builds a docker image then push it to the Docker registry.

### Pull request template:
Pull request templates allow your organizations to have a default text when you create a pull request on GitHub. It is quite useful to make sure to follow a standard process for every pull request and to have a to-do list for the author to check before requesting a review.
When you add a pull request template to your repository, project contributors will automatically see the template's contents in the pull request body.