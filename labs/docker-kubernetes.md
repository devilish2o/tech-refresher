# Lab: Docker and Kubernetes

## Part A — Docker

Build a tiny sample service in a multi-stage Dockerfile. Use a small trusted base image, a non-root runtime user, a .dockerignore file, explicit ports and a health check where supported. Inspect image layers and scan the image. Run with resource limits and a read-only filesystem if the application permits it.

Documentation: [Docker Get Started](https://docs.docker.com/get-started/) · [Build best practices](https://docs.docker.com/build/building/best-practices/)

## Part B — Kubernetes

Use kind or minikube locally. Create a namespace, Deployment and Service. Add readiness and liveness probes, resource requests/limits and labels. Perform a rolling update, inspect rollout status and roll back to the previous revision.

Troubleshoot one intentional failure: bad image name, failing probe, incorrect selector or insufficient resources. Use `kubectl describe`, `kubectl logs`, `kubectl get events` and `kubectl rollout history`.

Documentation: [Kubernetes concepts](https://kubernetes.io/docs/concepts/) · [kubectl reference](https://kubernetes.io/docs/reference/kubectl/)
