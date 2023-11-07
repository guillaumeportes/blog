# Game infrastructure management with ECS and Docker Compose

#aws #ecs #docker #common-lisp #dynamodb #redis #bash #devops #gamedev

## Introduction

In this post I will try to describe in detail our server infrastructure for Mana Storm, and how we manage it.

Mana Storm is a server authoritative multiplayer (1v1) card game, where players battle synchronously in turn based matches ([https://manastorm.tinka.games/play](https://manastorm.tinka.games/play)).

## Requirements

While Mana Storm is a relatively simple game (from the infrastructure point of view), it does have some particularities, and we wanted to be able to achieve the following:

* Run Common Lisp code, as this is our language of choice for the game backend
* Ability to run the server setup locally as closely as possible to the cloud one
* HTTPS support (for REST services)
* WSS support (for WebSocket communication)
* Ability to easily connect to the running instance (even though it is serverless!)

We also wanted to keep things as simple as possible, and under control, i.e. avoid third party dependencies where appropriate.

## Infrastructure overview

At a high level our server infrastructure is as follows:

* It runs on AWS services
* Server code runs on [ECS (Elastic Container Services)](https://aws.amazon.com/ecs/)/[Fargate](https://aws.amazon.com/fargate/) via [Docker](https://www.docker.com/) containers
* Player data is stored in [DynamoDB](https://aws.amazon.com/dynamodb/)
* Client-server communication is done through [Elasticache](https://aws.amazon.com/elasticache/) (running [Redis](https://redis.io/)). That is, clients connect to the server via WebSockets, and messages are passed back and forth through [Redis Pub/Sub](https://redis.io/docs/interact/pubsub/)

At Tinka we have been using AWS for a long time, and live-operated a relatively big game on it (BattleHand, which peaked at around 300K DAUs), so the cloud provider was a pretty straightforward decision. That title also used DynamoDB and Redis, and we saw no reason to switch to other software.

With BattleHand, server code was a monolithic Java application (running on the [Play Framework](https://www.playframework.com/)), which we deployed on EC2 instances. Those instances were based on an custom image. The pipeline was orchestrated by [Ansible](https://www.ansible.com/) scripts: instances, load balancers, auto-scaling groups, etc. were deployed and updated through those scripts.

This made for a pretty complex setup. As we do not really have a devops or infrastructure team (I see myself as a game programmer first and foremost), I was very keen to simplify things by quite a bit.

More automated solutions included [Lambda](https://aws.amazon.com/lambda/) and [Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/), but did not satisfy one important requirement: I really wanted to develop the game backend using Common Lisp, and those types of solution tend to restrict you in terms of what technology you can use (for instance, Lambda supports Python, C#, Go and a few others, but not Common Lisp, as it is not a mainstream language, to say the least).

With that in mind the solution that seemed to match our requirements the best was a combination of ECS and Docker (to deploy containers running our software) using Fargate (for a serverless "experience"). Indeed, Docker Compose has an ECS integration which essentially translate compose files into [Cloud Formation](https://aws.amazon.com/cloudformation/) templates. Note that this integration is being deprecated soon, but I believe it is simply being "migrated" to another tool (i.e. I do not believe they are retiring the concept of docker compose allowing to control ECS deployments).

## Continuous integration

A major part of the pipeline is of course to automatically build the containers. At the time of writing, it is impossible to build AMD64 containers on an ARM64 architecture (i.e. I cannot build the AWS target containers on my MacBook Pro), or rather the resulting containers constantly crash on startup. Luckily [Docker Hub](https://hub.docker.com/) offers a build service that is very easy to setup. Every time we push to certain branches, containers are built and pushed to the registry with the apprpriate tags. All our script needs to do is to pull the right one when deploying to ECS.

## Software

As I mentioned before, I was keen to keep things as simple as possible (because of previous experiences). I have found that this often means avoiding using third party software (that is when what you want to achieve is simple). With that in mind I figured that the simplest way of administrating our infrastructure was to do it from Bash scripts (one single script actually). The reason behind this is it often makes sense to use the AWS CLI, and the best way to do so is through Bash scripts (I wanted to avoid SDKs such as Boto as that always seems to add yet another layer of complexity and coupling).

Here are the various tasks we wanted to be able to easily run:

* Deploy / update an environment (including the "local" one). This includes setting up the DynamoDB tables, the Redis cluster, etc.
* Tear down an environment
* Handle Route53 setup (i.e. setup the URLs for the game endpoints)
* Open an SSH tunnel to the running container (including the "local" one)
* Connect to the running container (through a shell session)
* Run the application dependencies (DynamoDB and Redis) in order to run outside of Docker

As we use docker containers, a lot of the information is embedded in the Dockerfile or the Docker Compose file. There are however things that are not, and for those we pass them to the Bash script through some sort of *ini* file, containing key value pairs separated by `=`.

Here is the environment file for our `develop` environment:

```
app_cluster=ccg-develop
app_compose=app-compose-develop.yml
ecs=true
redis_nodes=1
redis_node_type=cache.t2.micro
docker_tag=latest
```

The great thing about using Bash is that it forces us to use standard command line tools, which I always forget the syntax.

Parsing this environment file is done this way:

```
parse-env() {
    # $1 = environment file
    vars=("app_cluster" "app_compose" "ecs" "redis_nodes" "redis_node_type" "docker_tag")
    for var in "${vars[@]}"; do
        declare -g $var=$(sed -n "s/^$var=\(.*\)/\1/p" "$1")
    done
}
```

### DynamoDB resources

What we wanted to achieve was relatively standard:

* Declare a list of table specifications
* Check whether those tables exist
* If they do not exist, create them
* Setup auto scaling on the tables
* Support creating those tables locally

To declare the table specifications, we used a Bash array with `/` separated strings, as it is very easy to manipulate strings in Bash.

```
tables=("collection/player-id/card-id" "decks/player-id" "deck/deck-id" "packs/player-id/pack-id")
for table in "${tables[@]}"; do
    # split up table name and attributes
    local table_name="${table%%/*}"
    local attributes="${table#*/}"
    local attribute1="${attributes%/*}"
    local attribute2="${attributes#*/}"
```


To check a table already exists, we use `aws dynamodb describe-table` and check whether the result is empty.

```
local cmd="aws dynamodb describe-table --region eu-west-1 --table-name $table_name"
if [[ "$aws_region" = "local" || "$aws_region" = "" ]]; then
    cmd="$cmd --endpoint-url http://localhost:9000"
fi
local contents=$($cmd)
if [[ "$contents" = "" ]]; then
    # if it doesn't, create it
    ...
```

Creating a table is done through `aws dynamodb create-table`

```
cmd="aws dynamodb create-table --region eu-west-1 --table-name $table_name --attribute-definitions AttributeName=$attribute1,AttributeType=S"
if [[ "$attribute1" != "$attribute2" ]]; then
    cmd="$cmd AttributeName=$attribute2,AttributeType=S"
fi
    cmd="$cmd --key-schema AttributeName=$attribute1,KeyType=HASH"
if [[ "$attribute1" != "$attribute2" ]]; then
    cmd="$cmd AttributeName=$attribute2,KeyType=RANGE"
fi
    cmd="$cmd --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1"
if [[ "$aws_region" = "local" || "$aws_region" = "" ]]; then
    cmd="$cmd --endpoint-url http://localhost:9000"
fi
$cmd > /dev/null
```

I was initially hoping that setting up auto scaling would be a case of simply passing a bunch of parameters to `create-table` (as it is something exposed to the DynamoDB Dashboard), but unfortunately it is a bit more convoluted that that, and requires us to go through the `application-autoscaling` tool:

```
policy_name="$app_cluster"-scaling-policy-"$table"
aws application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:WriteCapacityUnits" --min-capacity 1 --max-capacity 100 > /dev/null
aws application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:ReadCapacityUnits" --min-capacity 1 --max-capacity 100 > /dev/null
aws application-autoscaling put-scaling-policy --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:WriteCapacityUnits" --policy-name "$policy_name"-write --policy-type "TargetTrackingScaling" --target-tracking-scaling-policy-configuration file://scaling-policy-write.json > /dev/null
aws application-autoscaling put-scaling-policy --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:ReadCapacityUnits" --policy-name "$policy_name"-read --policy-type "TargetTrackingScaling" --target-tracking-scaling-policy-configuration file://scaling-policy-read.json > /dev/null
```

`scaling-policy-read.json` and `scaling-policy-write.json` look like this:

```
{
    "PredefinedMetricSpecification": {
        "PredefinedMetricType": "DynamoDBReadCapacityUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 60,
    "TargetValue": 70.0
}
```

```
{
    "PredefinedMetricSpecification": {
        "PredefinedMetricType": "DynamoDBWriteCapacityUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 60,
    "TargetValue": 70.0
}
```

Quite a bit more verbose than I was hoping!

Deleting tables is much more straightforward, as expected, although we still need to manually delete the auto scaling resources. We also check that the table actually exists before proceeding.

```
aws application-autoscaling deregister-scalable-target --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:ReadCapacityUnits" > /dev/null
aws application-autoscaling deregister-scalable-target --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:WriteCapacityUnits" > /dev/null
aws application-autoscaling delete-scaling-policy --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:ReadCapacityUnits" --policy-name "$policy_name"-read > /dev/null
aws application-autoscaling delete-scaling-policy --service-namespace dynamodb --resource-id "table/$table_name" --scalable-dimension "dynamodb:table:WriteCapacityUnits" --policy-name "$policy_name"-write > /dev/null
aws dynamodb delete-table --region eu-west-1 --table-name "$table_name" > /dev/null
```

You can check the whole thing together in this [gist](https://gist.github.com/guillaumeportes/10626e31b4c7d3238f1c32c89ce317e8).

### Redis

Similarly to DynamoDB tables, creating the Redis cluster is a case of first checking whether it exists or not. We also don't have to deal with multiple clusters. The only nuance is that for some environments (the `main` environment, which is what we call our production environment) we want our cluster to have a replication group and be protected in case the main node reboots or fails (in my experience this is very rare, but does happen).

```
create_redis() {
    # first check whether the cluster already exists
    cluster_data=$(aws elasticache describe-replication-groups --replication-group-id "$redis_cluster")
    if [[ "$cluster_data" = "" ]]; then
        echo "Creating redis cluster $redis_cluster."
        if [[ "$redis_nodes" = 1 ]]; then
            # create a node with replication enabled
            aws elasticache create-replication-group --replication-group-id "$redis_cluster" --region eu-west-1 --security-group-ids XXXXXXXXXXXXXXXX --engine redis --cache-node-type "$redis_node_type" --num-cache-clusters "$redis_nodes" --replication-group-description "$redis_cluster" --snapshot-retention-limit 1 > /dev/null
        else
            # create a node without replication
            aws elasticache create-replication-group --replication-group-id "$redis_cluster" --region eu-west-1 --security-group-ids XXXXXXXXXXXXXXXX --engine redis --cache-node-type "$redis_node_type" --num-cache-clusters "$redis_nodes" --multi-az-enabled --automatic-failover-enabled --replication-group-description "$redis_cluster" --snapshot-retention-limit 1 > /dev/null
        fi
    else
        echo "Redis cluster $redis_cluster already exists."
    fi
}
```

Deleting is this time very straightforward.

```
delete_redis() {
    cluster_data=$(aws elasticache describe-replication-groups --replication-group-id "$redis_cluster")
    if [[ "$cluster_data" = "" ]]; then
        echo "No redis cluster found, skipping."
    else
        echo "Deleting the redis cluster."
        aws elasticache delete-replication-group --replication-group-id "$redis_cluster" > /dev/null
    fi
}
```

### Game server container

Our server software is made of only one container. It is a monolithic application handling both HTTP and WS requests from clients.

The compose file has a couple of ECS specific bits:

```
x-aws-role:
  Version: "2012-10-17"
  Statement:
  - Effect: Allow
    Action:
      - ssmmessages:CreateControlChannel
      - ssmmessages:CreateDataChannel
      - ssmmessages:OpenControlChannel
      - ssmmessages:OpenDataChannel
    Resource: "*"
```

This is required to enable "execute command" on the Fargate container. Execute command is required to connect to the running instance through a terminal.

```
x-aws-cloudformation:
  Resources:
    AppTCP80Listener:
      Properties:
        Certificates:
          - CertificateArn: "arn:aws:acm:eu-west-1:XXXXXXXXXXXXXXXX"
        Protocol: TLS
        Port: 443
    AppTCP4000Listener:
      Properties:
        Certificates:
          - CertificateArn: "arn:aws:acm:eu-west-1:XXXXXXXXXXXXXXXX"
        Protocol: TLS
        Port: 4430
```

This bit is to enable HTTPS and WSS on the load balancers.

Deploying and updating the application is done through the Docker Compose / ECS integration, and consists of the following steps:

* Create the Redis cluster if needed
* Create the DynamoDB tables if needed
* Pull the container from Docker Hub
* Push the container to [AWS ECR](https://aws.amazon.com/ecr/)
* Deploy the container to ECS

Version management is handled for us, so we do not need to check whether things have changed or not, `docker compose` does it for us and will not do anything if it does not need to.

When creating the Redis cluster, we need to wait to make sure the operation has finished before proceeding, as it is done asynchronously.

```
get_redis_host_and_port() {
    cluster_data=$(aws elasticache describe-replication-groups --replication-group-id "$redis_cluster")
    if [[ "$cluster_data" = "" ]]; then
        echo "Error: redis cluster $redis_cluster does not exist."
    else
        redis_host=$(echo "$cluster_data" | jq ".ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address")
        redis_host="${redis_host//\"/}"
        redis_port=$(echo "$cluster_data" | jq ".ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Port")
        redis_port="${redis_port//\"/}"
        if [[ "$redis_host" = null ]]; then
            redis_host=""
            redis_port=""
        fi
    fi
}

get_redis_host_and_port
if [[ "$redis_host" = "" ]]; then
    create_redis
    echo "Waiting for redis cluster $redis_cluster to be available"
    while [[ "$redis_host" = "" ]]; do
        get_redis_host_and_port
        sleep 5
    done
fi
```

Pulling the container from the Hub and pushing it to ECR is quite straightforward:

```
ecr=XXXXXXXXXX.dkr.ecr.eu-west-1.amazonaws.com
containers=(ccg-server-app)

docker login
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin "$ecr"
for container in "${containers[@]}"; do
    docker pull anotherplace/"$container":"$docker_tag"
    docker tag anotherplace/"$container":"$docker_tag" "$ecr"/"$container":"$docker_tag"
    docker push "$ecr"/"$container":"$docker_tag"
done
```

And deploying is done via `docker compose`, as mentioned above:

```
(export REDIS_HOST="$redis_host"; export REDIS_PORT="$redis_port"; export DYNAMODB_PREFIX="$app_cluster"; docker --context myecscontext compose --project-name "$app_cluster" -f "$app_compose" up)
route53 UPSERT "$app_cluster" "AppService" 80
enable_exec "$app_cluster"
```

Variables are `export`ed before calling `docker compose up` in order for the compose file to be able to access them. Note that docker is run with `--context myecscontext`, which is simply a context that needs to be created upfront (as described [here](https://docs.docker.com/cloud/ecs-integration/)).

`enable_exec` is called in order to enable Execute Command on the container, which will allow us to connect into it for debugging. Weirdly we need to force a new deployment although it was tagged as enabled in the compose file.

```
enable_exec() {
    # $1 = cluster
    services=$(aws ecs list-services --cluster "$1" | jq '.serviceArns[]')
    services=($services)
    for service in "${services[@]}"; do
        service="${service//\"/}"
        enabled=$(aws ecs describe-services --cluster "$1" --services "$service" | jq ".services[0].enableExecuteCommand")
        if [[ "$enabled" = true ]]; then
            echo "execute-command already enabled on service \"$service\""
        else
            echo "Enabling execute-command on service \"$service\""
            aws ecs update-service --cluster "$1" --service "$service" --enable-execute-command --force-new-deployment > /dev/null
        fi
    done
}
```

### Route 53 setup

We also need to setup the Route 53 URIs, i.e. point https://manastorm.tinka.games to the new container's load balancer. Unfortunately this is a bit more convoluted than I felt it should be.

First, we need to get the hosted zone id from a hosted zone name. In our case, the name is `tinka.games`.

```
zone=tinka.games
zones=$(aws route53 list-hosted-zones | jq ".HostedZones[].Name")
zones=($zones)
zone_ids=$(aws route53 list-hosted-zones | jq ".HostedZones[].Id")
zone_ids=($zone_ids)
# find the zone that matches ours, and get the corresponding id
for (( i=0; i<${#zones[@]}; i++ )); do
    if [[ "\"$zone.\"" = "${zones[$i]}" ]]; then
        zone_id="${zone_ids[$i]}"
    fi
done
zone_id="${zone_id//\/hostedzone\//}"
zone_id="${zone_id//\"/}"
```

We then need to return the load balancer associated with our ECS cluster. This is done by iterating through the cluster's services and finding the one that corresponds to the right port. We then return the associated load balancer ARN.

```
load-balancer() {
    # $1 = cluster
    # $2 = service
    # $3 = port
    services=$(aws ecs list-services --cluster "$1" | jq '.serviceArns[]')
    services=($services)
    for service in "${services[@]}"; do
        service="${service//\"/}"
        if [[ $service =~ .*"$2".* ]]; then
            groups=$(aws ecs describe-services --cluster "$1" --services "$service" | jq '.services[0].loadBalancers[].targetGroupArn')
            groups=($groups)
            ports=$(aws ecs describe-services --cluster "$1" --services "$service" | jq '.services[0].loadBalancers[].containerPort')
            ports=($ports)
            for (( i=0; i<${#groups[@]}; i++ )); do
                if [[ "${ports[$i]}" = "$3" ]]; then
                    group="${groups[$i]//\"/}"
                    elb=$(aws elbv2 describe-target-groups --target-group-arns "$group" | jq '.TargetGroups[0].LoadBalancerArns[0]')
                    echo "${elb//\"/}"
                fi
            done
        fi
    done
}
```

Finally, we figure out the load balancer's DNS, its hosted zone id, and we generate the input for the route 53 tool. This input is generated from a JSON template based on this:

```
{
    "HostedZoneId": "XXXXXXXXXXXXXXXX",
    "ChangeBatch": {
        "Changes": [
            {
                "Action": "UPSERT",
                "ResourceRecordSet": {
                    "Name": "manastorm.tinka.games",
                    "Type": "A",
                    "AliasTarget": {
                        "HostedZoneId": "XXXXXXXXXXXXXXXX",
                        "DNSName": "gameb-loadb-XXXXXXXXXXXXXXXX.eu-west-1.amazonaws.com",
                        "EvaluateTargetHealth": true
                    }
                }
            }
        ]
    }
}
```

Here are the missing bits:

```
dns=$(aws elbv2 describe-load-balancers --load-balancer-arns "$elb" | jq '.LoadBalancers[0].DNSName')
elb_zoneid=$(aws elbv2 describe-load-balancers --load-balancer-arns "$elb" | jq '.LoadBalancers[0].CanonicalHostedZoneId')
json=$(cat record.json | jq ".ChangeBatch.Changes[0].ResourceRecordSet.Name = \"$cluster.tinka.games\"" | jq ".ChangeBatch.Changes[0].ResourceRecordSet.AliasTarget.DNSName = $dns" | jq ".ChangeBatch.Changes[0].ResourceRecordSet.AliasTarget.HostedZoneId = $elb_zoneid" | jq ".ChangeBatch.Changes[0].Action = $action" | jq ".HostedZoneId = \"$zoneid\"")
aws route53 change-resource-record-sets --hosted-zone-id "/hostedzone/$zone_id" --cli-input-json "$json" > /dev/null
```

You can look at the whole thing on this [gist](https://gist.github.com/guillaumeportes/35f2d17b965e370337d0705b79c4fbda).

### Logging in

From time to time it is necessary to connect to the running machine. Logs are not always enough. Of course, this is trivial when the software is running on actual instances. With Fargate, there are no instances, so it is a bit more complicated, but with the following bit of code, it becomes trivial.

```
login() {
    container="$login_container"
    if [[ "$container" = "" ]]; then
        container=app
    fi
    tasks=$(aws ecs list-tasks --cluster "$app_cluster" | jq '.taskArns[]')
    tasks=($tasks)
    for task in "${tasks[@]}"; do
        names=$(aws ecs describe-tasks --cluster "$app_cluster" --tasks "${task//\"/}" | jq '.tasks[0].containers[].name')
        names=($names)
        arns=$(aws ecs describe-tasks --cluster "$app_cluster" --tasks "${task//\"/}" | jq '.tasks[0].containers[].containerArn')
        arns=($arns)
        for (( i=0; i<${#names[@]}; i++ )); do
            if [[ "${names[$i]//\"/}" = "$container" ]]; then
                aws ecs execute-command --region eu-west-1 --cluster "$app_cluster" --task "${task//\"/}" --container "$container" --command /bin/sh --interactive
                return
            fi
        done
    done
}
```

As you can see, we need to figure out which task in the cluster is running the container we want to log into (in this case, usually "app"). We then use `aws ecs execute-command` to run "sh" on it. Et voilà !

### Opening an SSH tunnel

The final piece is incredibly useful when using Common Lisp. Common Lisp encourages interactive style development, whereby one always has a REPL (read-eval-print-loop) within which one constantly tries functions, compiles code, etc. The beauty of it is that with setting up an SSH tunnel, we can connect our REPL to the running instance of a container, and debug the live software. When using [Emacs](https://www.gnu.org/software/emacs/), you can even setup TRAMP in order to navigate the source code as if it was on your own machine.

The steps to do so are as follows:

* Iterate through the cluster's tasks, and find the one running the container with the matching name
* Find the network interface details from the task info
* Gather all network interfaces from EC2
* Iterate through them and find the one that matches the one we got from the task info
* Get the public IP from the network interface
* Open the SSH tunnel

The function is quite chunky:

```
ssh-tunnel() {
    if [[ "$ecs" = true ]]; then
        container=app
        tasks=$(aws ecs list-tasks --cluster "$app_cluster" | jq '.taskArns[]')
        tasks=($tasks)
        for task in "${tasks[@]}"; do
            names=$(aws ecs describe-tasks --cluster "$app_cluster" --tasks "${task//\"/}" | jq '.tasks[0].containers[].name')
            names=($names)
            arns=$(aws ecs describe-tasks --cluster "$app_cluster" --tasks "${task//\"/}" | jq '.tasks[0].containers[].containerArn')
            arns=($arns)
            for (( i=0; i<${#names[@]}; i++ )); do
                if [[ "${names[$i]//\"/}" = "$container" ]]; then
                    details_names=$(aws ecs describe-tasks --cluster "$app_cluster" --tasks "${task//\"/}" | jq '.tasks[0].attachments[0].details[].name')
                    details_names=($details_names)
                    details_values=$(aws ecs describe-tasks --cluster "$app_cluster" --tasks "${task//\"/}" | jq '.tasks[0].attachments[0].details[].value')
                    details_values=($details_values)
                    for (( j=0; j<${#details_names[@]}; j++ )); do
                        if [[ "${details_names[$j]//\"/}" = networkInterfaceId ]]; then
                            interface_ids=$(aws ec2 describe-network-interfaces | jq '.NetworkInterfaces[].NetworkInterfaceId')
                            interface_ids=($interface_ids)
                            for (( k=0; k<${#interface_ids[@]}; k++ )); do
                                if [[ "${interface_ids[$k]//\"/}" = "${details_values[$j]//\"/}" ]]; then
                                    public_ip=$(aws ec2 describe-network-interfaces | jq ".NetworkInterfaces[$k].Association.PublicIp")
                                    rm -f ssh_config
                                    echo "Host ccg-server" >> ssh_config
                                    echo "  HostName ${public_ip//\"/}" >> ssh_config
                                    echo "  User root" >> ssh_config
                                    echo "  IdentityFile ~/ccg-server/id_rsa" >> ssh_config
                                    command1="ssh -L 5000:127.0.0.1:4005 root@${public_ip//\"/} -N -i id_rsa -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
                                    command2="ssh -L 5001:127.0.0.1:4006 root@${public_ip//\"/} -N -i id_rsa -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
                                    echo "$command1"
                                    echo "$command2"
                                    (trap 'rm -f ssh_config; kill 0' SIGINT; ($command1) & ($command2))
                                    return
                                fi
                            done
                        fi
                    done
                fi
            done
        done
    else
        # this is to connect to a local Docker container
        rm -f ssh_config
        echo "Host ccg-server" >> ssh_config
        echo "  HostName localhost" >> ssh_config
        echo "  User root" >> ssh_config
        echo "  IdentityFile ~/ccg-server/id_rsa" >> ssh_config
        command1="ssh -L 5000:127.0.0.1:4005 root@127.0.0.1 -N -i id_rsa -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
        command2="ssh -L 5001:127.0.0.1:4006 root@127.0.0.1 -N -i id_rsa -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no"
        echo "$command1"
        echo "$command2"
        (trap 'rm -f ssh_config; kill 0' SIGINT; ($command1) & ($command2))
    fi
}
```

We run two ssh tunnels in parallel in order to support two different REPLs at the same time ([SLIME](https://slime.common-lisp.dev/) and [SLY](https://github.com/joaotavora/sly)) on different ports.

Of course, in order to open SSH tunnels, the container needs to run an SSH server and be setup with the right key.

```
RUN apt-get install cmake git sudo libboost-all-dev openssh-server -y
RUN mkdir /root/.ssh
COPY /id_rsa.pub /root/.ssh/authorized_keys

CMD "src/server/cmd.sh"
```

With `cmd.sh` containing the following:

```
service ssh start
/usr/sbin/sshd -D
```

### Running locally

As you have seen from the bits of code in this post, we handle local containers quite easily.

We do also support running the application locally without containers. This actually means, in our case, simply running the AWS services that we use locally, i.e. DynamoDB and Redis. This is of course super simple, and is just about running local containers with DynamoDB and Redis on them.

```
run() {
    (trap 'kill 0' SIGINT;
     (docker compose --project-name dynamodb-local -f dynamodb-compose.yml up) &
     (docker compose --project-name redis-local -f redis-compose.yml up))
}
```

With `dynamodb-compose.yml` looking like this:

```
version: "3.9"

services:
  dynamo:
    image: amazon/dynamodb-local:latest
    ports:
      - "9000:9000"
    volumes:
      - ./dynamodb-data:/home/dynamodblocal/data
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath /home/dynamodblocal/data/ -port 9000"
```

And `redis-compose.yml` like that:

```
version: "3.9"

services:
  dynamo:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - ./redis-data:/var/lib/redis
      - ./redis-conf:/use/local/etc/redis/redis.conf
    command: "redis-server"

```

## Conclusion

As you can see, we have something that handles quite a few things automatically. Writing it ourselves means that we gained a better understanding of the inner workings of ECS and Docker Compose. Finally learning Bash properly (through [this fantastic guide](http://mywiki.wooledge.org/BashGuide) as well as [Exercism](https://exercism.org/dashboard)) was also very satisfying.

More importantly, we have something that works and that we control and understand, which I think is invaluable as it means failures are identified and dealt with much more easily.
