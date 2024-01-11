# 888Spectate Release Project

## Architecture
![image](https://github.com/DAGhadge/888SpectateReleaseProject/assets/37247362/802ed060-3880-4807-89fb-310d4a9699d9)

![image](https://github.com/DAGhadge/888SpectateReleaseProject/assets/37247362/268dcddc-3319-4703-9cd5-886c7ba864e3)



Architecture flow
1. The internet-facing Application Load Balancer (ALB) directs user traffic to the Frontend Service, which is integrated with the reader endpoint of the Aurora Database. Two target groups are set up for the old and new versions of the Frontend Service. Initially, 100% of the traffic is routed to the old version (Blue Deployment) using load balancer weight routing. The deployment of the new Frontend version (Green Deployment) is orchestrated through the ansible playbook frontendDeploy.yaml.
   
2. During the new schema deployment, the Aurora DB undergoes a Prisma migration. Rather than having a standby Aurora Cluster and replicating data from the primary DB, an alternative approach is Aurora Blue/Green Deployment, which helps minimize downtime. While replication is in progress, changes can be made to the new DB schema. However, it's crucial to ensure that schema changes are backward compatible and idempotent to avoid errors during replication, especially when altering or deleting columns.
In my implementation, I've opted for the traditional method of deploying the DB version, utilizing Prisma migrations for a seamless transition between schemas. This involves having a migrations folder in the Git repository, fetching details from the repo, and specifying Aurora DB connection details in the schema.prisma file. The migration command commonly used is "npx prisma migrate deploy."
The reader endpoints are exposed to the Frontend service since it only performs read operations on the DB. On the other hand, the Event Management Service requires both read and write access, so both reader and writer endpoints are exposed accordingly.

3.The Event Management Service is a containerized application intended for deployment on the ECS service within the cloud environment. To facilitate this deployment, we require an identical replica of the stable deployment, encompassing a target group with ECS tasks organized under a service within the ECS cluster. These tasks are placed behind a load balancer, which initially routes 100% of the traffic to the stable target group of tasks.
In the Jenkins pipeline stage, we undertake the provisioning and deployment of the provided image onto a new task definition. Simultaneously, a new service is created, housing the newly defined tasks with a specified number of desired instances. A fresh target group is generated to accommodate these containers, and this target group is linked to the existing Application Load Balancer (ALB) with a 0% traffic weight initially. Meanwhile, the stable version of containers retains a 100% weight allocation.
It is essential to note that we don't assume that the infrastructure for the ECS new version has been provisioned by the infrastructure team. Instead, the ECS CLI is utilized within the Jenkins pipeline to dynamically create the Task Definition, set up the new service, create a new target group, and establish connections with the existing ALB.

4.The deployment of Feed Normalizers follows a similar blue/green approach. Feed Normalizers are organized into distinct target groups based on their configured sports. Traffic routing from the Application Load Balancer (ALB) is determined entirely by the weightage assigned to the stable target groups.
Deployment of Feed Normalizers on the instances is orchestrated through Ansible playbooks. Configuration files specify the type of sports data each Feed Normalizer should normalize, sourced from feed providers. These configured sport-normalized feeds are then transmitted to the containerized Event Management Service, which subsequently inserts or updates the data in the Aurora DB.

## 🚀 Deployment strategy
The deployment strategy employed in this scenario is Blue/Green Deployment, with a primary focus on three Application Load Balancers (ALBs). The deployment process involves multiple pipelines, each serving a specific purpose.
1. First Pipeline:
   - Performs a Git repository checkout to retrieve the final migration folder used by developers.
   - The migration folder is utilized in the DB schema migration stage, where the Aurora DB schema migrates from the old schema to the new schema. The `schema.prisma` file holds the Aurora DB credentials for the migration.
   - Upon successful completion, the post-processing block triggers the Service Pipeline.
2. Service Pipeline:
   - Deploys Feed Normalizers, Frontend, and the Event Management service on the cloud.
   - In the Event Management stage, provisions the task definition, creates the service, and establishes a target group.
   - Continuous Delivery is not implemented, requiring manual verification of the deployment before proceeding to the next stage.
3. Final Pipeline:
   - Manually triggered after successful verification of the deployment.
   - Switches traffic for all ALBs (Frontend, Feed Normalizer, and Event Management) from Blue (old) Deployment to Green (new) Deployment.
   - The pipeline consists of multiple stages targeting each service ALB, allowing parallel execution to ensure a smooth transition from blue to green.
4. Rollback Pipeline:-
- The Rollback pipeline will include the traffic routing shifting from all the green deployments to the stable deployments again. 

This approach ensures a controlled and validated deployment process with manual verification steps before finalizing the switch to the new version.

