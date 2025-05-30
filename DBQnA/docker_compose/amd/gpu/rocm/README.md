# Deploying DBQnA on AMD ROCm GPU

This document outlines the deployment process for DBQnA application which helps generating a SQL query and its output given a NLP question, utilizing the [GenAIComps](https://github.com/opea-project/GenAIComps.git) microservice pipeline on an AMD GPU. The steps include Docker image creation, container deployment via Docker Compose, and service execution to integrate microservices. We will publish the Docker images to Docker Hub soon, which will simplify the deployment process for this service.

Note: The default LLM is `mistralai/Mistral-7B-Instruct-v0.3`. Before deploying the application, please make sure either you've requested and been granted the access to it on [Huggingface](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3) or you've downloaded the model locally from [ModelScope](https://www.modelscope.cn/models).

## Table of Contents

1. [DBQnA Quick Start Deployment](#dbqna-quick-start-deployment)
2. [DBQnA Docker Compose Files](#dbqna-docker-compose-files)
3. [Validate Microservices](#validate-microservices)
4. [Conclusion](#conclusion)

## DBQnA Quick Start Deployment

This section describes how to quickly deploy and test the DBQnA service manually on an AMD ROCm GPU. The basic steps are:

1. [Access the Code](#access-the-code)
2. [Configure the Deployment Environment](#configure-the-deployment-environment)
3. [Deploy the Services Using Docker Compose](#deploy-the-services-using-docker-compose)
4. [Check the Deployment Status](#check-the-deployment-status)
5. [Validate the Pipeline](#validate-the-pipeline)
6. [Cleanup the Deployment](#cleanup-the-deployment)

### Access the Code

Clone the GenAIExample repository and access the DBQnA AMD ROCm GPU platform Docker Compose files and supporting scripts:

```bash
git clone https://github.com/opea-project/GenAIExamples.git
cd GenAIExamples/DBQnA
```

Then checkout a released version, such as v1.3:

```bash
git checkout v1.3
```

### Configure the Deployment Environment

To set up environment variables for deploying DBQnA services, set up some parameters specific to the deployment environment and source the `set_env.sh` script in this directory:

Set the values of the variables:

- **HOST_IP, HOST_IP_EXTERNAL** - These variables are used to configure the name/address of the service in the operating system environment for the application services to interact with each other and with the outside world.

  If your server uses only an internal address and is not accessible from the Internet, then the values for these two variables will be the same and the value will be equal to the server's internal name/address.

  If your server uses only an external, Internet-accessible address, then the values for these two variables will be the same and the value will be equal to the server's external name/address.

  If your server is located on an internal network, has an internal address, but is accessible from the Internet via a proxy/firewall/load balancer, then the HOST_IP variable will have a value equal to the internal name/address of the server, and the EXTERNAL_HOST_IP variable will have a value equal to the external name/address of the proxy/firewall/load balancer behind which the server is located.

  We set these values in the file set_env.sh

- **Variables with names like "**\*\*\*\*\*\*\_PORT"\*\* - These variables set the IP port numbers for establishing network connections to the application services.
  The values shown in the file set_env.sh or set_env_vllm.sh they are the values used for the development and testing of the application, as well as configured for the environment in which the development is performed. These values must be configured in accordance with the rules of network access to your environment's server, and must not overlap with the IP ports of other applications that are already in use.

Setting variables in the operating system environment:

```bash
export HUGGINGFACEHUB_API_TOKEN="Your_HuggingFace_API_Token"
source ./set_env_*.sh # replace the script name with the appropriate one
```

Consult the section on [DBQnA Service configuration](#dbqna-configuration) for information on how service specific configuration parameters affect deployments.

### Deploy the Services Using Docker Compose

To deploy the DB services, execute the `docker compose up` command with the appropriate arguments. For a default deployment with TGI, execute the command below. It uses the 'compose.yaml' file.

```bash
cd docker_compose/amd/gpu/rocm
docker compose -f compose.yaml up -d
```

To enable GPU support for AMD GPUs, the following configuration is added to the Docker Compose file:

- compose_vllm.yaml - for vLLM-based application
- compose_faqgen_vllm.yaml - for vLLM-based application with FaqGen
- compose.yaml - for TGI-based
- compose_faqgen.yaml - for TGI-based application with FaqGen

```yaml
shm_size: 1g
devices:
  - /dev/kfd:/dev/kfd
  - /dev/dri:/dev/dri
cap_add:
  - SYS_PTRACE
group_add:
  - video
security_opt:
  - seccomp:unconfined
```

This configuration forwards all available GPUs to the container. To use a specific GPU, specify its `cardN` and `renderN` device IDs. For example:

```yaml
shm_size: 1g
devices:
  - /dev/kfd:/dev/kfd
  - /dev/dri/card0:/dev/dri/card0
  - /dev/dri/render128:/dev/dri/render128
cap_add:
  - SYS_PTRACE
group_add:
  - video
security_opt:
  - seccomp:unconfined
```

**How to Identify GPU Device IDs:**
Use AMD GPU driver utilities to determine the correct `cardN` and `renderN` IDs for your GPU.

> **Note**: developers should build docker image from source when:
>
> - Developing off the git main branch (as the container's ports in the repo may be different > from the published docker image).
> - Unable to download the docker image.
> - Use a specific version of Docker image.

Please refer to the table below to build different microservices from source:

| Microservice       | Deployment Guide                                                                                                   |
| -------------------| ------------------------------------------------------------------------------------------------------------------ |
| TGI                | [TGI project](https://github.com/huggingface/text-generation-inference.git)                                        |
| postgres           | [postgres](https://hub.docker.com/_/postgres)                                                                      |
| text2sql           | [text2sql](https://github.com/opea-project/GenAIComps/blob/main/comps/text2sql/src/README.md)                      |
| text2sql-react-ui  | [text2sql-react-ui](../../../../react/README.md)                                                                   |


### Check the Deployment Status

After running docker compose, check if all the containers launched via docker compose have started:

```bash
docker ps -a
```

For the default deployment with TGI, the following 4 containers should have started:

```

```

### Validate the Pipeline


We test the API in frontend validation to check if API returns HTTP_STATUS: 200 and validates if API response returns SQL query and output

The test is present in App.test.tsx under react root folder ui/react/

Command to run the test

```bash
npm run test
```

### 🚀 Launch the React UI

Open this URL `http://${host_ip}:${DBQNA_UI_PORT}` in your browser to access the frontend.

![project-screenshot](../../../../assets/img/dbQnA_ui_init.png)

Test DB Connection
![project-screenshot](../../../../assets/img/dbQnA_ui_successful_db_connection.png)

Create SQL query and output for given NLP question
![project-screenshot](../../../../assets/img/dbQnA_ui_succesful_sql_output_generation.png)

### Cleanup the Deployment

To stop the containers associated with the deployment, execute the following command:

```bash
# if used TGI
docker compose -f compose.yaml down
```

## DBQnA Docker Compose Files


| File                                                   | Description                                                                                                              |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| [compose.yaml](./compose.yaml)                         | The LLM serving framework is TGI. Default compose file using TGI as serving framework and redis as vector database       |


## Validate MicroServices


1. TGI Service

```bash
curl --location http://${host_ip}:${DBQNA_TEXT_TO_SQL_PORT}/v1/postgres/health \
    --header 'Content-Type: application/json' \
    --data '{"user": "'${POSTGRES_USER}'","password": "'${POSTGRES_PASSWORD}'","host": "'${host_ip}'", "port": "5442", "database": "'${POSTGRES_DB}'"}'
```

2. Test the Database connection

```bash
curl --location http://${host_ip}:${DBQNA_TEXT_TO_SQL_PORT}/v1/postgres/health \
    --header 'Content-Type: application/json' \
    --data '{"user": "'${POSTGRES_USER}'","password": "'${POSTGRES_PASSWORD}'","host": "'${host_ip}'", "port": "5442", "database": "'${POSTGRES_DB}'"}'
```

3. text2SQL Service 

```bash
curl http://${host_ip}:${DBQNA_TEXT_TO_SQL_PORT}/v1/texttosql \
    -X POST \
    -d '{"input_text": "Find the total number of Albums.","conn_str": {"user": "'${POSTGRES_USER}'","password": "'${POSTGRES_PASSWORD}'","host": "'${host_ip}'", "port": "5442", "database": "'${POSTGRES_DB}'"}}' \
    -H 'Content-Type: application/json'
```

## Conclusion

This guide should enable developers to deploy the default configuration or any of the other compose yaml files for different configurations. It also highlights the configurable parameters that can be set before deployment.

