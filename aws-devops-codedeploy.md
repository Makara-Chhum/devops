Deploy code to EC2 instances - you can use ansible, chef, terraform, etc... but you can also use managed service CodeDeploy
- Each EC2 (or on-prem machine) needs to run CodeDeploy agent - polls CodeDeploy for work
- CodeDeploy sends appspec.yml file
- Application pulled from GitHub or S3 - source code+appspec.yml file 
- Agent runs deployment instructions and report success/failure
- If restart EC2, need to restart the agent

Define an Application
- Each application defines the compute platform - EC2, Lambda, ECS
- One or more Deployment Groups - groups instances to deploy to, often dev/test/prod
- Deployment groups can specify different parameters like in-place vs blue/green, load balancer/ASG configs, rollback behavior, alerts, triggers
- Group instances by deployment group (dev/test/prod)
  - Flexibility, can have sub-groups
  - Chain into CodePipeline and use artifacts from there
  - Blue/green only works with EC2
  - CodeDeploy does not provision resources

So steps:
- Provision the EC2
- Make sure it has a role assigned with read access to S3
- Will want tags to direct CodeDeploy - Environment (Development) and Name (webserver)
- Create a CodeDeploy project
  - EC2/On-prem, lambda, or ECS
  - Create deployment group - for dev, test, prod
  - Deployment type - in-place or blue/green

appspec.yml
- version, OS sections
- files section - copy files from the local root (/index.html) to absolute path on the machine (/var/www/html)
- hooks differ by deployment target type (EC2/on-prem, ECS, lambda) https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html
  - ApplicationStop - runs first, it's from the PREVIOUS download. So can think of this as being the LAST hook, not the first one.
  - DownloadBundle - pull from S3, NO HOOK for this
  - BeforeInstall - set things up, install dependencies such as httpd
  - Install - copies files, NO HOOK
  - AfterInstall - configure application, set file permissions, etc.
  - ApplicationStart - start services you stopped during ApplicationStop (e.g. httpd)
  - ValidateService - validate that the service started OK
  - More hooks if you use a load balancer - BlockTraffic (NO HOOK, but Before/After), and AllowTraffic (same)
- In in-place deployment you have access to all the hooks, sequential on the same servers
- In blue/green, the steps are distributed among the old and new instances, and rollback events also happen
- There are also 5 environment variables available in hooks: APPLICATION_NAME, DEPLOYMENT_ID, DEPLOYMENT_GROUP_NAME, DEPLOYMENT_GROUP_ID, LIFECYCLE_EVENT
  - Lets you customize your deployment e.g. use the same appspec for dev and prod, and differentiate behavior by DEPLOYMENT_GROUP_NAME

Rollback/Redeploy - https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-rollback-and-redeploy.html
- Manual or automatic
- Manual rollback - go to a previous version, it just redeploys
- Automatic rollback - under Advanced, set up for a deployment group
  - Rollback when deployment fails - rollback on any deployment on any instance fails
  - Rollback when alarm thresholds met - let's say deployment goes fine; but after startup, CPU goes to 100%. So we want to rollback on alarm. 
    - Add alarm to deployment group. This in itself doesn't cause a rollback.
    - Can continue deployment ignoring alarms or if status unavailable
    - Then enable rollback when alarm trips

On-premise instances - https://docs.aws.amazon.com/codedeploy/latest/userguide/instances-on-premises.html
- Create on-prem instance, register with CodeDeploy, and TAG IT (important)
  - KEY IDEA #1: Need to have the on-prem instance log in. Could use an IAM user, but you need a separate one for EACH on-prem instance.
  - KEY IDEA #2: Need to register the instances, then TAG the instances to link them with specific deployment groups (dev/test/prod)
  - Better to use IAM role with STS, more scalable and more secure, but harder to set up. But more automatable.
  - Could use an EC2 instance, but don't give it a role. It's a standalone VM. Could be in Azure too.
  - IAM User or Role needs to have S3 read permissions (to read the appspec bundle)
  - Create a config codedeploy.onpremises.yml file with access key, secret key, user ARN, and region
  - Install AWS CLI with credentials
  - Install CodeDeploy agent
  - Register on-prem instances with CodeDeploy (AWS CLI - aws deploy register-on-premises-instance - not from the on-prem instance though)
  - Finally tag the instance - you use these tags in the CodeDeploy deployment group

CodeDeploy to Lambda
- KEY IDEA: You have an old and new version - key is how you shift traffic from the old to the new version
- Three options in deployment settings
  - 1. LambdaAllAtOnce - all the traffic goes from version 1 to version 2
  - 2. Canary - shifted in two increments - in first one, only shift X percent of traffic for Y minutes; then second
  - 3. Linear - shifted more gradually, equal increments and equal time between, e.g LambdaLinear10PercentEvery1Minute
  - Note that these latter two use Lambda versions and aliases to actually do the shift!
- Linear is smoother, easier to monitor
- Still have triggers, alarms, rollback options
- There is an appspec.yml file for Lambda, but much simpler
  - Just two hooks: BeforeAllowTraffic, AfterAllowTraffic
  - Each hook specifies another Lambda function to run
  - BeforeAllowTraffic - check DB connectivity, do DB migrations
  - AfterAllowTraffic - health check, how to verify Lambda was deployed right