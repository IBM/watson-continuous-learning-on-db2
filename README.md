# Continuously train a cloud-based machine learning model from on-premise Db2 data

[![Build Status](https://travis-ci.org/IBM/watson-continuous-learning-on-db2.svg?branch=master)](https://travis-ci.org/IBM/watson-continuous-learning-on-db2)

TODO

![Architecture](doc/source/images/architecture.png)

## Flow

1. Source data is retained in on-premise Db2 database.
1. Data is accessible to Watson Studio via a Secure Gateway.
1. Secure gateway is utilized to train a cloud-based machine learning model.
1. Training feedback is stored in Db2 Warehouse for continuous learning.

## Included components

* [IBM Db2 Warehouse on
  Cloud](https://www.ibm.com/cloud/db2-warehouse-on-cloud): An elastic,
  fully-managed cloud data warehouse service that is powered by IBM BLU
  Acceleration technology for increased performance and optimization of
  analytics at a massive scale.
* [IBM Watson Studio](https://www.ibm.com/cloud/watson-studio): Build and train
  AI models, and prepare and analyze data, in a single, integrated environment.

## Featured technologies

* [Docker](https://www.docker.com/): Docker containers wrap up software and its
  dependencies into a standardized unit for software development that includes
  everything it needs to run: code, runtime, system tools and libraries.

## Watch the video

[![Demo on YouTube](https://img.youtube.com/vi/HCVxGMd1RiQ/maxresdefault.jpg)](https://www.youtube.com/watch?v=HCVxGMd1RiQ)

## Steps

1. [Create IBM Cloud service instances](#create-ibm-cloud-service-instances)
1. [Load sample data into an on-premise Db2 database](#load-sample-data-into-an-on-premise-db2-database)
1. [Configure a secure gateway to IBM Cloud](#configure-a-secure-gateway-to-ibm-cloud)
1. [Connect to on-premise Db2 database from Watson Studio](#connect-to-on-premise-db2-database-from-watson-studio)
1. [Create a machine learning model](#create-a-machine-learning-model)
1. [Enable continuous learning](#enable-continuous-learning)

### Create IBM Cloud service instances

In order to build and train your machine learning model, you'll need to [sign
up for Watson Studio](https://www.ibm.com/cloud/watson-studio), which you can
do for free. In our use case, Watson Studio will rely on the Object Storage
service to store it's data, the Apache Spark service for data processing,
and the Machine Learning service for building machine learning models.
Finally, we'll use a DB2 Warehouse to enable continuous learning.

From the [IBM Cloud Catalog](https://console.bluemix.net/catalog/), select the
[**Storage**](https://console.bluemix.net/catalog/?category=storage) category,
and then the [**Object
Storage**](https://console.bluemix.net/catalog/services/cloud-object-storage)
service. Then, click **Create**.

![Create IBM Cloud Object Storage service](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-create-object-storage.png)

From the [IBM Cloud Catalog](https://console.bluemix.net/catalog/), select the
[**Web and
Application**](https://console.bluemix.net/catalog/?category=app_services)
category, and then the [**Apache
Spark**](https://console.bluemix.net/catalog/services/apache-spark) service.
Then, click **Create**.

![Create IBM Cloud Apache Spark instance](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-create-apache-spark.png)

From the [IBM Cloud Catalog](https://console.bluemix.net/catalog/), select the
[**AI**](https://console.bluemix.net/catalog/?category=ai) category, and then
the [**Watson
Studio**](https://console.bluemix.net/catalog/services/watson-studio) service.
Then, click **Create**.

![Create IBM Cloud Watson Studio service](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-create-watson-studio.png)

From the [IBM Cloud Catalog](https://console.bluemix.net/catalog/), select the
[**Databases**](https://console.bluemix.net/catalog/?category=databases)
category, and then the [**Db2
Warehouse**](https://console.bluemix.net/catalog/services/db2-warehouse)
service. Then, click **Create**.

![Create IBM Cloud Db2 Warehouse instance](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-create-db2-warehouse.png)

### Load sample data into an on-premise Db2 database

The fastest way to get started with Db2 on-premise is to use the no-charge
community edition, running in a Docker container. However, if you already have
an on-premise Db2 instance, feel free to substitute that instead.

We'll be populating the database with a sample dataset of building code
violations, provided by the city of Chicago.

If you don't already have Docker installed, follow the official [installation
guide](https://docs.docker.com/installation/).

Start by using [`docker run`](https://docs.docker.com/engine/reference/run/) to
launch Db2 community edition in a container running in the background.

> Including the `--env LICENSE="accept"` argument indicates your acceptance of
> the [license
> agreement](http://www-03.ibm.com/software/sla/sladb.nsf/displaylis/5DF1EE126832D3F185257DAB0064BEFA?OpenDocument)
> to use the software contained in the Docker image.

This command sets a password for the default instance user (`db2inst1`) to
`db2inst1-pwd` and binds Db2 to listen on port `50000` of your workstation:

```bash
docker run \
  --name db2 \
  --env LICENSE="accept" \
  --env DB2INST1_PASSWORD="db2inst1-pwd" \
  --publish 50000:50000 \
  --detach \
  ibmcom/db2express-c \
  db2start
```

Next, run a `db2` command inside the running container using [`docker
exec`](https://docs.docker.com/engine/reference/commandline/exec/) as the
`db2inst1` user to [create a
database](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_10.5.0/com.ibm.db2.luw.admin.dbobj.doc/doc/t0004916.html)
named `onprem`:

```bash
docker exec db2 su - db2inst1 -c "db2 CREATE DATABASE onprem"
```

Next, we need to create a regular user for Watson Studio to connect as. Users
in Db2 are managed externally from Db2 itself, so let's create a system user
named `watson`, assign them a password (`secrete`), and grant them
database-level permissions.

```bash
docker exec db2 useradd -g db2iadm1 watson
```

```bash
docker exec -it db2 bash -c "echo secrete | passwd --stdin watson"
```

```bash
docker exec db2 su - db2inst1 -c "db2 CONNECT TO onprem; db2 'GRANT DBADM, CREATETAB, BINDADD, CONNECT, CREATE_NOT_FENCED, IMPLICIT_SCHEMA, LOAD ON DATABASE TO watson'"
```

Using the new `watson` user account, create a database table in the `watson`
database for building code violations named `violations`:

```bash
docker exec db2 su - db2inst1 -c "db2 CONNECT TO onprem USER watson USING secrete; db2 'CREATE TABLE violations(ID INTEGER, VIOLATION_CODE VARCHAR(20), INSPECTOR_ID VARCHAR(15), INSPECTION_STATUS VARCHAR(10), INSPECTION_CATEGORY VARCHAR(10), DEPARTMENT_BUREAU VARCHAR(30), ADDRESS VARCHAR(250), LATITUDE DOUBLE, LONGITUDE DOUBLE)'"
```

Then, use [`docker
cp`](https://docs.docker.com/engine/reference/commandline/cp/) to push the
[`violations.csv`](violations.csv) file (from this repository) into the `db2`
container.

```bash
docker cp violations.csv db2:/tmp/
```

Load the sample data into the `onprem` database in Db2:

```bash
docker exec db2 su - db2inst1 -c "db2 CONNECT TO onprem USER watson USING secrete; db2 'LOAD FROM /tmp/violations.csv OF DEL REPLACE INTO violations'"
```

At this point, you have a Db2 database instance loaded with sample data.

### Configure a secure gateway to IBM Cloud

To enable Watson studio to access the on-premise database, you'll need to
configure a secure gateway, which allows limited network ingress to your
on-premise network as governed by an access control list (ACL).

Before you begin, you need to take note of your workstation's LAN IP. You can
find that address using:

```bash
$ hostname -I
192.168.1.100
```

> Your workstation may have more than one IP, but any of them will likely work
> for the purposes of this code pattern, because Docker will bind to all
> network interfaces.

> It's worth mentioning that `loopback` addresses (e.g. `127.0.0.1`) will not
> work here, because IBM Cloud services consuming the Secure Gateway will not
> be able to distinguish between their own `loopback` interfaces and a
> `loopback` interface on your end of the Secure Gateway.

For the purposes of this code pattern, we'll assume that your LAN IP is
`192.168.1.100`.

Create a [Secure
Gateway](https://console.bluemix.net/catalog/services/secure-gateway) from the
IBM Cloud Catalog.

![Create a new Secure Gateway](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-new-secure-gateway.png)

Once the gateway is created, select **Connect Client** and choose **Docker** as
the connection method.

The console will provide you with a complete Docker command to download and run
the secure gateway client, which looks something like `docker run -it
ibmcom/secure-gateway-client $GATEWAY_ID -t $SECURITY_TOKEN` (with a real
gateway ID and security token unique to your Secure Gateway). Use the copy icon
to copy the command, and run it locally.

By default, the gateway starts with an access control list in a default-deny
state, meaning that all incoming connections through the secure gateway will be
blocked from accessing any network resources. To allow Watson Studio to access
your Db2 instance, we need to specifically allow access to the port published
by Docker on your workstation's LAN IP address (e.g. `192.168.1.100`):

    acl allow 192.168.1.100:50000

Incoming connections from Watson Studio which pass through the secure gateway
will now be able to access Db2.

> When you're finished with this code pattern, you can close the secure gateway
> by typing `quit` at the prompt and pressing <kbd>Enter</kbd>.

See the
[documentation](https://console.bluemix.net/docs/services/SecureGateway/index.html#getting-started-with-sg)
for additional configuration options.

### Connect to on-premise Db2 database from Watson Studio

Create a **New Project**.

![Watson Studio home](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-dataplatform-home.png)

Select **Complete**, when prompted.

Enter `Violations` as the project name, and click **Create**.

![New Watson Studio project](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-new-project.png)

Near the top right of the screen, select the **Add to project** dropdown, choose
**Connection**, and select **Db2** from the available options.

![Add to Watson Studio project](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-add-to-project.png)

![New Db2 connection](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2-new-db2-connection.png)

Configure the connection as follows:

* **Name**: `On-Premise`
* **Database**: `onprem`
* **Hostname or IP Address**: your workstation's LAN IP (e.g. `192.168.1.100`)
* **Port**: `50000`
* **Secure Gateway**: &#9745; (and ensure your new Secure Gateway is selected in
  the corresponding dropdown menu)
* **Username**: `watson`
* **Password**: `secrete`

In the secure gateway terminal, you should see a log message indicating that a
connection was successfully established from Watson Studio:

    [2018-08-31 10:38:38.708] [INFO] (Client ID K75lSQ0Oppd_d87) Connection #1 is being established to 192.168.1.100:50000

Click the **Add to Project** dropdown again, and choose **Connected assets**.

Click **Select source**, then choose the **On-Premise** connection, then the
**WATSON** database, then the **VIOLATIONS** table, and finally, click the
**Select** button at the bottom of the screen.

#### Refine the asset

Watson Machine Learning models do not directly support data assets from
on-premise Db2 instances, so we have to setup a conversion process to "refine"
the data asset into a `CSV` file in object storage.

From the **Data assets** table, click on **Violations** (with **Data Asset** in
the **Type** column). At the top right, click **Refine**. We don't need to
manipulate the data, so simply click the "run" button labeled with a
**&#9654;** icon at the top right. The data flow output will show that you're
creating a `CSV` file, which will be saved into your object storage bucket.
Click **Save and Run**. You can then opt to view the data flow's progress by
clicking **View Flow**.

From your project **Assets** screen, you should now see a new **Data asset**
named `Violations_shaped.csv`.

### Create a machine learning model

#### Create a WML model

From your project's **Assets** screen, click **New Watson Machine Learning
model**. Name your model **Violations Predictor**, select your Apache Spark
service instance from the **Spark Service or Environment** dropdown, and click
**Create**.

You'll then be asked to choose your data asset. Use the radio button to select
the CSV file you just created, named `Violations_shaped.csv`, and click
**Next**.

When the data is finished loading, you'll be asked to **Select a technique**.
Choose `INSPECTION_STATUS (String)` as your **Column value to predict (Label
Col)**, and choose **Multiclass Classification** as your technique. Click
**Train**.

When training is complete, click **Save** to store your model.

You can now use the model's endpoint in your own application. However, if you
also enable continuous learning, then your application will become smarter over
time, without having to update the application.

### Enable continuous learning

In order to enable continuous learning, we need to create a feedback loop to
the machine learning model. To accomplish that, we need to create a feedback
table in Db2 Warehouse on Cloud.

From the IBM Cloud dashboard, navigate to your Db2 Warehouse instance, and
click **Open**. Use the hamburger menu **&#9776;** in the top left, and click
**Run SQL**. Run the following two statements to create a feedback table, and a
SQL trigger to automatically populate a new column as new rows are inserted.

> TODO: This schema is not accepted by the performance monitor unless all the
> `INTEGER` and `DOUBLE` columns are typed as VARCHAR, which seems absurd.
> Otherwise, you get errors configuring performance monitoring such as "The
> feedback table indicated in learning configuration VIOLATIONS_FEEDBACK
> already exists, but it is not compatible with expected schema: Column ID has
> incompatible type INTEGER vs expected StringType"

> TODO: Why would you want the _TRAINING column to be non-nullable in the first
> place?

> TODO: Is the Db2 trigger intended to ease first time setup, or is it supposed
> to remain as part of the continuous learning process? I would think the
> machine learning service would populate that column by itself, when
> re-training occurs.

    CREATE TABLE
      violations_feedback(
        ID INTEGER,
        VIOLATION_CODE VARCHAR(20),
        INSPECTOR_ID VARCHAR(15),
        INSPECTION_STATUS VARCHAR(10),
        INSPECTION_CATEGORY VARCHAR(10),
        DEPARTMENT_BUREAU VARCHAR(30),
        ADDRESS VARCHAR(250),
        LATITUDE DOUBLE,
        LONGITUDE DOUBLE,
        "_TRAINING" TIMESTAMP NOT NULL)
      ORGANIZE BY ROW;

    CREATE TRIGGER
      feedback_trigger
      NO CASCADE
      BEFORE INSERT ON violations_feedback
      REFERENCING NEW AS n
      FOR EACH ROW SET n."_TRAINING"=CURRENT_TIMESTAMP;

    COMMIT;

#### Performance Monitoring

Next, we need to create a trigger in Watson Studio for re-training.

From your project's **Assets** tab in Watson Studio, Click your machine
learning model ("Violation Predictor"), click the **Evaluation** tab, and click
**Configure Performance Monitoring**.

Configure performance monitoring using the following values:

* **Spark Service or Environment**: choose your Apache Spark instance
* **Prediction type**: _multiclass_
* **Metric details**: _accuracy_ (leave the value blank)
* **Record count required for re-evaluation**: `100`
* **Auto retrain**: _when model performance is below threshold_
* **Auto deploy**: _when model performance is better than previous version_

Click **Save**.

When the configured trigger occurs, the Machine Learning service will pull in
new data from the feedback table and re-train the model. If the new model
performs better, then it will be automatically deployed.

## Sample output

![Continuous learning](http://browser-testing-cdn.dolphm.com/watson-continuous-learning-on-db2.png)

## Troubleshooting

TODO

## Links

* [Continuous Learning on Watson Studio](https://medium.com/ibm-data-science-experience/continuous-learning-on-watson-data-platform-cc39f3fd5042)
* [IBM Continuous Learning Blog Post Companion Materials](https://github.com/IBMDataScience/buildings_blog)

## Learn more

TODO

## License

[Apache 2.0](LICENSE)
