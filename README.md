# live-environments

This repository is the single source of truth for what runs in the Kubernetes clusters (GitOps).
Argo CD watches it and makes each cluster match what is written here.

> Rule: do not change anything directly in the cluster or in the Argo CD UI.
> Change it here, in Git, and let Argo CD apply it.

## How it works

```
  user-crud-app repo                              live-environments repo (this one)
  ------------------                              ---------------------------------
  code + charts/user-crud-app
  GitHub Actions (CI):
   1. test
   2. build 2 images -> ECR (OIDC)    ----------> image repository + tags
   3. package chart -> S3 (OIDC)      ----------> chart URL + chart version
   4. commit them here (deploy key)               environments/dev/user-crud-app/
                                                    Chart.yaml   (chart URL + version)
                                                    values.yaml  (image repository + tags)
                                                         |
                                       Argo CD (in the EKS cluster) polls this repo about
                                       every 3 minutes, downloads the chart from S3,
                                       renders it with values.yaml and applies the result
```

1. A commit to `main` in the `user-crud-app` repo (changes in `app/` or `charts/`) starts CI.
2. CI runs the tests, builds the backend and frontend images and pushes them to ECR with the tags `backend-sha-<commit>` and `frontend-sha-<commit>`.
3. CI packages the Helm chart and uploads it to the S3 chart repository.
4. The last CI job writes the chart URL and version into `Chart.yaml`, and the image repository and tags into `values.yaml`, then pushes one commit to this repository.
5. Argo CD sees the new commit, renders the chart and updates the pods.

CI never runs `kubectl` or `helm` against the cluster. Only Argo CD changes the cluster.
