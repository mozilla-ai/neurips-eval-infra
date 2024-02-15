# Neurips Model Server and HELM Eval Server

> [!INFO]
>
> This is a sample reference implementation of the architecture we used to finetune-evaluate NeurIPS Large Language Model Efficiency Challenge submissions. 
> It doesn't include Dockerfile image paths and is missing some K8s deployment-level values. 
> If you'd like to deploy in your own environment, you'll need to add those values. 

In order to evaluate [NeurIPS 2023 submissions](https://llm-efficiency-challenge.github.io/challenge), we're running a Kubernetes deployment with two pods: 
+ a model pod (`Dockerfile.model`) and 
+ a HELM server pod (`Dockerfile.helm`). 

For evaluation, we run HELM from the server pod to hit the model endpoint on the model pod. 

We build both from a Dockerfile locally and push to the Coreweave Docker Registry. Our Kubernetes cluster then starts a deployment with two containers.  For this, we're creating a custom Helm chart.

## Building: 

### Helm Image

1. Build a custom Docker image, either locally if you have enough memory, or on a Docker server. We use a Coreweave Virtual Server for this. Build the Docker image for [Helm-Server based on instructions at HELM](https://github.com/llm-efficiency-challenge/neurips_llm_efficiency_challenge/blob/master/helm.md)
with this [config file](https://raw.githubusercontent.com/llm-efficiency-challenge/neurips_llm_efficiency_challenge/master/run_specs_full_coarse_600_budget.conf). 

If you're building locally on MacOS with `arm`, make sure to specify the platform so we can run it on Coreweave

  `docker build --platform linux/amd64 -t {docker-registry}/helm-server:1 .`


2. Push to the registry 

`docker push{docker-registry}/helm-server:1`

### Model Image

1. Build with the instructions specified by the contestant and also push to the registry. 
2. Make sure the Dockerfile for your model container exposes port `8080` to serve the model.  You can run it from within the pod, `CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]` as the last line; alternatively to start manually for troubleshooting: `uvicorn main:app --host 0.0.0.0 --port 8080` 

## Run fine-tuning

1. Train the model using the steps provided in the contestant's repo to repro.
2. The model artifact should be generally in HuggingFace-style, with a repo full of artifacts and several checkpoints. 
Save all checkpoints and artifacts to shared storage. 

## Running in K8s

 
1. Once the two Docker images are in the registry, install via Helm chart: `helm install {deployment name} . `

2. Test that the model container works by hitting the endpoints locally: `curl -X POST -H "Content-Type: application/json" -d '{"prompt": "The capital of france is "}' http://{model_container_ip}:8080/process`

3. The two containers should see each other through the internal IP network on the Pod via port 8080. 

4. To run the eval server, run the sample script from the eval server. Specify `nohup` or use `screen` or `tmux` if you'd like to run without interruptions. 

`helm-run --conf-paths run_specs_full_coarse_600_budget.conf --suite v1 --max-eval-instances 1`






