q1.How do you implement rollback strategy in ECS?

ans.. In Amazon ECS, rollback strategies are about ensuring that if a new deployment fails (for example, a bad image, misconfig, or health check failures), the service automatically or 

manually reverts to the previous working task definition.Here’s how rollback strategies can be implemented in ECS (both ECS on EC2 and ECS on Fargate):
1. Using CodeDeploy with ECS (Blue/Green)

If you deploy services with AWS CodeDeploy + ECS Blue/Green (often via CodePipeline):

CodeDeploy shifts traffic from the old (blue) task set to the new (green) one gradually or all-at-once.

Health checks (from ELB/Target Group) validate if the new tasks are healthy.

If deployment fails (health check timeout, alarms triggered):

CodeDeploy automatically stops the green deployment.

Rolls back traffic to the previous (blue) task set.

ECS retains the last known good task definition.

You can configure CloudWatch alarms or manual approval in CodeDeploy to trigger rollback.

✅ Best for zero-downtime + safe rollback.

2. ECS Deployment Circuit Breaker (Recommended)

Amazon ECS has a built-in deployment circuit breaker:

Configured in the ECS service definition:

"deploymentConfiguration": {
  "deploymentCircuitBreaker": {
    "enable": true,
    "rollback": true
  }
}


If ECS detects tasks failing to start or failing load balancer health checks:

Circuit breaker stops the deployment.

ECS rolls back to the last working task definition automatically.

✅ Best for simple, fast rollback without CodeDeploy.

3. Manual Rollback via Task Definition Versions

Every ECS deployment creates a new task definition revision:

If new deployment fails, you can manually:

Open the ECS service.

Select a previous task definition revision.

Force a redeploy (Update Service).

Can be automated using a script/CI/CD pipeline:

aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --task-definition my-task:12


✅ Simple, but requires manual intervention or automation.

4. CI/CD Driven Rollback (CloudWatch + Lambda)

Use CloudWatch alarms (e.g., on Application Load Balancer 5xx, ECS task failures, custom metrics).

Trigger a Lambda function that:

Calls aws ecs update-service with the last known good task definition.

Can be integrated with CodePipeline for automatic rollback triggers.

✅ Useful for custom health criteria beyond ECS/ELB.


q2.How do you achieve zero downtime in EC2-based deployments?

Achieving zero downtime deployments on EC2-based environments means serving user traffic continuously while rolling out new application versions. 
Since EC2 doesn’t have built-in deployment orchestration like ECS/EKS, you need to carefully design your deployment strategy. Here are the main approaches:

1. Load Balancer with Blue-Green Deployment

Setup:

Have two environments (Blue = current, Green = new).

Both are behind an Elastic Load Balancer (ELB/ALB) or target groups.

Process:

a.Deploy new version (Green) to a fresh set of EC2 instances or ASG.
b.Run health checks to ensure Green is stable.

c.Switch ELB traffic from Blue → Green.

d.Keep Blue running for rollback safety.

Rollback: Switch traffic back to Blue.

2. Rolling Deployment with Auto Scaling Groups

Setup:

Application is running in an Auto Scaling Group (ASG) behind a Load Balancer.

Process:

Launch new EC2 instances (with updated AMI or user-data for app).

ELB health checks ensure only healthy new instances serve traffic.

Gradually terminate old instances while scaling in new ones.

Rollback: Halt rollout, terminate bad instances, redeploy old AMI.

✅ Lower infra overhead than Blue-Green.
❌ Slower rollback if errors slip through.

Run health checks to ensure Green is stable.

Switch ELB traffic from Blue → Green.

Keep Blue running for rollback safety.

Rollback: Switch traffic back to Blue.

✅ Fast rollback, predictable.
❌ Requires more infrastructure temporarily.

3. Canary Deployment

Setup:

Small subset of EC2 instances with new version, rest with old.

Process:

Deploy new version to 1–2 instances in the ASG.

Route small % of traffic (using ALB weighted target groups or Route53 weighted DNS).

Gradually increase traffic split if metrics look good.

Rollback: Remove canary instances, keep old version serving traffic.

✅ Early detection of issues.
❌ Needs strong monitoring & routing controls.
