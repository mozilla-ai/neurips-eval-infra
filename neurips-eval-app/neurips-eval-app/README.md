# Neurips Model and HELM Eval Server

In order to evaluate NeurIPS 2023 submissions, we need to be able to pass custom Docker images from private repos. [See comments here.](

Steps: 

1. Make sure the Dockerfile for your container exposes port `8080`. You can run it from within the pod, `CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]` should the be last line; alternatively to start manually: `uvicorn main:app --host 0.0.0.0 --port 8080` 

2. Build your model container on a CoreWeave Virtual Server and push to the Docker Registry. To spin up a virtual server, [follow instructions here.](

Make sure to include the Github eval repo prefix, i.e. `llm-efficiency-challenge/neurips_llm_efficiency_challenge` when image is built so we can keep track of submissions.  If you're building locally on MacOS with `arm`, make sure to specify the platform so we can run it on Coreweave

`docker build --platform linux/amd64 -t docker-registrylit-gpt:1 .` 

otherwise, build with: `docker build -t docker-registrylit-gpt:1 .`

2a. Train the model using the steps provided in the repo to repro. save the model artifact, preferrably to HuggingFace.  Use this server for inference as well. 

3. In order to evaluate NeurIPS 2023 submissions, we need to also be able to deploy the Helm server to accompany the custom Docker submission. Build your eval container locally and push. Make sure to include the repo of the GitHub repo we're evaluating as the prefix when image is built so we can keep track of submissions. [The Docker image is built here.](https://github.com/mozilla-ai/helm-server) 

4. [Push to the registry]() and 
check to make sure it landed in the [registry UI:](https://docker-registry) 
`docker push docker-registrylit-gpt:1`
 
5. Install via Helm: `helm install {deployment name} . `

You can override parameters from the chart's values.yaml on the command line, here is an example:

```
helm install neurips-4090-eval . --set gpus=4 --set cpus=32 --set memory=64Gi --set image=docker-registry
``````

6. Test that the model container works by hitting the endpoints from the API: `curl -X POST -H "Content-Type: application/json" -d '{"prompt": "The capital of france is "}' http://{ip}:8080/process`

7. The two containers should see each other through the internal IP network on the Pod via port 8080. 

8. To run the eval server, run the sample script from the eval server. 
`helm-run --conf-paths run_specs_full_coarse_600_budget.conf --suite v1 --max-eval-instances 10`

9. Storage for HELM: 

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  storageClassName: shared-hdd-lga1
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```



