ExampleApplication
==============

Table of contents
=================

   * [Overview](#Overview)
   * [Running the application](#Running-the-application-as-a-kubernetes-pod-within-Spectrum-Discover)
      * [As a kubernetes pod within Spectrum Discover](#Running-the-application-as-a-kubernetes-pod-within-Spectrum-Discover)
      * [As a docker container on Spectrum Discover node](#Running-the-application-as-a-docker-container-on-Spectrum-Discover-node)
      * [As a python file](#Running-the-application-as-a-python-file)
   * [Implementation of ExampleApplication](#Implementation-of-ExampleApplication)
   * [Editing Example Application with custom code](#Editing-Example-Application-with-custom-code)
   * [Building your application](#Building-your-application)
   * [Publishing your application for use in the IBM Spectrum Discover Application Catalog](#Publishing-your-application-for-use-in-the-IBM-Spectrum-Discover-Application-Catalog)
   * [Additional Information](#Additional-Information)

## Overview
This Example Application will be used as a template in creating your own application.

Before starting this application:
1. Following pre-requisites are met:
    * Spectrum Discover 2.0.2+ is installed and running in your environment.
    * Spectrum Discover host is reachable over the network from the node on which you want use to run the application. This is needed if you are going to be running outside of the Spectrum Discover host.
    * Python 3 is installed on the system. This is needed if you are going to be running outside a docker container or kubernetes pod.
    * Python package manager `pip` is installed on the system. This is needed if you are going to be running outside a docker container or kubernetes pod.

2. There are 3 main ways to run an application
   1. Running on the IBM Spectrum Discover host as a kubernetes pod
   2. Running as a docker container (manual setup required)
   3. Running as a python file on Spectrum Discover or remote host (manual setup required)

Below, we detail each of the methods:

## Running the application as a kubernetes pod within Spectrum Discover
Information to run as a kubernetes pod can be found on the IBM Spectrum Discover Knowledge Center
1. https://www.ibm.com/support/knowledgecenter/SSY8AC
2. Choose your product release version
3. Table of Contents
4. Administering
5. Using the IBM Spectrum Discover application catalog


## Running the application as a docker container on Spectrum Discover node
1. See below topic: [Building your application](#Building-your-application) if you have not already created your image.
2. Create a file called `info.txt`. In here we will add required environment vars that will be passed into the docker container.
    ```
    SPECTRUM_DISCOVER_HOST=https://<spectrum_discover_host>
    # The IP or Fully Qualified Domain Name of your IBM Spectrum Discover instance.
    # This is required

    APPLICATION_USER=<application_user>
    # A dataadmin user. Ex: sdadmin
    # This is required

    APPLICATION_USER_PASSWORD=<application_user_password>
    # The password of the above dataadmin user.
    # This is required

    APPLICATION_NAME=<application_name>
    # A short but descriptive name of your application.
    # EX: exif-header-extractor-application or cos-x-amz-meta-extractor-application
    # Default: sd_sample_application

    KAFKA_DIR=<directory_to_save_certificates>
    # Directory where the kafka certs will be saved.
    # Default: ./kafka

    LOG_LEVEL=<ERROR WARNING INFO DEBUG>
    # Default: INFO
    ```
3. Using your docker "IMAGE ID" or tagged name from when you built your container, you can start it with the following command:  
`docker run --mount 'type=bind,src=/gpfs/gpfs0/connections/scale/id_rsa,dst=/keys/id_rsa' --env-file info.txt <container_id>`


## Running the application as a python file
1. Install the required python packages:  
    `sudo python3 -m pip install -r requirements.txt`

2. Install the required OS packages:  
If you have any NFS connections, you will need to install required packages.  
`sudo yum install nfs-utils`  


   NOTE: For each connection you have in the Admin page in the UI, the IBM Spectrum Discover Application SDK will create the following:
   * A sftp connection for each IBM Spectrum Scale connection
   * A local NFS mount for each NFS connection
   * A boto3 client for each COS connection

3. Define environment variables as follows:
    ```
    export SPECTRUM_DISCOVER_HOST=https://<spectrum_discover_host>
    # The IP or Fully Qualified Domain Name of your IBM Spectrum Discover instance.
    # Default: https://localhost
    ```
    ```
    export APPLICATION_NAME=<application_name>
    # A short but descriptive name of your application.
    # EX: exif-header-extractor-application or cos-x-amz-meta-extractor-application
    # Default: sd_sample_application
    ```
    ```
    export APPLICATION_USER=<application_user>
    # A dataadmin user. Ex: sdadmin
    ```
    ```
    export APPLICATION_USER_PASSWORD=<application_user_password>
    # The password of the above dataadmin user.
    ```
    ```
    export KAFKA_DIR=<directory_to_save_certificates>
    # Directory where the kafka certs will be saved.
    # Default: ./kafka
    ```
    ```
    export LOG_LEVEL=<ERROR WARNING INFO DEBUG>
    # Default: INFO
    ```


4. Now, you can start the sample application, running the code as follows:
    ```
    sudo -E python3 ./ExampleApplication.py
    ```

## Implementation of ExampleApplication

The code is extensively commented, and should be fairly self-evident.
The basic flow of an application is:

1. Register and initialize the application
```
    application = ApplicationBase(registration_info)
    application.start()
```
2. Get a message hander and read a message
```
    am = ApplicationMessageBase(application)
```

3. Read and parse the message
```
    msg = am.read_message(timeout=100)  # or timeout can be zero to poll
    work = am.parse_work_message(msg)
```

4. Process the message by getting a key for each document and retrieving it
```
    for docs in work['docs']:
        # DocumentKey is a unique identifier for a document, amalgam of connection + name
        key = DocumentKey(docs)
        drh = DocumentRetrievalFactory().create(application, key)

        tmpfile_path = drh.get_document(key)
```

5. Parse the content in the application specific way. This is where the custom code is per each application.
    If you are using this as a template, you can replace the code within the "Start Custom Code" and
    "End Custom Code" sections. See [Editing Example Application with custom code](#Editing-Example-Application-with-custom-code) section for more details.

6. Construct and send reply
```
     reply = ApplicationReplyMessage(msg)
     reply.add_result('success', key, tags)
     am.send_reply(reply)
```
For each document, we send a corresponding status of success, failed, skipped.

`success` is used when the file has been properly inspected.  
`failed` is used when the file failed to be inspected or could not be found.  
`skipped` is used when the file did not match the criteria to be inspected. ie: only inspect .jpg files and skip all others.  

## Editing Example Application with custom code
The first thing to do is to clone this repository. You can do so with the following command:
```
git clone git@github.com:IBM/Spectrum_Discover_Example_Application.git
```

The example application is documented with comments to explain what is happening within the file, but for a lot of use cases, you will only need to edit the code between
```
################ Start Custom Code ###############
```
and
```
################# End Custom Code ################
```

## Building your application
Once you have your custom code in place you can test it as a standalone python file or test it as a conatiner or pod. In order to be found in the IBM Spectrum Discover UI, it will need to be containerized and available on dockerhub. To build that container, follow the below procedure:

1. Rename your .py file to something descriptive of your application other then ExampleApplication.py.

2. Edit the Dockerfile:
   1. There are two places where the Dockerfile references the .py file. One where it copies the file into the container and the other where it executes it when starting up. Change these two locations with the name of your .py file.
   2. Edit the LABELS  
   In the example_application Dockerfile you will see `LABELS`. Some of these are required and others are optional. We will use these for filterting and displaying applications within the UI.
      The required ones are:
      ```
      LABEL application_name="example"
      The name of the application. We will append -application to the end for identification.
      ```
      ```
      LABEL filetypes="all"
      A comma separated list of file types this application works on. For ex: jpg,jpeg,tiff.
      ```
      ```
      LABEL description="Performs a character count on a specified file."
      A description of the application.
      ```
      ```
      LABEL version="1.2.3"
      A "Semantic Versioning 2.0.0" format of this application.
      ```
      ```
      LABEL license="mit"
      A short hand description of the license the source code is published under.
      ```

      The optional ones are:
      ```
      LABEL company_name=""
      The company name that published the application.
      ```
      ```
      LABEL company_url=""
      A URL linking to the company that created the application or link to the source code.
      ```
      ```
      LABEL maintainer="" # email address
      Email address of the maintainer.
      ```
      ```
      LABEL icon_url=""
      A URL to an icon to display from the UI.
      ```
      ```
      LABEL parameters=""
      A json formatted string of parameters.
      ```
   3. Add any additional OS packages your application needs. If you need to install OS packages you can add to the `RUN` line of the Dockerfile so it looks along the lines of:  
   ```
   RUN yum install -y pkg1 pkg2 pkg3 etc && \
   ...
   ```

3. Edit requirements.txt file:  
If you have any imports in your .py file that are required for your application add them to this file. It is preferred to specify exact versions that you tested with in order to ensure the best compatibility.

4. Build your docker image:  
`docker build .`

5. Once completeled successfully, the last line will be:
`Successfully built xxxxxxxxxxxx`


## Publishing your application for use in the IBM Spectrum Discover Application Catalog
Once you have built and tested your application and any changes have been accepted upstream you can publish your application to dockerhub.
This will allow for usage by other customers within their environment.

1. Once you have your container image built from the `docker build .` command, you should see an output with the last line being:
Successfully built \<container id\>  
EX: `Successfully built 20c6059bd651`


2. Tag your image with your docker username and image_name. Username is your username from hub.docker.com when you are signed in. The image_name will be a unique name of your choosing.  
EX: ibmcom/spectrum-discover-application-sdk  
To tag your image run:  
docker tag \<container_id\> \<username\>/\<image_name\>  
EX: `docker tag 20c6059bd651 ibmcom/spectrum-discover-application-sdk`


3. Push your image to hub.docker.com  
You will need to know your hub.docker.com credentials  
EX: `docker push ibmcom/spectrum-discover-application-sdk`  
You will be prompted for your dockerhub username and password. At this point enter them in.  
You will see that your container is now being uploaded to dockerhub.


4. Login to hub.docker.com in a browser if you haven't already.


5. Add the short description to the hub.docker.com image. This is required to be found by IBM Spectrum Discover as an application.  
Go to `https://hub.docker.com/r/<username>/<image_name>`  
EX: https://hub.docker.com/r/ibmcom/spectrum-discover-application-sdk
   1. Click 'Manage Repository'
   2. Right under your username/image_name in the upper right area, there is a small pencil icon. Click that
   3. Add the text 'IBM_Spectrum_Discover_Application'
   4. Click 'Update'


## Additional Information
   * Information on how the Application SDK works can be found here: https://github.com/IBM/Spectrum_Discover_Application_SDK
   * Information on existing IBM provided applications can be found here: https://github.com/IBM/Spectrum_Discover_App_Catalog