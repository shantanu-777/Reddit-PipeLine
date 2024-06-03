Setup (development)
Create a dbt.

Create a project or just stick to the default project created.

Setup Up a Database Connection - Select Redshift

On the next page, enter the relevant Redshift details. This includes the hostname. You'll find this in the AWS Redshift console, or from the terraform output command mentioned earlier. It will start with the name of your cluster and end with amazonaws.com. It will also require the port (likely 5439), the database name (likely dev). You'll also need the database username (likely awsuser) and password for the database you specified in the Terraform step. Once everything is input, click the Test button at the top. If successful, click Continue.

Once connection is established, choose managed directory and give it a name. You can also choose Github if you have a Github repo setup for the dbt part of this project, like I have, although this will require a bit of configuration.

Once you've worked through these initial steps, click on Start Developing.

You are now in an IDE which is connected to your Redshift cluster. Here we'll run some basic transformations on our data.

Click on initialize project. This will populate the directory on the left hand side with folder and files we may need.

Under the models folder, create new files called reddit_transformed.sql (this name will be the name of the new model you create) and schema.yml. You can delete the example folder.

In the schema.yml file, copy the following and save. Here we are defining some basic tests and documentation for our table. I haven't added much, as it's mostly for demonstration purposes. You can see that I've added a not_null test for id. For the rest, I've only added in the name of the column and a description.

version: 2

models:
  - name: reddit_transformed
    description: Transformed Reddit Data
    columns:
      - name: id
        description: Reddit ID of Post
        tests:
          - not_null
      - name: title
        description: Title of Reddit Post
      - name: text
        description: Body Text of Reddit Post
      - name: score
        description: Score of Reddit Post
      - name: comments
        description: Number of Comments for Post
      - name: url
        description: Full URL of Reddit Post
      - name: comment
        description: Top comment for Reddit Post
      - name: dateposted
        description: Date Reddit Data was Downloaded
In the text_posts.sql file, copy the following and save. This wil be our only transformation. This is basically removing some columns that don't really need, and also splitting the UTC datetime column into utc_date and utc_time. Feel free to transform the data in whichever way you want. This is just a VERY basic example.

SELECT id, 
   title, 
   num_comments, 
   score,
   author,
   created_utc,
   url,
   upvote_ratio,
   created_utc::date as utc_date,
   created_utc::time as utc_time
FROM dev.public.reddit
If you check the bottom right of the screen, you'll see a preview button. Click this to see what your outputted table will look like based on the above SQL query. Basically, when we run dbt run in the UI (further down this page) what'll happen is this table will be created in a new schema within our Redshift database.

Under the dbt_project.yml, update it to the following. All we've really changed here is the project name to reddit_project and told dbt to create all models as tables (rather than views). You can leave it as views if you wish.

# Name your project! Project names should contain only lowercase characters
# and underscores. A good package name should reflect your organization's
# name or the intended use of these models
name: 'reddit_project'
version: '1.0.0'
config-version: 2

# This setting configures which "profile" dbt uses for this project.
profile: 'default'

# These configurations specify where dbt should look for different types of files.
# The `source-paths` config, for example, states that models in this project can be
# found in the "models/" directory. You probably won't need to change these!
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"  # directory which will store compiled SQL files
clean-targets:         # directories to be removed by `dbt clean`
  - "target"
  - "dbt_packages"


# Configuring models
# Full documentation: https://docs.getdbt.com/docs/configuring-models

# In this example config, we tell dbt to build all models in the example/ directory
# as tables. These settings can be overridden in the individual model files
# using the `{{ config(...) }}` macro.
models:
  reddit_project:
    materialized: table
To test what we've done, we can run the following commands at the bottom of the DBT IDE and make sure an error isn't returned:

dbt run
dbt test
dbt run will generate our table within a different schema in Redshift. Easiest way to see this is to navigate to the AWS Console, login, search for Redshift, and use the Query Editor V2. dbt test will check the new model passes the tests (or test in our case) specified in the schema.yml file.

The next step is to click commit on the left hand menu to commit our changes.

If you ran dbt run above and no error was returned, a new table will have been created in our Redshift database, under a new schema name. You would have specified this schema name during the initial setup of dbt during the Redshift connection phase. If you left it as the default, it'll likely be something like dbt_<dbt user name>.
