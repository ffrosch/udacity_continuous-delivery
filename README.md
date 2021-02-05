# Deploying a Flask API

This is the project starter repo for the fourth course in the [Udacity Full Stack Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004): Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'.
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token.

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.

## Initial setup
1. Fork this project to your Github account.
2. Locally clone your forked version to begin working on the project.

## Dependencies

- Docker Engine
    - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/).
    - For Mac users, if you have no previous Docker Toolbox installation, you can install Docker Desktop for Mac. If you already have a Docker Toolbox installation, please read [this](https://docs.docker.com/docker-for-mac/docker-toolbox/) before installing.
 - AWS Account
     - You can create an AWS account by signing up [here](https://aws.amazon.com/#).

## Steps to run the API locally (to test endpoints)
1. Install python dependencies with `pip install -r requirements.txt`.
2. Set up the environment with `touch .env`, open the file and add these variables:
    ```bash
    JWT_SECRET='myjwtsecret'
    LOG_LEVEL=DEBUG
    ```
3. Run the app with the flask development server with `python main.py`

Open a new browser to run http://127.0.0.1:8080/ and it will give you a response as `"Healthy"`.

## Try the API endpoints on Command-Line
4. Open a new shell and install jq, which is a package that helps to read or manipulate JSON processors.
Use `sudo apt-get install jq` on Linux or `chocolatey install jq` on Windows.
5. To try the `/auth` endpoint, use the following command, replacing `<EMAIL>` and `<PASSWORD>` with any values:
    ```bash
    export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
    ```
6. This calls the endpoint 'localhost:8080/auth' with the `{"email":"<EMAIL>","password":"<PASSWORD>"}` as the message body. The return value is a JWT token based on the secret string you supplied. We are assigning that secret to the environment variable 'TOKEN'. To see the JWT token run `echo $TOKEN`.

7. To try the /contents endpoint which decrypts the token and returns its content, run:
    ```bash
    curl --request GET 'http://127.0.0.1:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
    ```

## Build and run a Docker Image
Build the Image:
```bash
docker build -t "jwt-api-test" .
docker image ls
```

If required, you can delete an image using `docker image rm -f <image_name>`

Run the Image locally:
```bash
docker run --env-file=.env -p 80:8080 jwt-api-test
docker container ls
```

If required, you can stop a container using `docker stop [OPTIONS] CONTAINER [CONTAINER...]`
or delete a container using `docker rm [OPTIONS] CONTAINER [CONTAINER...]`

## Project Steps

Completing the project involves several steps:

1. Write a Dockerfile for a simple Flask API
2. Build and test the container locally
3. Create an EKS cluster
4. Store a secret using AWS Parameter Store
5. Create a CodePipeline pipeline triggered by GitHub checkins
6. Create a CodeBuild stage which will build, test, and deploy your code

For more detail about each of these steps, see the project lesson [here](https://classroom.udacity.com/nanodegrees/nd004/parts/1d842ebf-5b10-4749-9e5e-ef28fe98f173/modules/ac13842f-c841-4c1a-b284-b47899f4613d/lessons/becb2dac-c108-4143-8f6c-11b30413e28d/concepts/092cdb35-28f7-4145-b6e6-6278b8dd7527).