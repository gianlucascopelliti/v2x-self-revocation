# Efficient and Timely Revocation of V2X Credentials

This repository contains software artifacts for the paper _Efficient and Timely
Revocation of V2X Credentials_ that will appear at NDSS Symposium 2024.

## Artifact Abstract

In Intelligent Transport Systems, secure communication between vehicles,
infrastructure, and other road users is critical to maintain road safety. This
includes the revocation of cryptographic credentials of misbehaving or malicious
vehicles in a timely manner. However, current standards are vague about how
revocation should be handled, and recent surveys suggest severe limitations in
the scalability and effectiveness of existing revocation schemes. In our paper
"Efficient and Timely Revocation of V2X Credentials" (to appear at NDSS
Symposium 2024), we present a formally verified mechanism for self-revocation of
Vehicle-to-Everything (V2X) pseudonymous credentials, which relies on a trusted
processing element in vehicles but does not require a trusted time source. Our
scheme is compatible with ongoing standardization efforts and, leveraging the
Tamarin prover, is the first to guarantee the actual revocation of credentials
with a predictable upper bound on revocation time and in the presence of
realistic attackers. We further test our revocation mechanism in a virtual
5G-Edge deployment scenario running on Kubernetes, where a large number of
vehicles communicate with each other simulating real-world conditions such as
network malfunctions and delays. Our approach relies on distributing revocation
information via so-called Pending Revocation Lists (PRLs) where, unlike classic
Certificate Revocation Lists (CRLs), a pseudonym only needs to stay for a
limited time, needed to guarantee revocation. The process of adding and removing
pseudonyms from PRLs can be represented as a finite state machine where the
states are the possible sizes of the list, and a Markov model to describe the
probability of moving from state to state. We use such a model to predict the
size of PRLs and the related impact on the V2X system for different scenarios.

## Structure

The repository contains the following sub-artifacts:

- `prl`
   contains scripts to generate and plot the markov matrices 
   (Sec. VII-B / Appendix D)
- `proofs`
   contains the Tamarin models used to formally verify our revocation scheme 
   (Sec. VI / Appendix B)
- `simulation`
   Containing the code used for our evaluation 
   (Sec. VII-A / Appendix C)

A README is also included on each individual folder containing instructions to
run the artifacts.

## Artifact Evaluation

Below you can find detailed instructions to reproduce the results of our paper.

To facilitate the artifact evaluation process, we give instructions for running
the experiments using our pre-built Docker images. However, if desired, the
README files on each folder contains extensive instructions for running all the
experiments locally.

