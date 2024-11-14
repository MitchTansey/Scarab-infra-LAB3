# Scarab-infra
Dockerfiles of docker images running data center workloads and run Scarab simulation

For UCSC LitzLab users, please refer to README.LitzLab.md to avoid downloading the traces and to use shared traces in UCSC NFS.

## Docker setup
Install Docker based on the instructions from official [docker docs](https://docs.docker.com/engine/install/).

## Requirements
To run scarab in a docker container, the host machine should use non-root user and the user has a proper GitHub setup to access https://github.com/Litz-Lab/scarab.
To run docker as a non-root user, run: "sudo chmod 666 /var/run/docker.sock" (ref: https://stackoverflow.com/questions/48957195/how-to-fix-docker-got-permission-denied-issue)
To generate a new SSH key and it to the machine's SSH agent: https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent?platform=linux
Add a new SSH key to your GitHub account: https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account?platform=linux

## Running on a Slurm cluster
This script must be run on a slurm cluster connected to the NFS. It will spawn all simpoints as slurm jobs to run on any node where the docker container is found. If no nodes are found it will attempt launch a container on one of the nodes using this Scarab-Infra repository (which should be on the nfs to launch containers from).

### Requirements
Must run on a slurm cluster node
All nodes must have docker installed

### Location of stats, parameters file, and scarab binary
The stats are moved to a different location for the slurm runner. The stats for an experiment can be found in your docker home foler, under the following path:
`/docker_home/experiment_name/configuration_name/workload/simpoint_id/`

The file containing the architectural parameters and the scarab binary used by an experiment can be found at the following path:
`/docker_home/experiment_name/`

### New descriptor file options
These options should not affect functionality of the existing infrastructure. New options are:

Scarab path:
Ex: `"scarab_path": "/path/to/scarab_bin"`
This is a path to the scrab binary to be used. Absolute path reccomended

Docker home:
Ex: `"docker_home": "path/to/docker/container/home"`
This is the path to the 'home' directory where all docker containers are linked. Should have been provided to build command via '-o' option

Docker prefix:
Ex: `"docker_prefix": "app_groupname"`
This corresponds to the APP_GROUPNAME used in the existing infrastructure, which comes from `apps.list`. This is used when creating docker containers using `./run.sh -o .. -b 2`, which produces containers with `{APP_GROUPNAME}_{USERNAME}`. You can check these container names with `docker ps -a`. This APP_GROUPNAME is provided via docker_prefix so the scarab launch script can detect your containers and run containers in them. And example of an app groupname would be `docker_traces`

### Quickstart

#### 1. Build a container
Follow the normal instructions for building a container. Make note of A) the path provided via -o, and B) the name placed in apps.list (Ex: docker_traces).
Note that the path of the home directory should be on NFS

#### 2. Configure desciptor file
Open the slurm_exp.json file and place the docker home directory (the path provided via -o in the previous step) as the "docker_home" option. Then put the name frome apps.list into the json file under "docker_prefix".

#### 3. Run the experiment
Now run `python3 run_slurm.py -dbg 3 -d slurm_exp.json`. Check the running slurm jobs with `squeue`. The statistics should be placed in your docker home folder, under the experiment name. In this folder you will see subfolders for the specified configurations, which contain folders for the workloads, which contain folders for individual simpoint's stats.

### Cancel an experiment
If you want to kill an experiment you previously launched, you can rerun the command used to launch it with `--kill`. Using the launch command of `python3 run_slurm.py -dbg 3 -d slurm_exp.json` as an example, you can cancel it with `python3 run_slurm.py -dbg 3 -d slurm_exp.json --kill`.

### Slurm runner Usage
usage: run_slurm.py [-h] -d DESCRIPTOR_NAME 
                    [-k | --kill | --no-kill]
                    [-dir HOME_DIR] 
                    [-m SCARAB_MODE] 
                    [-s SCARAB_BIN] 
                    [-a ARCH_PARAMS] 
                    [-dbg DEBUG] 
                    [-si SCARAB_INFRA] 
                    [-pref DOCKER_PREFIX]

Runs scrab on a slurm network

options:
  -h, --help            show this help message and exit
  -d DESCRIPTOR_NAME, --descriptor_name DESCRIPTOR_NAME
                        Experiment descriptor name. Usage: -d exp2.json
  -k, --kill, --no-kill
                        Don't launch jobs from descriptor, kill running jobs as described in descriptor
  -dir HOME_DIR, --home_dir HOME_DIR
                        Home directory for the docker containers
  -m SCARAB_MODE, --scarab_mode SCARAB_MODE
                        Scarab mode. Usage -m 2
  -s SCARAB_BIN, --scarab_bin SCARAB_BIN
                        Scarab binary. Path to custom binary to be used
  -a ARCH_PARAMS, --arch_params ARCH_PARAMS
                        Path to a custom <architecture>.PARAMS file for scarab
  -dbg DEBUG, --debug DEBUG
                        1 for errors, 2 for warnings, 3 for info
  -si SCARAB_INFRA, --scarab_infra SCARAB_INFRA
                        Path to scarab infra repo to launch new containers
  -pref DOCKER_PREFIX, --docker_prefix DOCKER_PREFIX
                        Prefix of docker container. Should be found in apps.list. Can be confirmed using docker ps -a and using prefix from {prefix}_{username} under NAMES

