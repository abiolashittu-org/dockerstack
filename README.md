```markdown
# Dive Deeper into Docker: Creating a Docker Stack

![image](https://github.com/shittuay/docker-stack/assets/108594160/947ceee4-b3aa-4c63-ba6a-70a17ed66f88)


This is a continuation from the last article, where we created a three-tier architecture using AWS and Docker Swarm, featuring:
- A service based on the Redis Docker image with 4 replicas.
- A service based on the Apache Docker image with 10 replicas.
- A service based on the Postgres Docker image with 1 replica.

## Tasks
- Create a Docker Stack using the basic project requirements.
- Ensure no stacks run on the manager (administrative) node.

## What is a Docker Stack?

A Docker stack is a way to deploy a group of Docker services together as a single unit. It helps organize and manage containers across multiple machines as one. In order to do this, we will use a YAML file called a compose file. In this file, we will specify:
- The images to use for each service.
- The number of replicas we need for each service.
- Other options such as environment variables, ports, or volume mounts.

## Why Don’t We Want Docker Stacks on the Manager Node?

- **Resource Management:** The manager node is responsible for coordinating the swarm (and managing the worker nodes). Deploying stacks on the manager node can use up resources needed for these tasks.
- **Failure Risk:** If the manager node fails, it will cause all other nodes to fail. By placing the stacks on the worker nodes, it minimizes the chance of a single point of failure for the entire swarm.
- **Security:** The manager node generally holds sensitive information related to the swarm, such as encryption keys, passwords, API keys, or other information related to managing the swarm.

To ensure no stacks run on the manager node, we will use placement constraints to specify that the services are deployed on worker nodes only.

## Docker Stack File

Below is the Docker stack file we will use, with explanations and alternatives for some of the parts:

```yaml
version: '3'

services:
  redis:
    image: redis
    deploy:
      replicas: 4
      placement:
        constraints:
          - node.role == worker
    ports:
      - "6379:6379"
      
  postgres:
    image: postgres
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == worker
    environment:
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
      
  apache:
    image: httpd
    deploy:
      replicas: 10
      placement:
        constraints:
          - node.role == worker
    ports:
      - "8080:80"
```

For placement constraints, we could also use `node.role != manager` to specify that it should not be a manager node:
```yaml
node.role != manager
```

We can also use labels as a placement constraint. For example, to ensure a service is deployed only to nodes with a specific label:
```yaml
node.labels.type==xxxx
```

## Deploying the Stack

To deploy this stack, we will use the following command:
```bash
docker stack deploy -c docker-compose.yml abistack
```
I have named my stack ‘abistack’ but you can change that to suit your purposes.

First, we must copy our file to one of the EC2 instances in the swarm. We will use the `scp` (secure copy) command to transfer the file from your local machine to the remote EC2 instance:

```bash
scp -i /path/to/your/key.pem /path/to/your/docker-compose.yml ec2-user@your-ec2-instance:/home/ec2-user/docker-compose.yml
```

This will copy our `docker-compose.yml` file to the home directory of the EC2 user account on the AWS EC2 instance.

We must also make sure our Docker swarm from the last article is not still running. We can remove these services one by one with the following commands on the manager node:
```bash
docker service rm redis
docker service rm apache
docker service rm postgres
```

We will then remove all unused resources by running:
```bash
docker system prune
```

We will now deploy our stack and check it with `docker stack ls` and `docker stack ps abistack` to see all of our services. Everything should look good, with 10 Apache, 1 Postgres, and 4 Redis instances. 

Finally, run `docker node ls` to verify that none of the services are running on the manager node’s IP (in our case it is 172.31.62.52).

## Conclusion

Thank you for reading! In this project, we created a Docker Compose file with three services, each based on a different Docker image, and deployed the stack using the `docker stack deploy` command. 

We also ensured that no stacks were running on the manager (administrative) node by using placement constraints in our Docker Compose file to specify that our services should run only on worker nodes, and provided some other methods for doing so.



---

**Tags:** Docker, Docker Compose, Container Orchestration, Containerization
```
