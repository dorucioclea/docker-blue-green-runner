# Docker-Blue-Green-Runner

> Zero-downtime Nginx Blue-Green deployment on a service layer

To deploy web projects must be [simple](https://github.com/Andrew-Kang-G/docker-blue-green-runner).

## Introduction

With your project and its only Dockerfile (docker-compose.yml in the 'samples' folder is ignored), Docker-Blue-Green-Runner handles the rest of the Continuous Deployment (CD) process. Nginx allows your project to be deployed without experiencing any downtime.

![img.png](/documents/images/img.png)

## How to Start with a Node Sample.

A Node.js sample project (https://github.com/hagopj13/node-express-boilerplate) that has been receiving a lot of stars, comes with an MIT License and serves as an example for demonstrating how to use Docker-Blue-Green-Runner.

```shell
# First, as the sample project requires Mongodb, run it separately.
cd samples/node-express-boilerplate
docker-compose up -d mongodb
```

```shell
cd ../../
# [NOTE] Initially, since the sample project does not have the "node_modules" installed, the Health Check stage may take longer.
bash run.sh
```

## Environment Variables
```shell
# If this is set to be true, that means running 'stop-all-containers.sh & remove-all-images.sh'
# Why? In case you get your project renamed or moved to another folder, docker may NOT work properly.  
DOCKER_LAYER_CORRUPTION_RECOVERY=false

# If this is set to true, Nginx will be restarted, resulting in a short downtime. This option should be used when Nginx encounters errors or during the initial deployment.
NGINX_RESTART=false
CONSUL_RESTART=false

# The value must be json or yaml type, which is injected into docker-compose-app-${app_env}.yml
PROJECT_ENVIRONMENTS={"MONGODB_URL":"mongodb://host.docker.internal:27017/node-boilerplate","NODE_ENV":"development"}
```
## Emergency
1) Nginx (like when Nginx is NOT booted OR 502 error...)
```shell
bash emergency-nginx-restart.sh

# If the script above fails, set NGINX_RESTART to be true on .env. and..
bash run.sh

# Ways to check logs
docker logs -f ${project_name}-nginx   # e.g. node-express-boilerplate-nginx
# Ways to check Nginx error logs
docker exec -it ${project_name}-nginx bash # now you're in the container. Check '/var/log/error.log'
```

## Structure
```shell
In run.sh

_main() {

  check_necessary_commands

  cache_global_vars

  check_env_integrity

  apply_env_service_name_onto_app_yaml
  apply_ports_onto_nginx_yaml
  apply_project_environments_onto_app_yaml
  create_nginx_ctmpl

  backup_app_to_previous_images
  backup_nginx_to_previous_images

  if [[ ${app_env} == 'local' ]]; then

      give_host_group_id_full_permissions
  else

      create_host_folders_if_not_exists
  fi

  #docker system prune -f
  if [[ ${docker_layer_corruption_recovery} == true ]]; then
    terminate_whole_system
  fi


  load_app_docker_image


  load_consul_docker_image


  load_nginx_docker_image

  if [[ ${app_env} == 'real' ]]; then
    inject_env_real
    sleep 2
  fi

  load_all_containers

  ./activate.sh ${new_state} ${state} ${new_upstream} ${consul_key_value_store}


  re=$(check_availability_out_of_container | tail -n 1);
  if [[ ${re} != 'true' ]]; then
    echo "[ERROR] Failed to call app_url outside container. Consider running bash rollback.sh. (result value : ${re})" && exit 1
  fi

  ## From this point on, regarded as "success"

  backup_to_new_images

  echo "[NOTICE] The previous (${state}) container exits because the deployment was successful. (If NGINX_RESTART=true or CONSUL_RESTART=true, existing containers have already been terminated in the load_all_containers function.)"
  docker-compose -f docker-compose-app-${app_env}.yml stop ${project_name}-${state}

  echo "[NOTICE] Delete <none>:<none> images."
  docker rmi $(docker images -f "dangling=true" -q) || echo "[NOTICE] If any images are in use, they will not be deleted."
}

```