For reference, our results are also computed via our [CI
pipeline](https://github.com/gianlucascopelliti/v2x-self-revocation/actions/workflows/simulation.yml).
The latest workflow contains the same steps described in this section and the
artifacts generated as output.

### Overview

* [Getting started](#getting-started) (~10-30 human-minutes, ~15 compute-minutes)
* [Kick-the-tires stage](#kick-the-tires-stage) (~8-10 human-minutes, ~14-16 compute-minutes)
* [Tamarin models](#tamarin-models) (~2 human-minutes, ~10 compute-minutes)
* [Simulations](#simulations) (~20-30 human-minutes, ~5 compute-hours)
* [PRL evaluation](#prl-evaluation)

### Getting started

Our artifacts can be run on a commodity desktop machine with a x86-64 CPU an a
recent Linux operating system installed (preferably one between Ubuntu 20.04,
Ubuntu 22.04 and Debian 12). A machine with at least 8 cores and 16 GB of RAM is
_recommended_ to ensure that all the artifacts run correctly.

Below, we require the following software components installed and configured on
the machine:
- `git`, `make`, `screen` installed via apt: `sudo apt install -y git make screen`
- [Docker](https://docs.docker.com/engine/install/)
   - The `Docker Compose` plugin is also needed but should be installed
     automatically
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)

Below, we provide instructions to set up all dependencies on a fresh VM (we
tested on Ubuntu 20.04 and 22.04).

```bash
# Install dependencies
./install.sh

# enable docker group in the current shell
# NOTE: to make it permanent, you should logout and login again
newgrp docker
```

In the end, make sure of the following:
1. Docker is installed correctly: the `docker run --rm hello-world` command
   succeeds
2. Docker Compose is installed: `docker compose version` succeeds
3. Kubectl is installed: `kubectl version` succeeds
4. Minikube is installed: `minikube version` succeeds

The time to complete this step depends on whether you already have some (or all)
of the dependencies installed, and whether you use our `install.sh` script or
you install the dependencies manually. Overall, this should not take more than
30 human-minutes and 15 compute-minutes.

### Kick-the-tires stage

In this step, we will test our setup to ensure that our artifacts run correctly.

Note, we assume you are using a _fresh_ environment (e.g., no containers or
Minikube clusters are running) and you already followed the [Getting
started](#getting-started) instructions.

**STEP 1: Tamarin models (~2-3 human-minutes, ~1 compute-minutes)**

```bash
# go to the `proofs` folder
cd proofs

# 2A: prove the lemma `all_heartbeats_processed_within_tolerance` of the `centralized-time` model (~5 seconds)
#     Expected output: 
#     - Tamarin computations and "summary of summaries" at the end
#     - `all_heartbeats_processed_within_tolerance` marked as "verified"
#     - a `output_centralized.spthy` file under `./out`
make test MODEL=centralized-time OUT_FILE=output_centralized.spthy

# 2B: prove the lemma `all_heartbeats_processed_within_tolerance` of the `distributed-time` model (~20 seconds)
#     Expected output: 
#     - Tamarin computations and "summary of summaries" at the end
#     - `all_heartbeats_processed_within_tolerance` marked as "verified"
#     - a `output_distributed.spthy` file under `./out`
make test MODEL=distributed-time OUT_FILE=output_distributed.spthy

# 2C: clean up (~1-5 seconds)
make clean

# go back to the root folder
cd ..
```

**STEP 2: Simulations (~5 human-minutes, ~10 compute-minutes)**

Note: for step 3C, please follow the [Test instructions](./simulation/README.md#test).

```bash
# go to the `simulation` folder
cd simulation

# 3A: create a new Minikube instance (~1-5 minutes)
#     Note: by default we use all CPUs and half the memory in your machine
#     You can override this by setting the MINIKUBE_CPUS and MINIKUBE_MEMORY variables
#     Expected output:
#     - A success message similar to "Done! kubectl is now configured..."
#     - The "./logs" folder created
#     - kubectl works correctly: try running `kubectl get nodes`
make run_minikube

# 3B: build application from source on the Minikube instance (~3 minutes)
#     Expected output:
#     - No error messages
make build_minikube

# 3C: run and interact with the application (~5 minutes)
#     Please follow closely the "Test" section in "simulation/README.md" (see above)

# 3D: Shut down application, delete minikube instance and remove files (~2 minutes)
make clean_all

# go back to the root folder
cd ..
```

**Step 3: PRL evaluation (~1-2 human-minutes, ~3-5 compute-minutes)**

```bash
# go to the `prl` folder
cd prl

# 1A: test setup (~1-5 seconds)
#     Expected output: markov matrix and the "Done" message, two cached files under `./cached`
make test

# 1B: single distribution (~3-5 minutes)
#     Expected output: markov matrix and "Done" message, two cached files under `./cached`
make single

# 1C: clean up (~1-5 seconds)
make clean

# go back to the root folder
cd ..
```

### Tamarin models

| Artifact     | Paper references    | Description                            |
|--------------|---------------------|----------------------------------------|
| `centralized-time` Tamarin model | Sect. VI, Appendix B, Figs. 3,4,9, Table I | Model and proofs that verify the properties defined in Sect. V-A, for the main design of Sect. V |
| `distributed-time` Tamarin model | Appendix B, Fig. 10, Table II | Variant of the model that assumes a trusted time source in TCs, as discussed in Sect. V-B, and proofs |

The steps for reproducing our results are the same as done in kick-the-tires
stage, but this time we will ask Tamarin to verify _all_ lemmas.

In total, this evaluation should take around **10 compute-minutes**.

```bash
# go to the `proofs` folder
cd proofs

# Step 1: prove all lemmas of `centralized-time` model (~5 minutes)
#     Expected output: 
#     - Tamarin computations and "summary of summaries" at the end
#     - In the summary, all lemmas are marked as "verified"
#     - a `output_centralized.spthy` file under `./out`
make prove MODEL=centralized-time OUT_FILE=output_centralized.spthy

# Step 2: prove all lemmas of `distributed-time` model (~5 minutes)
#     Expected output: 
#     - Tamarin computations and "summary of summaries" at the end
#     - In the summary, all lemmas are marked as "verified"
#     - a `output_distributed.spthy` file under `./out`
make prove MODEL=distributed-time OUT_FILE=output_distributed.spthy

# Step 3: clean up (~1-5 seconds)
make clean

# go back to the root folder
cd ..
```

### Simulations

| Artifact        | Paper references        | Description                            |
|-----------------|-------------------------|----------------------------------------|
| `scenario-a1`   | Sect. VII-A, Fig. 5,11  | Box plots showing distributions of revocation times under different attacker classes, for T_v = 30 seconds and no trusted time in TCs |
| `scenario-a2`   | Appendix C, Fig. 12     | Box plots showing distributions of revocation times under different attacker classes, for T_v = 150 seconds and no trusted time in TCs |
| `scenario-b1`   | Appendix C, Fig. 13     | Box plots showing distributions of revocation times under different attacker classes, for T_v = 30 seconds and assuming a trusted time in TCs |
| `scenario-b2`   | Appendix C, Fig. 14     | Box plots showing distributions of revocation times under different attacker classes, for T_v = 150 seconds and assuming a trusted time in TCs |

In order to run the simulations locally and within a few hours, we provide a
scaled-down configuration that spawns 50 vehicles and runs all simulations in
around **4.5 to 5 compute-hours**. This configuration is described in the
`conf/ae.yaml` file, and should provide _similar_ results compared to ours,
although with much less collected data. For a more detailed description of this
setup, [click here](./simulation/README.md#scaled-down-setups).

Note: we recommend running the simulations when nothing else is running on the
same machine.

```bash
# go to the `simulation` folder
cd simulation

# Step 1: create a new Minikube instance and build application (~3-8 minutes)
#     Note: These are the same steps as done in the kick-the-tires stage
#           If you still have a Minikube cluster running, you can skip this step
#     Expected output:
#     - Same as the `kick-the-tires` stage (no errors, `logs/` created)
make run_minikube
make build_minikube

# Step 2: Test - run the first run of the first scenario for five minutes (~5 minutes)
#     This step is only to make sure our setup works
#     Expected output:
#     - a `simulation` folder created
#     - (after ~1 minute since start) the log file `simulations/out.log` does not show errors and is "SLEEPING"
#     - (after ~2-3 minutes since start) `kubectl -n v2x get pods` shows all pods in "Running" state
#     - (after ~5-6 minutes since start) `kubectl -n v2x get pods` shows pods in "Terminating" state or no pods at all
#     - (after ~6-7 minutes since start) the log file `simulations/out.log` shows "ALL DONE" as last message
#     - (after ~6-7 minutes since start) the `simulations/results` folder contains `scenario-a1_1-honest.json`
make run_simulations_background CONF=conf/ae.yaml SCENARIO=scenario-a1 RUN=1-honest SIM_TIME=300 DOWN_TIME=30

# Step 3: Run all simulations (~4.5-5 hours)
#     Note: you can run the command, grab a coffee and come back in 5 hours
#     Expected output:
#     - a `simulations` folder created. `simulations/scenarios` contain one `.properties` file for each run (total: 16 files)
#     - (after ~1 minute since start) the log file `simulations/out.log` does not show errors and is "SLEEPING"
#     - (after ~2 minutes since start) `kubectl -n v2x get pods` shows all pods in "Running" state
#     - (after ~4.5 hours since start) `kubectl -n v2x get pods` shows pods in "Terminating" state or no pods at all
#     - (after ~4.5 hours since start) the log file `simulations/out.log` shows "ALL DONE" as last message
#     - (after ~4.5 hours since start) the `simulations/results` folder contains one JSON file for each run (total: 16 files)
make run_simulations_background CONF=conf/ae.yaml

# Step 4: Plot results (~2 minutes)
#     Expected output:
#     - `data`, `figs` and `tikz` folders created under `simulations`
#     - `data` contains 8 files (2 for each scenario), `figs` and `tikz` one file for each scenario
#     - 
make plot_all

# Step 5 (optional): Copy results somewhere safe (`simulations` folder will be deleted in the next step!)

# Step 6: Shut down application, delete minikube instance and remove files (~2 minutes)
make clean_all

# go back to the root folder
cd ..
```

### PRL evaluation

| Artifact              | Paper references    | Description                            |
|-----------------------|---------------------|----------------------------------------|
| `p-plot`              | Sect. VII-B, Fig. 6 | Plots percentiles for maximum PRL sizes under different scenarios and shares of attackers, with fixed T_prl and number of pseudonyms |
| `tv-distribution`     | Sect. VII-B, Fig. 7 | Evaluates T_eff, heartbeat frequency, heartbeat size, and required bandwidth under different values for T_v |
| `tikz-graph`          | Appendix D, Fig. 15 | Simple transition graph |
| `n-plot`              | Appendix D, Fig. 16 | Plots 99th percentile for maximum PRL sizes under different number of pseudonyms, in four different scenarios |
| `t-plot`              | Appendix D, Fig. 17 | Plots 99th percentile for maximum PRL sizes under different values for T_prl, in four different scenarios |

In total, this evaluation should take around **2.5 compute-hours**.

```bash
# go to the `prl` folder
cd prl

# Step 1: transition graph (~1-5 seconds)
#     Expected output: markov matrix and the "Done" message, `tikz-graph.tex` file under `plots`
make tikz

# Step 2: Plot series over the different probabilities (~16 minutes)
#     Expected output:
make p-plot

# Step 3: Plot series over the number of pseudonyms (~26 minutes)
#     Expected output:
make n-plot

# Step 4: Plot series over the time each pseudonym stays in the list (~90 minutes)
#     Expected output:
make t-plot

# Step 5: Generate distribution for Tv (~18 minutes)
#     Expected output:
make tv-distribution

# go back to the root folder
cd ..
```