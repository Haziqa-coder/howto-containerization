# How to containerize a PHP application using Docker and Github Actions

## What is Containerization ?
It is a process in which you install the prerequisites of an application in a package form under the same kernel.

Normally if you configure an application A on the server which runs on port 80. Now you want another application B to run which has the same packages and libraries, in this case you will have to share those packages and libraries and if proper permissions are not assigned they will interfere with one another and potentially both applications will crash at this stage.

Here application containerization came into the picture.

In application containerization we create an image of the application and that image can run multiple instances of the application without interfering with one another.

In this article we will focus on how you can containerize your application with best practices. Moreover there will be practical demonstration at the end of this with python application

### 1. Choose an image
There are alot of technology specific images that are already available publicly. For example 
https://hub.docker.com/_/python (For python base application)
https://hub.docker.com/_/node (For nodejs base application)

If the application uses any custom framework, you can select a base OS image like ubuntu, centos and on top of that you can install the application

### Install Packages and Libraries
This is a very important step in which you have to install the packages required by your application.
**Tip: Make sure that you keep the image size as minimum as possible. Tools like telnet, net-tools can be installed in another image for debugging and troubleshooting**
- You might often need to run apt-get update and after that you will be able to install any linux packages / libraries. This is because apt-get update cache the layers of packages in the system so it can recognize and install those packages
- Make sure that your application only needs to install those packages which are required.

### Custom Configuration Files
If your application needs configuration files you can add them using two command
- **ADD**: It will copy all the files present in your host machine folder and transfer it to docker image while building
Example: 
    ```
    ADD config /tmp/folder/
    ```
    So whatever the files in the config folder will be copied over to /tmp/folder/
- **COPY**: it will copy the file from your host machine to docker image while building
Example: 
    ```
    COPY config.txt /tmp
    ```
### Non-root User
Application should run as non-root user this is the best security practice. While building a docker image you can define an USER from which you perform the steps.
Example:
```
FROM nginx
USER username
RUN command
```
### Exposed Ports
At the end of Dockerfile you can mention an executable script from which application can run.
- **Entrypoint**: Mention at the end of dockerfile mostly and it cant be override at the runtime command
- **CMD**: Mention at the end of dockerfile and it can be override at runtime command

### Environment Variables
You can use the environment variables inside the dockerfile. Also if you are using an already predefined system image lets say mariadb has its own environment variable while running the docker container.
Have a look at mariadb official image
https://hub.docker.com/_/mariadb

### Build Arguments
You can use the build arguments while building the dockerfile. The build arguments will help you define inputs in the Dockerfile.
Example:
```
ARG NODE_VERSION
FROM node:$NODE_VERSION
```
Now we will execute this dockerfile using build arguments
```bash
docker build -f Dockerfile . --build-arg NODE_VERSION=10
```
### Configuration Method
Every application requires some kind of parameterization. There are three types of configuration that we can define.
- Application specific configuration file, usually its some kind of environment variable file (.env) in which you have define the required parameters for the application
- Use built-in system variables. I.e OS environment variables etc
- Define a YAML file for all the required fields and use those fields in your code. This will also help to see the entire application configuration if it needs debugging.

### No Persistent Data
The best practice is to not save your data inside the container. Make sure that all your data needs to be in some kind of database. If your application needs to save data then use volumes that are hosted on the base machine in which the docker engine is running.

### Logs Handling
Make sure that your application is generating and saving the logs in a file. Moreover that file and folder is mounted through the volume to the base machine os. As a best practice, logs need to be persistent. It helps in troubleshooting and debugging the application issues.

## DEMO

Now most of the concepts and best practices are covered, we will now move on to the practical demonstration. In this demo we will build a docker image of a python flask base application. We will also create a pipeline with Github Actions so if anything changes in the github repository it will build and push to image to Github Registry. 

Below is the Github workflow that we will creating through this article

![Alt text](./images/images-1.png?raw=true "Github flow")

### Step 1: Create an app.py file
We will create a simple python file in which we have defined a function that will display “Hello from Docker”.
```python
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello_geek():
    return '<h1>Hello Docker</h2>'
if __name__ == "__main__":
    app.run(debug=True)
```
### Step 2: Create requirement.txt file
Now we will create a requirement.txt which contains all the python packages that are required to run the flash application as mentioned in step 1.
```python
click==8.0.3
Flask==2.0.2
itsdangerous==2.0.1
Jinja2==3.0.2
MarkupSafe==2.0.1
Werkzeug==2.0.2
```
### Step 3: Create Dockerfile
Next we will create an image through the dockerfile which contains the packages required to run the application and the end of dockerfile we will place a command that will inside the container and expose on 5000 port internally.
```
ARG PYTHON_VERSION
FROM python:$PYTHON_VERSION-slim-buster

WORKDIR /python-docker

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt


EXPOSE 5000

CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0"]
```

### Step 4: Build Docker image
Execute the below command to build the docker image with the name of python-docker.

```bash
docker build -f Dockerfile . --build-arg NODE_VERSION=3.8 --tag python-docker 
```

### Step 5: Test Docker Image
Once the building of the image is successful we can now deploy the container from it.
```bash
docker run -d --name python -p 5000:5000 python-docker
```
The above command parameter details given below.
- -d: running in detach mode
- -p 5000:5000: denotes <container port>:<exposed port on host>
- At the end we have mentioned the docker image name (python-docker)

### Step 6: Verify Its working
We will use curl command to display the flask function that is giving the right message or not.
```bash
curl http://localhost:5000
```
![Alt text](./images/image-2.png?raw=true "Curl Verification")

Its giving us an output that’s good. Once you are done with the testing then move this code to github repository using the below commands.

```bash
git clone https://github.com/<github-username>/docker-python.git
cp Dockerfile requirements.txt app.py docker-python/
git add .
git commit -m 'initial commit'
git push origin master
```

### Step 7: Create the Github Actions Pipeline
Goto your github repository. Click on Actions select **Python Application**

![Alt text](./images/image-3.png?raw=true "Pipeline")

Remove all the lines from steps section and add the below code
```yaml
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2
        name: checkout the code
      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          #password: ${{ secrets.GH_PASSWORD }}
      - name: docker build
        run: |
          docker build . --file Dockerfile --build-arg PYTHON_VERSION=3.8 --tag ghcr.io/<github-username>/docker-python:latest
      - name: docker push
        run: |
          docker push ghcr.io/<github-username>/docker-python:latest
```

In the above YAML code we have defined three steps and every step uses the action in it.
- First step is checking out the code
- Second step is authenticating to github registry with username and password
- Third step it is building the docker image using Dockerfile
- Fourth step its pushing the image to github registry
For more details about the github action used in the steps check out the below link.

https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions

Once the code is added it will look like this as mentioned in the below screenshot

![Alt text](./images/image-4.png?raw=true "Github Actions Templates")

After that you need to commit the file. The successful step will show it like this

![Alt text](./images/image-5.png?raw=true "Pipeline Output")















