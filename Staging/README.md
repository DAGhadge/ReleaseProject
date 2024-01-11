Staging

I've set up the EKS cluster to facilitate the deployment of various services. To achieve this, Docker images for all three services are created using Dockerfiles. These images are then utilized within the deployment manifest to deploy pods under the services resource.

The EKS cluster already hosts essential resources, including the Ingress Controller, Services, and Deployments. Additionally, a staging Aurora DB service is running to support the staging services.

Integration with the ALB ingress controller is a key aspect. Cloud ALB resources are attached to the Ingress to effectively route traffic to specific services within the cluster.

Three distinct Ingress controllers—Feed Normalizer, Event Management, and Frontend—are employed, each integrated with an ALB. These ALBs are responsible for routing traffic to the respective services within the cluster.

Deployment manifests encompass pod templates and container images, facilitating a rolling deployment—the default deployment strategy in Kubernetes.

The pipeline utilizes the 'rollout' command to deploy the updated image. In case of any stage failure, the 'rollout undo' command is employed to rollback to the previous image deployment.

Prisma migration is employed to facilitate the migration of the database during the staging process
