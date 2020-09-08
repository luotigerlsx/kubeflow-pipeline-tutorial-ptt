## Regional Container Registry
[Artifact Registry](https://cloud.google.com/artifact-registry)
is a single place for your organization to manage container images and language packages (such as Maven and npm). It is fully integrated with Google Cloud’s tooling and runtimes and comes with support for native artifact protocols. More importantly, it supports regional and multi-regional repositories.

### The steps to create regional Docker repository is as follows

- Run the following command to create a new Docker repository named `AF_REGISTRY_NAME` in the location `AF_REGISTRY_LOCATION` with the description "Regional Docker repository".

```shell
gcloud beta artifacts repositories create $AF_REGISTRY_NAME \
    --repository-format=docker \
    --location=$AF_REGISTRY_LOCATION \
    --project=$PROJECT_ID \
    --description="Regional Docker repository"
```

- Run the following command to verify that your repository was created.

```shell
gcloud beta artifacts repositories list --project=$PROJECT_ID
```

The supported regions can be found [here](https://cloud.google.com/artifact-registry/docs/repo-organize#locations). The repository URI after creation will be
```
{AF_REGISTRY_LOCATION}-docker.pkg.dev/{PROJECT_ID}/{AF_REGISTRY_NAME}/
```
and the image full uri typically will be
```
{AF_REGISTRY_LOCATION}-docker.pkg.dev/{PROJECT_ID}/{AF_REGISTRY_NAME}/{IMAGE_NAME}:{TAG}
```

## Regional Endpoint of AI Platform Prediction
Interacting with AI Platform services, e.g. training and prediction, will require the access of the endpoint. There are two options available, i.e., **global endpoint** and **regional endpoint**:
- When you create a model resource on the global endpoint, you can specify a region for your model. When you create versions within this model and serve predictions, the prediction nodes run in the specified region. 
- When you use a regional endpoint, AI Platform Prediction runs your prediction nodes in the endpoint's region. However, in this case AI Platform Prediction provides additional isolation by running all AI Platform Prediction infrastructure in that region.

For example, if you use the us-east1 region on the global endpoint, your prediction nodes run in us-east1. But the AI Platform Prediction infrastructure managing your resources (routing requests; handling model and version creation, updates, and deletion; etc.) does not necessarily run in us-east1. On the other hand, if you use the europe-west4 regional endpoint, your prediction nodes and all AI Platform Prediction infrastructure run in europe-west4.

Current available regional endpoints are: `us-central1`, `europe-west4` and `asia-east1`. However, **regional endpoints do not currently support AI Platform Training**.

### Using regional endpoints
```python
from google.api_core.client_options import ClientOptions
from googleapiclient import discovery

endpoint = 'https://REGION-ml.googleapis.com'
client_options = ClientOptions(api_endpoint=endpoint)
ml = discovery.build('ml', 'v1', client_options=client_options)

request_body = { 'name': 'MODEL_NAME' }
request = ml.projects().models().create(parent='projects/PROJECT_ID',
    body=request_body)

response = request.execute()
print(response)
```

### Using regional endpoints of Kubeflow Pipeline GCP components
The pre-built reusable [GCP Kubeflow Pipeline components](https://github.com/kubeflow/pipelines/tree/master/components/gcp/ml_engine) don't provide regional endpoints capabilities. We have provided a customized version [here](https://github.com/luotigerlsx/pipelines/tree/master/components/gcp). To build and use the customized components, please follow the steps:
- Clone the source code
```shell
git clone git@github.com:luotigerlsx/pipelines.git
```
- Navigate to `pipelines/components` and find the `build_image.sh`. Replace the container registry address accordingly
    - To use global Container Registry, replace `asia.gcr.io/${PROJECT_ID}/` to `gcr.io/${PROJECT_ID}/`
    - To use regional Container Registry, replace `asia.gcr.io/${PROJECT_ID}/` to `Region.gcr.io/${PROJECT_ID}/`. Region can be: `us`, `eu` or `asia`
    - To use regional Artifact Registry, replace `asia.gcr.io/${PROJECT_ID}/` to `${AF_REGISTRY_LOCATION}-docker.pkg.dev/${PROJECT_ID}/${AF_REGISTRY_NAME}/`
- Navigate to `pipelines/components/gcp/container`. Run
```shell
bash build_image.sh -p ${PROJECT_ID}
```
- Navigate to `pipelines/components/gcp/ml_engine/deploy` and modify `component.yaml`
```yaml
implementation:
  container:
    image: {newly_built_image}
    args: [
```
- Save the modified `component.yaml` to a location that can be accessed by the ML taks through
```python
mlengine_deploy_op = comp.load_component_from_url(
    '{internal_accessable_address}/component.yaml')

```