# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

Healthy deploy (milestone 1):

```
------------------------------------------------------------------------
|                            DescribeStacks                            |
+------------+---------------------------------------------------------+
|  InstanceId|  i-0a10f0616022ede9d                                    |
|  ServiceUrl|  http://ec2-54-242-45-47.compute-1.amazonaws.com:8080   |
+------------+---------------------------------------------------------+
```

Scenario 2 (broken deploy):

```
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-01c6b4b369f27f485                                     |
|  ServiceUrl|  http://ec2-54-221-96-117.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

Healthy redeploy after scenario 2 (infrastructure fix):

```
--------------------------------------------------------------------------
|                             DescribeStacks                             |
+------------+-----------------------------------------------------------+
|  InstanceId|  i-06db455544b9525da                                      |
|  ServiceUrl|  http://ec2-34-235-154-139.compute-1.amazonaws.com:8080   |
+------------+-----------------------------------------------------------+
```

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

```
$ curl http://ec2-54-242-45-47.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

Three or four sentences, your own words. What compute, what network access, and what
glue made the service start.

The template created one t3.micro EC2 instance on Amazon Linux 2023 and a security group that opens inbound TCP 8080 (the service port) and 22 (SSH fallback) from the internet. The instance uses the Learner Lab IAM instance profile `LabInstanceProfile` so SSM sessions work, and the `vockey` key pair as a backup. On first boot, UserData installs Docker, pulls the public `lab04-service` image, and runs the container with host port 8080 forwarded to the app, which is what makes GET /api/health reachable at the stack's ServiceUrl. A four-hour `shutdown` is also scheduled so an abandoned instance stops burning credits.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

```
$ curl http://ec2-54-221-96-117.compute-1.amazonaws.com:8080/api/health
curl: (28) Connection timed out after 15004 milliseconds
```

**The log line that told you what was wrong:**

```
$ aws ssm start-session --target i-01c6b4b369f27f485

$ sudo docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS                                       NAMES
16138d886956   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service

$ sudo docker logs lab04-service
lab04-service listening on 9090
```

**What was wrong, and the fix you applied:**

`params-scenario2.json` set `PortOverride` to 9090, so UserData ran the container with `-e PORT=9090` while still publishing host port 8080 (`0.0.0.0:8080->8080/tcp`). The app therefore bound 9090 (`lab04-service listening on 9090`), and traffic that the security group and Docker mapping sent to 8080 never reached it. The fix was to delete the broken stack and recreate it with `infra/params-healthy.json` (`PortOverride` empty) so the process listens on 8080, matching the published port. I did not patch the running container, because that would leave the stack describing a different deploy than what was actually running.

**The healthy curl after the fix:**

```
$ curl http://ec2-34-235-154-139.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

Final teardown after the healthy redeploy (milestone 3):

```
$ aws cloudformation delete-stack --stack-name lab04-service
$ aws cloudformation wait stack-delete-complete --stack-name lab04-service
DELETE_COMPLETE

$ aws cloudformation describe-stacks --stack-name lab04-service
An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```
