# gpu-jupyter
#### Leverage the power of Jupyter and use your NVIDEA GPU and use Tensorflow and Pytorch in collaborative notebooks. 

![Jupyterlab Overview](/extra/jupyterlab-overview.png)

## Contents

1. [Requirements](#requirements)
2. [Quickstart](#quickstart)
3. [Deployment](#deployment-in-the-docker-swarm)


## Requirements

1.  Install [Docker](https://www.docker.com/community-edition#/download) version **1.10.0+**
2.  Install [Docker Compose](https://docs.docker.com/compose/install/) version **1.6.0+**

3.  Get access to use your GPU via the CUDA drivers, see this [blog-post](https://medium.com/@christoph.schranz)
4.  Clone this repository
    ```bash
    git clone https://github.com/iot-salzburg/gpu-jupyter.git
    cd gpu-jupyter
    ```

## Quickstart

As soon as you have access to your GPU locally (it can be tested via a Tensorflow or PyTorch), you can run these commands to start the jupyter notebook via docker-compose:
  ```bash
  ./start-local.sh
  ```
  
This will run *gpu-jupyter* on the default port [localhost:8888](http://localhost:8888). The general usage is:
  ```bash
  ./start-local.sh -p [port]  # port must be an integer with 4 or more digits.
  ```
In order to stop the local deployment, run:

  ```bash
  ./stop-local.sh
  ```
 
 ## Deployment in the Docker Swarm
 
A Jupyter instance often requires data from other services. 
If that data-source is containerized in Docker and sharing a port for communication shouldn't be allowed, e.g., for security reasons,
then connecting the data-source with *gpu-jupyter* within a Docker Swarm is a great option! \

### Set up a Docker Swarm

This step requires a running [Docker Swarm](https://www.youtube.com/watch?v=x843GyFRIIY) on a cluster or at least on this node.
In order to register custom images in a local Docker Swarm cluster, 
a registry instance must be deployed in advance.
Note that the we are using the port 5001, as many services use the default port 5000.

```bash
sudo docker service create --name registry --publish published=5001,target=5000 registry:2
curl 127.0.0.1:5001/v2/
```
This should output `{}`. \

Afterwards, check if the registry service is available using `docker service ls`.


### Configure the shared Docker network

Additionally, *gpu-jupyter* is connected to the data-source via the same *docker-network*. Therefore, This network must be set to **attachable** in the source's `docker-compose.yml`:

```yml
services:
  data-source-service:
  ...
      networks:
      - default
      - datastack
  ...
networks:
  datastack:
    driver: overlay
    attachable: true  
```
 In this example, 
 * the docker stack was deployed in Docker swarm with the name **elk** (`docker stack deploy ... elk`),
 * the docker network has the name **datastack** within the `docker-compose.yml` file,
 * this network is configured to be attachable in the `docker-compose.yml` file
 * and the docker network has the name **elk_datastack**, see the following output:
    ```bash
    sudo docker network ls
    # ...
    # [UID]        elk_datastack                   overlay             swarm
    # ...
    ```
  The docker network name **elk_datastack** is used in the next step as a parameter.
   
### Start GPU-Jupyter in Docker Swarm

Finally, *gpu-jupyter* can be deployed in the Docker Swarm with the shared network, using:

```bash
./add-to-swarm.sh -p [port] -n [docker-network]
```
where:
* port specifies the port on which the service will be available.
* and docker-network is the name of the attachable network from the previous step, e.g., here it is **elk_datastack**.

Now, *gpu-jupyter* will be accessable on [localhost:port](http://localhost:8888) and shares the network with the other data-source. I.e, all ports of the data-source will be accessable within *gpu-jupyter*, even if they aren't routed it the source's `docker-compose` file.
