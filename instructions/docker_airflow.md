Airflow in Docker Info
For this project, the docker-compose.yaml file comes from the Airflow in Docker. This defines all the services we need for Airflow, e.g., scheduler, web server, and so forth.

When we run this docker-compose file further down, it will start our containers/services. I've only changed a few things in this file:

These two extra lines added under volumes will mount these folders on our local file system to the docker containers. You can see other volumes are defined, one being to mount the ./dags folder (this is where we store dags airflow should run). The first line below mounts our extraction folder to /opt/airflow, which contains the scripts our airflow DAG will run. The second line mounts our aws credentials into the docker containers as read only.

- ./extraction:/opt/airflow/extraction
- $HOME/.aws/credentials:/home/airflow/.aws/credentials:ro
This line pip installs the specified packages within the containers. Note that there are others ways we could have done this.

_PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:- praw boto3 configparser psycopg2-binary}
Installing Docker 
First install Docker. Instructions here.

Next install Docker Compose. Instructions here.

Running Airflow 
To start our pipeline, we need to kick off Airflow which requires a couple more prerequisite steps.

If using Windows, you may need to make a small update to the below line in the docker-compose.yaml file. Here we are mounting our aws credentials file on to a docker container.

- $HOME/.aws/credentials:/home/airflow/.aws/credentials:ro
Increase CPU and Memory in Docker Desktop resource settings to whatever you think your PC can handle.

Run the following. You may be able to skip this step if you're not on linux. See here for more details.

cd ~/Reddit-API-Pipeline/airflow

# Create folders required by airflow. 
# dags folder has already been created, and 
# contains the dag script utilised by Airflow
mkdir -p ./logs ./plugins

# This Airflow quick-start needs to know your
# host user id
echo -e "AIRFLOW_UID=$(id -u)" > .env
Making sure you are still in the airflow directory, initialise the airflow database. This will take a few minutes. Make sure the Docker daemon (background process) is running before doing this.

docker-compose up airflow-init
Create our Airflow containers. This could take a while. You'll know when it's done when you get an Airflow login screen at http://localhost:8080.

docker-compose up
If interested, once containers are created, you can view them in Docker Desktop, or list them from the command line with:

docker ps
You can even connect into a docker container and navigate around the filesystem:

docker exec -it <CONTAINER ID> bash
As mentioned above, navigate to http://localhost:8080 to access the Airflow Web Interface. This is running within one of the Docker containers, which is mapping onto our local machine with port 8080. If nothing shows up, give it a few minutes more. Password and username are both airflow. For understanding the UI, I'd recommend looking at some guides like this one.

The dag etl_reddit_pipeline should be set to start running automatically once the containers are created. It may have already finished by the time you login. This option is set within the docker-compose file (AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'false'). The next DAG run will be at midnight. If you click on the DAG and look under the Tree view, all boxes should be dark green if the DAG run was successful. If there's any issues, this resource or the ones linked previously might help. Essentially, you'll want to click on any box that's red, click logs and scan through it until you find the issue.

If you want to shut down the airflow containers, run the following command from the airflow directory:

docker-compose down
Or if you want stop and delete containers, delete volumes with database data and download images, run the following. This can be useful if you want to remove everything and start from scratch. It's a good idea to do some reading into docker commands before running something like this though, so you understand what it's doing.

docker-compose down --volumes --rmi all
Explanation
If you check in the airflow/dags folder, you'll find a file titled elt_reddit_pipeline.py. This is our DAG which you saw in Airflow's UI.

It's a very simple DAG. All it's doing is running 3 tasks, one after the other. This DAG will run everyday at midnight. It will also run once as soon as you create the Docker containers. These tasks are using BashOperator, meaning that they are running a bash command. The tasks here are running a bash command to call external Python scripts (these Python scripts also exist within our docker container through the use of volumes). You'll find them under the extraction folder.

Read below for more details:

extract_reddit_data_task

This is extracting Reddit data. Specifically, it's taking the top posts of the day from r/DataEngineering and collecting a few different attributes, like the number of comments. It's then saving this to a CSV within the /tmp folder.

upload_to_s3

This is uploading the newly created CSV to AWS S3 for storage within the bucket Terraform created.

copy_to_redshift

This is creating a table in Redshift if it doesn't already exist. It's then using the COPY command to copy data from the newly uploaded CSV file in S3 to Redshift. This is designed to avoid duplicate data based on post id. If the same post id is in a later DAG run load, then warehouse will be updated with that record. Read here for information on the COPY command.
