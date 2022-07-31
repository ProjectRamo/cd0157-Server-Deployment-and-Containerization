# Up and running

git cloned from repo
used conda env "contain"
updated requirements.txt to flask 2.1.0 because the old one used too many unsupported packages
pip installed requirements.txt

# Just used their example defaults:

 ## A secret text string to be used to creating a JWT
 > export JWT_SECRET='myjwtsecret'
 > export LOG_LEVEL=DEBUG
 ## Verify
 > echo $JWT_SECRET
 > echo $LOG_LEVEL


 # Testing

## To run server
> python main.py
## New terminal
> curl --request GET http://localhost:8080/
"Healthy"

## check token
tried "brew install jq" but it was already installed
> export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
> echo $TOKEN

## decode jwt
> curl --request GET 'http://localhost:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .

# Dockerize

## Dockerfile

"
# Use the `python:3.7` as a source image from the Amazon ECR Public Gallery
# We are not using `python:3.7.2-slim` from Dockerhub because it has put a  pull rate limit. 
FROM public.ecr.aws/sam/build-python3.7:latest
# Set up an app directory for your code
COPY . /app
WORKDIR /app
# Install `pip` and needed Python packages from `requirements.txt`
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
# Define an entrypoint which will run the main app using the Gunicorn WSGI server.
ENTRYPOINT ["gunicorn", "-b", ":8080", "main:APP"]
"

## .env_file
JWT_SECRET='myjwtsecret'
LOG_LEVEL=DEBUG

## build the image
docker build -t udfsndimage .

## check image was built
docker image ls

## run the image
docker run --name udfsnd --env-file=.env_file -p 80:8080 udfsndimage

## test container
curl --request GET 'http://localhost:80/'
### Calls the endpoint 'localhost:80/auth' with the email/password as the message body. 
### The return JWT token assigned to the environment variable 'TOKEN' 
export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"WindowsPwd"}' --header "Content-Type: application/json" -X POST localhost:80/auth  | jq -r '.token'`
echo $TOKEN
# Decrypt the token and returns its content
curl --request GET 'http://localhost:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .


