# Monitoring a ClearBlade Platform Instance

In this repo you will find everything you need to setup an ELK stack and be able to easily monitor a ClearBlade Platform instance. This includes a step-by-step guide, and related configuration files to make this as easy as possible.

## Step 1: Setup ELK stack

First things first, you will need an ELK (Elasticsearch, Logstash, Kibana) stack up and running. All of these tools are open source and free to use.

ClearBlade suggests leveraging [docker-elk](https://github.com/deviantony/docker-elk) to make this a quick and easy process. If you already have an ELK stack, you can check the logstash directory within this repo for the needed config you will need to easily process ClearBlade Platform log messages. It's recommended to set this stack up on a seperate host machine than your platform instance. Below are steps to get the ELK stack up and running on a Linux based host machine:

1. Follow the Host setup steps within the Requirements section in the docker-elk README
    - Install Docker
    - Install Docker Compose
    - Clone the docker-elk repo locally on the host
2. Modify the default docker-elk docker-compose.yml
    - Add log options to all 3 containers to prevent excesive logging
        - Add a logging section at the bottom of each service, it should look something like this:
            ```yml
            logging:
                driver: "json-file"
                options:
                    max-size: "10m"
                    max-file: "10"
            ```
    - Persist Elasticsearch data
        - Under elasticsearch volumes, you need to mount an elasticsearch data volume so all data will persist if the container is removed (change the first path to wherever you want on the host machine):
            ```yml
            - /path/to/storage/on/host:/usr/share/elasticsearch/data
            ```
        - **Note**, you also need to change the permissions on the host machine for the directory you picked above. Specifically the directory must be owned by the uid `1000`
3. Modify the Dockerfile for Logstash
    - We need to add some logstash plugins to be installed when the logstash container is created. Simply modify the Dockerfile at `docker-elk/logstash` by adding the below snippet to the end of the file:
    ```Dockerfile
    RUN logstash-plugin install logstash-input-beats logstash-filter-translate logstash-filter-grok  logstash-filter-date logstash-filter-mutate logstash-filter-drop logstash-filter-json 
    ```
4. Add ClearBlade's logstash.conf
    - You'll need to grab the logstash.conf within this repo at `cb-monitoring/logstash`, and copy it over to the ELK host, specifically at `docker-elk/logstash/pipeline`
5. Start the ELK stack
    - Run `docker-compose up -d` within the docker-elk directory.
    - It can take a while (~5-10 minutes) depending on the host hardware to initialize everything
    - Once complete, you'll be able to access the Kibana web UI at `http://localhost:5601`
6. Import default Metricbeat dashboards
    - This step will load some useful base Dashboards and Visualizations within Kibana, as well as load an Index Template into Elasticsearch
    - Just run the following commnd on the ELK host to import everything at once:
    ```bash
    docker run docker.elastic.co/beats/metricbeat:5.5.1 ./scripts/import_dashboards
    ```
7. Check everything is up and running
    - Check that you can access Kibana web UI at `http://localhost:5601`
    - You should see a number of Dashboards and Visualizations already available. When viewing them they will likely error out and not display anything, but that is expected for now.


## Step 2: Create and Add Monitoring Docker Containers

Next on the list is to configure your ClearBlade Platform host to send both logs and resource usage data to the newly created ELK stack. Since this host already is likely using Docker, we just need to run two new containers.

1. Add logspout container to send logs from the ClearBlade container to ELK
    - On your host running the ClearBlade Platform, simply run the following command (after replacing in your ELK stack IP/host) to download and run the most recent logspout container, and configure it to only send logs from the clearblade container to your ELK stack. 
    ```bash
    docker run -d --name="logspout" --volume=/var/run/docker.sock:/var/run/docker.sock --restart always \
     --log-opt max-size=1m --log-opt max-file=3 \
     gliderlabs/logspout tcp://<your ELK stack IP address or host name>:5000?filter.name=clearblade
    ```
2. Add metricbeat container to send host machine usage and docker details to ELK
    - You can find a suggested metricbeat config file at `metricbeat/metricbeat.yml`
    - To create and run the container, run the following docker command:
    ```bash
    docker run \
     --volume=/path/to/metricbeat/config:/usr/share/metricbeat/metricbeat.yml \
     --volume=/proc:/hostfs/proc:ro \ 
     --volume=/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro \ 
     --volume=/:/hostfs:ro \ 
     --net=host 
     docker.elastic.co/beats/metricbeat:5.5.1 -system.hostfs=/hostfs
    ```
3. Create a Logstash Index in Elasticsearch
    - Access your Kibana web UI, and navigate to `Management -> Index Patterns -> Create Index`
    - In Index name or pattern box, put `logstash-*`
    - Click `Create` at the bottom of the screen
4. Finally, check that both ClearBlade Platform container logs are being sent, as well as host machine metrics
    - In the Kibana web UI, navigate to `Discover`
    - You should in the top left see a drop down that containes both a `metricbeat-*` and `logstash-*` index. Select each and make sure you see data for both indices

## Step 3: Visualizing monitoring and logging data in Kibana

Fow now, we recommend looking at the default metricbeat provided dashboards and visualizations, and customizing them to your liking. In the near future we hope to provide our own template dashboards and visualizations you can easily import into Kibana.

## Other Stuff

### Curator

Also in this repo is a `curator` directory. Curator is a tool to help manage your elasticsearch data. ClearBlade specifically leverages it to clear out old log and metric data from elastic search. Check the README within that directory for more details.
