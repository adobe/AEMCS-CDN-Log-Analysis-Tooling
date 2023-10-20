# ELK Docker Containers for AEM as a Cloud Service CDN Log Analysis

This repo lets you analyze AEM as a Cloud Service (AEMCS) CDN log files and visualize metrics through dashboards using the ELK stack.

## Overview

ELK stands for three popular projects: Elasticsearch, Logstash, and Kibana. The ELK stack is helpful for aggregating, analyzing logs, and creating visualizations for monitoring, and troubleshooting purposes. To quickstart the analysis, the following dashboards are provided:

- Traffic Filter Rules (including WAF)
- CDN Cache Hit Ratio

However, you can enhance and create additional dashboards to gain further insights and optimize the CDN configurations.

## Prerequisites

- Install [Docker](https://docs.docker.com/engine/install/) and increase the memory limit (`Preferences -> Resources -> Advanced`) to at least 4 GB.

- Download the [CDN logs](https://experienceleague.adobe.com/docs/experience-manager-cloud-service/content/implementing/using-cloud-manager/manage-logs.html?lang=en) you would like to analyze.

## How to set up the ELK Docker container

1. Clone this GitHub repository:

    ```shell
    $ git clone git@github.com:adobe/AEMCS-CDN-Log-Analysis-ELK-Tool.git
    ```

1. Navigate to the cloned repository:

    ```shell
    $ cd AEMCS-CDN-Log-Analysis-ELK-Tool
    ```

1. Create one folder for each AEM environment that you would like to monitor inside `logs/` folder and place your `.log` files there. For example, you could have three subfolders corresponding to your `dev`, `stage` and `prod` environments:

    ```shell
    $ mkdir -p logs/dev
    $ mkdir -p logs/stage
    $ mkdir -p logs/prod
    ```
        
    Alternatively, you can separate your logs by both Program ID and Environment ID in case you are analyzing logs from multiple AEM programs:

    ```shell
    $ mkdir -p logs/p<PROGRAM-ID>-dev
    $ mkdir -p logs/p<PROGRAM-ID>-stage
    $ mkdir -p logs/p<PROGRAM-ID>-prod
    ```
 
    There are no assumptions about the naming format but it is recommended to use suggestive names for these subfolders since they appear in your dashboards.

1. When you are ready, run the command below: 

    ```shell
    $ docker compose up
    ```

    In case you would like to add more logs, you can do it without stopping or restarting the Docker containers. Simply place your logs in one of the `logs` subfolders.

1. Once the above command-related output stops appearing in the console, it means that the ELK stack is ready to use. You can also verify by checking your Docker Desktop application (in the `Containers` section).

1. Navigate to <http://localhost:5601/app/discover> in your browser. You see the Elastic home page. 

    ![Elastic - Kibana Home Page](images/elk-home.png)

 
1. Import the provided dashboards by clicking `Menu` (three horizontal bars on the top-left side), followed by `Stack Management` and `Saved Objects` from the options on the left. Finally, click `Import` and select the desired `.ndjson` files from the cloned repository's `dashboards` directory.

1. Verify the imported dashboards by clicking `Menu -> Analytics -> Dashboards` and selecting the desired dashboard from the list.

    ![CDN Cache Hit Ratio Dashboard](images/Dashboard-CDN-Cache-Hit-Ratio.png)


This completes the one-time setup. Now you can ingest the logs and analyze them.

## How to analyze the AEMCS CDN logs

1. Download the CDN logs from Adobe Cloud Manager.

1. Place the CDN logs inside the environment-specific folders. For example, if you have logs for `dev` environment then place them inside `logs/dev` folder. The logs should have `.log` extension.

1. Make sure that the ELK Docker containers are running.

1. Open the desired dashboard and select the time range for which you want to analyze the logs.

    ![CDN Cache Ratio Analysis](images/cache-ratio-analysis.png)

1. Based on how many logs you provided, data might still be loading. Make sure that an appropriate interval is selected from the time filter on the top-right side so your logs are covered. Also, select one of the environments from the drop-down list by locating on your screen the filter with text `aem_env_name: Please make a choice`.

## Troubleshooting

If you would like to start with a fresh setup, follow these steps:

1. Run `docker compose rm -v` from the repo directory (that is `AEMCS-CDN-Log-Analysis-ELK-Tool`).

1. Remove the `logs/.sincedb` file.

1. Follow again the steps in the [Setup Guide](#how-to-setup-the-elk-docker-container) above.

If you would like to change the port number for Kibana, you can do so by editing the [compose.yml](compose.yaml#L43) file and changing the value of `ports`  under the `kibana` section.

## How to stop the ELK Docker container

There are various ways to stop the ELK Docker container. You can use any of the following methods.

- Using Docker command.

    ```shell
    $ cd AEMCS-CDN-Log-Analysis-ELK-Tool
    $ make stop
    ```

- Using a Docker Desktop application.

- Control+C on the console where you started the ELK Docker container.

