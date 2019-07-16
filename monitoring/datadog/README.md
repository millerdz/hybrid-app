# Enable DataDog Monitoring

Get your `DD_API_KEY` from your [DataDog account](https://app.datadoghq.com/account/settings#api).

## Create Swarm Overlay Network for DataDog Agent and Your Apps to Communicate

1. Go to UCP
1. Log in as `admin`
1. Go to `Swarm` > `Networks`
1. Click `Create`
1. Name the network `datadog-agent`
1. Enable checkbox for `Allow any container to attach to this network`
1. Click `Create`

## Deploy DataDog Agent for Infrastructure

For each `worker1`, `worker2`, `worker3`, `manager1`, copy the below command, substitute `<YOUR_DD_API_KEY>` with your API KEY, then run the following command in the terminal:

```DOCKER_CONTENT_TRUST=1 \
docker run -d --name datadog-agent \
              --network datadog-agent \
              -v /var/run/docker.sock:/var/run/docker.sock:ro \
              -v /proc/:/host/proc/:ro \
              -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
              -e DD_API_KEY=<YOUR_DD_API_KEY> \
              -e DD_APM_ENABLED=true \
              -e DD_APM_NON_LOCAL_TRAFFIC=true \
              datadog/agent:latest
```

## Get the DataDog Java Agent, Modify pom.xml and Dockerfile to enable DataDog APM

1. On `worker1`, run the command:
    ```
    mkdir -p ~/hybrid-app/java-app/apm && wget -O ~/hybrid-app/java-app/apm/dd-java-agent.jar 'https://repository.sonatype.org/service/local/artifact/maven/redirect?r=central-proxy&g=com.datadoghq&a=dd-java-agent&v=LATEST'
    ```
1. `cp -vf ~/hybrid-app/monitoring/datadog/java-app/Dockerfile ~/hybrid-app/java-app/`
1. `cp -vf ~/hybrid-app/monitoring/datadog/java-app/pom.xml ~/hybrid-app/java-app/app/`
1. `cd ~/hybrid-app/java-app`
1. `docker image build --no-cache -t $DTR_HOST/java/java_web:latest-datadog .`
1. `docker login -u java_user $DTR_HOST`
1. `docker push $DTR_HOST/java/java_web`

## Deploy the Updated Application

1. Log in to UCP
1. Click `Shared Resources` > `Stacks`
1. Click `Create`
1. Enter the same name as the currently deployed app
1. Click `Next`
1. Copy the below `docker-compose.yml` into the section `2. Add Application File` of UCP Stacks window above and replace `<dtr hostname>` with the value of `$DTR_HOST`
    ```
    version: "3.3"

    services:

    database:
        image: <dtr hostname>/java/database
        # set default mysql root password, change as needed
        environment:
        MYSQL_ROOT_PASSWORD: mysql_password
        # Expose port 3306 to host. 
        ports:
        - "3306:3306" 
        networks:
        - back-tier
        - datadog-agent

    webserver:
        image: <dtr hostname>/admin/java_web:latest-datadog
        ports:
        - "8080:8080" 
        networks:
        - front-tier
        - back-tier
        - datadog-agent

    networks:
    back-tier:
        external: true
    front-tier:
        external: true
    datadog-agent:
        external: true

    secrets:
    mysql_password:
        external: true
    ```
1. Click `Create`
1. Make sure the application is up and running and you can create a new account:
    1. Go to `<UCP_URL>:8080/java-web`
    1. Click `Signup`
    1. Enter the name, email, date details, `Submit`, then confirm
    1. Click `Login`
    1. Enter your newly created `username` and `password` to login
1. Go to your your DataDog APM view at [https://app.datadoghq.com/apm/services](https://app.datadoghq.com/apm/services) to view the APM results. This may take a minute or two to show up.
