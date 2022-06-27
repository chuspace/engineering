---
title: Deploying Chuspace Globally
summary: Outlining thought process for deploying simple Rails app globally
date: 27-06-2022
topics: Rails, AWS, Digitalocean, Deployment
---

App deployment is a hard problem but deploying globally, and close to the reader is even harder.

Chuspace is a simple Rails app powered by a MySQL database and Memcached running on each node. MySQL database is managed and provided by [PlanetScale](https://planetscale.com/) with replicas spread across the globe. There is no database migration involved during deployment and ActiveJob is powered by Amazon SQS so all quite simple.

### Easy way

I searched for opensource [PAAS](https://github.com/search?o=desc&q=paas&s=updated&type=Repositories) options on Github and came across [Convox](https://convox.com/). Convox rack v3 is based on [Kubernetes](https://kubernetes.io/) and is available on AWS, DigitalOcean, Google Cloud, and Azure. Also, they don't charge you based on how many clusters you can add but instead on deployment workflows. You can create upto two workflows in the free plan. I had to do some firefighting to setup SSL and DNS, but otherwise, everything worked perfectly on Convox. I had a multi-region/cloud K8s cluster setup in the following regions: EU, Asia, and the America. DNS and Application load balancing was provided by [Cloudflare](https://www.cloudflare.com/en-gb/load-balancing/) using Geo-Steering.

I was happy for the day, but when I looked at monthly costs, it was outrageous - on AWS it was $180 per month per region. On the digital ocean, it was around $72 per region. For an early stage startup this wasn't sustainable. I also looked at Qovery and came upon the same issue.

I looked into Nomad + Waypoint, which seemed like a good option for edge deployments until it wasn't that easy to setup for production workloads. I didn't want to spend too much time doing DevOps.

### Hard way

After some research, I settled on Terraform for infrastructure management and good'ol SSH for deployments using a gem called [Tomo](https://github.com/mattbrictson/tomo). I think, Terraform is the best thing happened to infrastructure management. Otherwise, it would have been a nightmare to provision and manage all the resources on gigantic cloud platforms like AWS or GCP.

As of now, I have a Terraform module that provisions instances on-demand on a cloud provider with a base image preconfigured with monitoring tools and app code deployed using [Tomo](https://github.com/mattbrictson/tomo) and built using [Packer](https://www.packer.io/)

The top-level terraform module looks like this, which works at the provider level:

```hcl
module "aws_apac" {
  source = "./aws/apac"
  # ... options
}

module "do_us" {
  source = "./do/us"
  # ... options
}

module "gcp_canada" {
  source = "./gcp/canada"
  # ... options
}
```

And the submodules are resource-specific, which looks like this:

```hcl
module "eu_west_servers" {
  source = "../shared/server"

  region = "eu-west-1"
  name   = "hello-world"

  # VPC etc...

  instance_type        = "t3.small"
  instance_count       = var.instance_count
}
```

```hcl
module "us_sfo_servers" {
  source = "../server"

  region = "sfo3"
  name   = "hello-world"

  do_token             = var.do_token
  instance_type        = "s-1vcpu-2gb-intel"
  instance_count       = var.instance_count
}
```

```hcl
resource "aws_instance" "app" {
  # ...

  lifecycle {
    create_before_destroy = true
  }
}
```

For deployment, [Tomo](https://github.com/mattbrictson/tomo) reads Public IP addresses from Terraform output and then deploys code to the currently running instances in seconds:

```rb
environment :eu_central do
  host "instance1@1.2.3.4"
  host "instance2@1.2.3.5"
end

environment :ap_south do
  host "instance1@1.2.3.4"
  host "instance2@1.2.3.5"
end
```

In the event of load, more resources can be provisioned using terraform by increasing the instance count and if needed latest code can be deployed using Tomo. Since the base image always includes one deployment, subsequent deploys are very quick - all it needs is the latest code. Whenever Ruby version is updated, I create a new base image using packer and then run the deploy setup using Tomo so that if I have to scale up servers it's quick.

Most steps are manually run, but it's simple and good enough. I think I will convert this setup into a deployment tool in the future, but I don't have time now.


### Inspirations and Credits

[Hashicorp tutorial](https://www.hashicorp.com/blog/terraform-feature-toggles-blue-green-deployments-canary-test)
[Terraform](https://www.terraform.io/)
[Packer](https://www.packer.io/)
[Tomo](https://github.com/mattbrictson/tomo)