The -d (--descriptor_name) is the only required argument if the following options are defined in the descriptor file:
scarab_path - Path to scarab binary. --scarab_bin value will be used instead of the descriptor values if the argument is provided. If neither are provided, the binary in the docker home will be used
docker_home - Path to the home directory used by all docker containers (across all slurm nodes)
docker_prefix - Found in the name of the docker container. It is the part before the username, and is usually set based on the value in apps.list. An example would be docker_traces. To determine this value, run `docker ps -a` and from names of format {APP_GROUPNAME}_{USERNAME} the prefix will be the {APP_GROUPNAME} portion.

## Build a Docker image and run a container of a built image

### By using a script
#### Usage
```
Usage: ./run.sh [ -h | --help ]
                [ -o | --outdir ]
                [ -b | --build]
                [ -t | --trace]
                [ -s | --scarab ]
                [ -e | --experiment ]
                [ -c | --cleanup]

!! Modify 'apps.list' and '<experiment_name>.json' to specify the apps to build and Scarab parameters before run !!
The entire process of simulating a data center workload is the following.
1) application setup by building a docker image (each directory represents an application group)
2) collect traces with different simpoint workflows for trace-based simulation
3) run Scarab simulation in different modes
To perform the later step, the previous steps must be performed first, meaning all the necessary options should be set at the same time. However, you can only run earlier step(s) by unsetting the later steps for debugging purposes.
Options:
h     Print this Help.
o     Absolute path to the directory for scarab repo, pin, traces, simpoints, and simulation results. scarab and pin will be installed if they don't exist in the given path. The directory will be mounted as home directory of a container e.g) -o /soe/user/testbench_container_home
b     Build a docker image with application setup. 0: Run a container of existing docker image 1: Build cached image and run a container of the cached image, 2: Build a new image from the beginning and overwrite whatever image with the same name. e.g) -b 2
t     Collect traces with different SimPoint workflows. 0: Do not collect traces, 1: Only collect traces without simpoint clustering, 2: Collect traces based on SimPoint workflow - post-processing (trace, collect fingerprints, do simpoint clustering). e.g) -t 2
s     Scarab simulation mode. 0: No simulation 1: execution-driven simulation w/o SimPoint 2: trace-based simulation w/o SimPoint (-t should be 1 if no traces exist already in the container). 3: execution-driven simulation w/ SimPoint 4: trace-based simulation w/ SimPoint e.g) -s 4
e     Experiment name. e.g.) -e exp2
c     Clean up all the containers/volumes after run. 0: No clean up 2: Clean up e.g) -c 1
```
There are four ways to run Scarab: 1) execution-driven w/o SimPoint (-s 1) 2) trace-based w/o SimPoint (-s 2) 3) execution-driven w/ SimPoint (-s 3) 4) trace-based w/ SimPoint (-s 4). The execution-driven simulation runs the application binary directly without using traces while the trace-based simulation needs collected traces to run the application. SimPoints are used for fast-forwarding on the execution-driven run and for collecting traces/simulating on the trace-based run.
You need to provide the list of the applications you want to build images for them in 'apps.list' file, and the list of the Scarab parameters to generate parameter descriptor file in '<experiment_name>.json'. Please refer to the 'apps.list' and 'exp2.json' files for the examples.

The following steps are for the fourth running scenario (trace-based w/ SimPoint) with the traces of datacenter workloads we already collected.
#### 1. Build a Docker image and run a container of a built image where all the traces/simpoints are available and ready to run Scarab
```
./run.sh -o <path_to_mount_docker_home> -b 2
```
For example,
```
./run.sh -o /home/$USER/docker_home -b 2
```

#### 2. Run Scarab simulations by using a descriptor file
The example file descriptor for all the simulation scenarios is in docker_traces/desc.json and docker_traces/desc.pt.json
The simulations using memtrace or pt traces should be launched separately with different simulation mode (4 : memtrace, 5 : PT trace). Still the two executions can be parallelized (by launching on different terminals).
Run the following command with <experiment> name for -e.
```
./run.sh -o /home/$USER/docker_home -s 4 -e desc
./run.sh -o /home/$USER/docker_home -s 5 -e desc.pt
```
The script will launch Scarab simulations in background until it runs all the different scenarios x workloads. You can check if all the simulations are over by checking if there is any 'scarab' process running (UNIX 'top' command).

