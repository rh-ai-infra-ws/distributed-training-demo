# Distributed Training Demo

Hands-on notebooks for running distributed PyTorch training on **Red Hat OpenShift AI** using **Ray** (KubeRay), **CodeFlare SDK**, and **Kueue** for GPU quota and queueing. The flow provisions a Ray cluster in your project namespace, submits a training job through Ray’s Job Submission API, then inspects status and logs.

This repository is the companion artifact for the **Distributed Training** module in the OpenShift AI labs. For platform setup (DataScienceCluster components, Kueue, namespaces, and workbench creation), follow the lab guide in [rh-ai-infra-ws-labs](https://github.com/rh-ai-infra-ws/rh-ai-infra-ws-labs) under `docs/4-distributed-training/`.

## Prerequisites

- Red Hat OpenShift 4.21+ with OpenShift AI 3.3+
- GPU accelerators configured (NVIDIA GPU Operator, NFD)
- Distributed training components enabled on the DataScienceCluster: `ray`, `trainingoperator`, and `workbenches` (see [Enable Distributed Training Components](https://github.com/rh-ai-infra-ws/rh-ai-infra-ws-labs/blob/main/docs/4-distributed-training/1-enable-distributed-training.md))
- **Red Hat build of Kueue** installed (Kueue component on the DSC should remain **Unmanaged**)
- A Data Science project namespace (for example `<USER_NAME>-ray-train`) with a **LocalQueue** named `ray-train-lq`
- An OpenShift AI workbench using a notebook image that includes the **CodeFlare SDK** (for example *Jupyter | PyTorch | CUDA | Python 3.12*)

## Repository contents

| File | Purpose |
| --- | --- |
| `00-setup-ray.ipynb` | Authenticate to the cluster, define Ray cluster resources, and bring up the Ray cluster |
| `01-execute-distributed-trainig.ipynb` | Connect to the running cluster and submit a distributed training job |
| `02-review-distributed-trainig.ipynb` | List jobs, check status, and stream job logs |

> Notebook filenames use `trainig` (historical typo). Run them in numeric order.

## Quick start

### 1. Prepare the OpenShift project

Create the project and LocalQueue (replace `<USER_NAME>` with your lab username):

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Namespace
metadata:
  name: <USER_NAME>-ray-train
  labels:
    name: <USER_NAME>-ray-train
    opendatahub.io/dashboard: 'true'
EOF
```

```bash
cat << 'EOF' | oc apply -n <USER_NAME>-ray-train -f -
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  namespace: <USER_NAME>-ray-train
  name: ray-train-lq
spec:
  clusterQueue: default
EOF
```

### 2. Clone and open in a workbench

Clone this repository into your workbench workspace (or upload the notebooks), then open and run the notebooks in order.

### 3. Configure placeholders

Before running cells, replace the `TODO` placeholders in each notebook:

| Placeholder | Where used | Description |
| --- | --- | --- |
| OpenShift API token | All notebooks | User token from the OpenShift console (*Copy login command* → display token) |
| API server URL | All notebooks | Cluster API URL (for example `https://api.<cluster>:6443`) |
| Namespace | `01`, `02` | Your Ray project namespace (for example `<USER_NAME>-ray-train`) |

`TokenAuthentication` with `skip_tls=True` is used for lab environments with self-signed certificates. For production clusters, prefer proper TLS verification and kubeconfig-based auth when available.

### 4. Run the notebooks

**`00-setup-ray.ipynb`** — Provisions a Ray cluster named `ray-cluster`:

- Lists local Kueue queues via CodeFlare SDK
- Defines head/worker CPU, memory, and GPU requests (`nvidia.com/gpu`)
- Submits the cluster to `ray-train-lq` and waits until ready

Verify pods in your namespace:

```bash
oc get po -n <USER_NAME>-ray-train
```

Expect head and worker pods in `Running` with `READY` true, for example:

```text
ray-cluster-head-xxxxx                             2/2     Running   ...
ray-cluster-small-group-ray-cluster-worker-xxxxx   1/1     Running   ...
```

**`01-execute-distributed-trainig.ipynb`** — Submits a Ray job:

- Reconnects to the existing `ray-cluster`
- Uses the Ray Job Submission Client with entrypoint `python mnist_fashion.py`
- Packages the working directory and `requirements.txt` as the job runtime environment

**`02-review-distributed-trainig.ipynb`** — Monitors the submitted job:

- Lists jobs and selects a submission ID
- Prints job status
- Tails job logs asynchronously

You can also monitor jobs in the **Ray dashboard** linked from the OpenShift AI workbench or cluster UI.

## Default Ray cluster shape

The setup notebook uses illustrative resource values (adjust for your quota and hardware profile):

| Setting | Value |
| --- | --- |
| Cluster name | `ray-cluster` |
| Workers | 4 |
| Local queue | `ray-train-lq` |
| Head GPUs | 2 (`nvidia.com/gpu`) |
| Worker GPUs | 2 per worker |

## Related lab topics

The same module also covers **Kubeflow Trainer v2** `TrainJob` workloads and `ClusterTrainingRuntime` templates (`torch-distributed`, and others). That path is documented in the lab repo and does not use these Ray notebooks.

## Troubleshooting

- **Cluster not ready** — Check Kueue quota and LocalQueue binding; ensure the workbench GPU profile matches requested resources.
- **Authentication errors** — Refresh the OpenShift token; confirm API server URL and namespace.
- **Job pending or failed** — Inspect Ray head logs: `oc logs -n <namespace> -l ray.io/node-type=head`
- **Missing training script** — `01-execute-distributed-trainig.ipynb` expects `mnist_fashion.py` and `requirements.txt` in the job `working_dir` (typically the notebook directory).

## License

Use and distribution terms follow the parent lab / organization policy for `rh-ai-infra-ws` workshop materials.
