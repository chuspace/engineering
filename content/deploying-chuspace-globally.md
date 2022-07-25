---
title: Deploying Chuspace globally using Docker
summary: Outlining thought process for deploying simple Rails app globally
date: 27-06-2022
topics: Rails, AWS, Digitalocean, Deployment
---

Rails app deployment is a hard problem but deploying globally, and close to the reader is even harder. In this edition, we use [Docker](https://www.docker.com/) instead of machine images for Rails application deployment.

*A bit of a recap*, 

Chuspace is a simple Rails app powered by a MySQL database and Memcached running on each node. MySQL database is managed and provided by [PlanetScale](https://planetscale.com/) with replicas spread across the globe. The previous version used Terraform with Packer to build and deploy pre-built machine images.

Issues with [previous](/engineering/deploying-chuspace-globally/editions/1) version:

1. Complex - building and tracking multi-provider/region machine images on each deploy was slow and complicated. I had to manually copy new images to each region and build seperately for each cloud provider. Autoscaling was hard.

2. Deploy downtime (up to 10 mins)

## Docker way

I avoided docker to keep things simple and low overhead however it made the deployment process very complex, mainly managing VM images on multiple providers.

[Docker](https://www.docker.com/) solves the application portability problem by turning an application code into a deployable artifact that can be run on any supported platform.

**How this new flow works?**

There is still Terraform in the picture because it’s just great for managing multiple infrastructure resources. [Github CI](https://github.com/features/actions) uses terraform to spin new VMs on each code deploy with a pre-configured user data that installs [Docker](https://www.docker.com/) and runs the application using docker-compose. There is also a systemd file that restarts Docker containers in case of machine restarts.

The terraform module looks like this,

```tf
##############################################
# Provider: AWS
# Region: EU
##############################################

module "aws_eu_west" {
  source = "./aws"
  region = "eu-west-1"

  server_count              = 2
  logtail_token             = var.logtail_token
  aws_access_key_id         = var.aws_access_key_id
  aws_secret_access_key     = var.aws_secret_access_key
  aws_region                = "eu-west-1"
  aws_ssm_secret_key_name   = "eu-foobar"
}

##############################################
# Provider: DO
# Region: US
##############################################

module "do_us_west" {
  source = "./do"
  region = "sfo3"

  server_count              = 2
  do_token                  = var.do_token
  logtail_token             = var.logtail_token
  docker_access_token       = var.docker_access_token
  aws_access_key_id         = var.aws_access_key_id
  aws_secret_access_key     = var.aws_secret_access_key
  aws_region                = "eu-west-1"
  aws_ssm_secret_key_name   = "us-foobar"
}
```

Each new replacement VM is created first then destroyed to reduce downtime,

```tf
 lifecycle {
    create_before_destroy = true
  }
```

Application environment variables are managed via [AWS secrets manager](https://aws.amazon.com/secrets-manager/) as you may have noticed above and is injected on demand before running the docker containers,

```bash
(aws secretsmanager get-secret-value --secret-id example-app/some-env/${aws_ssm_secret_key_name} | jq '.SecretString' | xargs printf) > .env
```

The logs are aggregated using [Logtail](https://betterstack.com/logtail) from multiple running containers,

```bash
curl -1sLf \
  'https://repositories.timber.io/public/vector/cfg/setup/bash.deb.sh' \
  | sudo -E bash

sudo apt-get install -y  vector=0.22.3-1
sudo wget -O /etc/vector/vector.toml https://logtail.com/vector-toml/docker/${logtail_token}
sudo usermod -a -G docker vector
sudo systemctl restart vector
```

Monitoring is done using Weave [scope](https://www.weave.works/oss/scope/), which also has a nice UI for exec’*ing* into containers when needed, for example, to run rails console.

![Weave scope UI](/assets/screenshot-2022-07-25-at-214321.png)

Cloudflare manages the incoming traffic using a network load balancer, which routes traffic based on proximity. If a pool becomes unhealthy during deploys, all traffic gets re-routed to the primary region. Primary region is deployed last.

The deployment happens in two steps using Github CI and rake tasks - step one to all regions except the primary and then to primary. I think there is room for improvement here but I haven’t come up with anything solid yet. Docker makes the deployment process simple and easy to reason with. However, there is still some serious downtime during deploys - ***30 seconds***. I can mitigate that by keeping the old resources around a bit longer and then swapping the traffic using the application load balancer as described in Terraform blue-green deployment tutorial.

## In-between thoughts

**Should I use docker swarm?**

Maybe? But I have few unique env variables per region so not sure how that would work in swarm mode.

**Should I use hashicorp Nomad?**

Maybe? But for production deploys it requires few components like [Consul](https://www.consul.io/), [Vault](https://www.vaultproject.io/) and I don’t want to learn all that now.

Not sure how per region env variable would work here either.

---

## Inspirations and Credits

[Hashicorp tutorial](https://www.hashicorp.com/blog/terraform-feature-toggles-blue-green-deployments-canary-test)

[Terraform](https://www.terraform.io/)

[Docker](https://www.docker.com/)

[Weave cloud scope](https://www.weave.works/oss/scope/)

[Logtail](https://betterstack.com/logtail)