#### 3. Update and rebuild Scarab
A user can update scarab and rebuild for further simulation. Scarab can be updated 'outside' of the container because scarab visible inside the docker container is read only. To exploit already-set simulation/building environment, scarab build itself should be done 'inside' the container.
##### To update Scarab, cd to scarab directory in the absolute path you used for building the docker container.
```
cd /home/$USER/docker_traces/scarab
(update scarab)
```
##### To build scarab again with the updated scarab, go inside the container and build.
```
docker exec --user=$USER --privileged docker_traces_$USER /bin/bash -c "cd /home/$USER/scarab/src && make"
```

#### 4. Clean up any cached docker container/image/builds
```
docker stop docker_traces_$USER
docker rm docker_traces_$USER
docker rmi docker_traces:latest
docker system prune
```

### Step-by-step on an interactive attachment
#### Build an image
```
DOCKER_BUILDKIT=1 COMPOSE_DOCKER_CLI_BUILD=1 docker build . -f ./example/Dockerfile --no-cache -t example:latest
```
or
```
./run.sh -b 2 -o /home/$USER/example_home
```
or
```
export APPNAME="example"
export BUILD=2
source ./setup_apps.sh
./build_apps.sh
```

#### Check the built image
```
docker images
REPOSITORY                   TAG       IMAGE ID       CREATED        SIZE
example                      latest    1dd7a6097ef0   3 hours ago    6.66GB
```

#### Run a container of the image
'docker run' will stop the container after it runs the given command. Run with -v to create a volume and keep the updates inside the container remain. 'docker start' after the run will start the container again. You can run other commands inside the container by running 'docker exec'
```
export LOCAL_UID=$(id -u $USER)
export LOCAL_GID=$(id -g $USER)
export USER_ID=${LOCAL_UID:-9001}
export GROUP_ID=${LOCAL_GID:-9001}
docker run -e user_id=$USER_ID -e group_id=$GROUP_ID -e username=$USER -dit --privileged --name example_$USER --mount type=bind,source=/home/$USER/example_home example:latest /bin/bash
docker start example_$USER
docker exec --privileged example_$USER /bin/bash -c "/usr/local/bin/common_entrypoint.sh"
docker exec --user=$USER --workdir=/home/$USER --privileged example_$USER /bin/bash -c "cd /home/$USER/scarab/src && make"
```

#### Run simpoint workflow and collect traces on an existing container
```
./run.sh -o /home/$USER/example_home -b 0 -s 1 -t 1 -s 0
```

#### Run simulation with already collected simpoint traces on an existing container
```
./run.sh -o /home/$USER/example_home -b 0 -s 0 -t 0 -s 4
```

## Developers
When you add an application support of a docker image, please expand 'setup_apps.sh' script and 'run.sh' if needed so that the memtraces and Scarab results can be provided by running a single script. The rule of thumb is 1) to try to build a simple image where the basic essential packages are installed on a proper Ubuntu version (the first version of Dockerfile), 2) to run a container of the image, 3) to run the application, 4) to run the application with DynamoRIO (if 3) works), 5) to run Scarab with memtrace frontend feeding the collected traces from 4). 
If all 1) to 5) steps are working, you can add the processes you added after 1) to the Dockerfile and expand the script. Make sure that running the script provides the same environment and results as 1~5 steps.

An example commit for adding xgboost : https://github.com/5surim/dcworkloads-dockerfiles/commit/3a22b2ebc4620027a1c475a1ef33b67aa85376d8

An example commit for adding mongo-perf : https://github.com/5surim/dcworkloads-dockerfiles/commit/80f7f9003f8f198eadcb16bc890a2be3ba8619a5

## Notes
* DaCapo (cassandra, kafka, tomcat) - DynamoRIO, Scarab, and applications are successfully running, but DynamoRIO doesn't support jvm applications. memtraces cannot be collected. Only execution-driven simulation available.
* Renaissance (chirper, http)
* HHVM OSS (drupal7, mediawiki, wordpress) - Scarab complilation failed.
* SPEC2017 - only 502.gcc_r has been added. The SimPoint flow fixes the input size to `train` and the segment size to `100000000`.
* Verilator - setup based on https://github.com/efeslab/ispy-ripple. Clustering aborted due to out of memory.

# Publications

```
@inproceedings{oh2024udp,
  author = {Oh, Surim and Xu, Mingsheng and Khan, Tanvir Ahmed and Kasikci, Baris and Litz, Heiner},
  title = {UDP: Utility-Driven Fetch Directed Instruction Prefetching},
  booktitle = {Proceedings of the 51st International Symposium on Computer Architecture (ISCA)},
  series = {ISCA 2024},
  year = {2024},
  month = jun,
}
```
