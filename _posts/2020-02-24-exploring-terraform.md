---
title: Exploring Terraform
description: A quick journey into the world of Terraform.
permalink: "/blog/exploring-terraform"
---

These days, with everything in the mystical cloud (read: someone else's server), we're needing to manage a lot of infrastructure. We need to manage spinning up new instances, letting them communicate, grabbing objects from an S3 bucket, putting up a load balancer in front of our app, and the list goes on.

With that complexity comes plenty of headaches if we did it manually. _Enter: Infrastructure-as-Code (IaC)._ With IaC, we can define our infrastructure programmatically. This makes it faster to deploy, scalable, and less error-prone than the days of old. We can even version control our configuration and roll back to a previous state should something go awry.

## Different tools for the job

There are a number of players in the space, such as Terraform, Ansible, Chef, and Puppet. They all do things a little differently. Let's discuss a couple of them.

**Mutability**. This property refers to whether a configuration can be mutated, or altered, in-place. If a configuration can be mutated in-place, that means that the server doesn't have to go down to deploy a new change. However, this also makes _configuration drift_ more likely, whereby the configuration of one instance may not mirror the configuration of another instance that was intended to be identical.

For example, if someone goes and updates some piece of software on one instance, this causes that instance to become misaligned as it relates to other instances that were spun up with the same conditions. This can make life very tricky if you're expecting a certain, static set of conditions to be upheld by all instances that were defined by the same configuration initially.

**Execution method.** Tools follow either a procedural or declarative style. In the procedural style, you write code in a way that tells the tool to configure your infrastructure exactly as you say it and in the order you say it. On the other side, declarative-style tools have you write configurations that define the overall picture of your infrastructure. The tool then figures out to make that happen as the end state. As a result, declarative-style tools tend to be less confusing as they don't require knowledge of what has previously been configured.

## Installation

The installation is comically simple. You [grab a binary](https://www.terraform.io/downloads.html) for your operating system and architecture, unzip it, and move it to a directory of your choosing. You can find the SHA256 checksums on the downloads page above as well to ensure you've got a clean copy. On Windows, you can generate the checksum by running `CertUtil -hashfile C:\path\to\file SHA256`, which simply assumes it's on your `C:` drive, but does not require it.

> _Note:_ Either ensure the directory you chose is in your system's `PATH` or add it to your `PATH`.

## Deploying my first server

Using AWS, I created a user through [Identity and Access Management](https://console.aws.amazon.com/iam/home) (IAM). As an aside, AWS recommends that you not use the root account unless strictly necessary and instead create an administrator account to perform actions that require elevated privileges. This account will then, in turn, create lesser accounts with only the privileges they need to do the job. For example, when deploying this server, I only needed a user with the `AmazonEC2FullAccess` permission. This allowed for the creation and destruction of EC2 instances, as well as associated security group actions.

> Use `terraform init` to initialize Terraform in a new directory

Assuming the deployment of a single server with no additional bells, whistles, or even functionality, my `main.tf` looked like this:

```HCL
provider "aws" {
    region = "us-east-2"
}

resource "aws_instance" "test" {
    ami             = "ami-0fc20dd1da406780b"
    instance_type   = "t2.micro"
}
```

Let's break this down, block by block.

1. We declared the provider as `aws` and set the region to `us-east-2`.
2. We want to spin up an `instance` on `aws` and call it `test`. We then choose an Amazon Machine Image (AMI) and say we want to spin up an instance of type `t2.micro`.

The alphanumeric identifier after `ami-` is unique to each region, so make sure you're matching those with the right region.

To preview these changes, you can run `terraform plan`. If everything goes well, you'll see a list that shows the changes to the configuration that Terraform will make. Since this was my first deployment, everything shows as a green `+`, indicating that it is an addition. The output gives you a handy reminder at the top of what each symbol means if you ever forget.

Much of the information isn't known until it applies the changes, so don't worry about all the `(known after apply)` throughout the output.

To apply the changes, run `terraform apply`. `apply` gives the same preview that `plan` did, so it isn't actually necessary to run `plan` before running `apply`. To confirm the changes and actually push the deployment, confirm the changes by typing 'yes'.

It'll take a minute to apply and then a couple minutes to actually finish up on the AWS side. At this point, there's nothing that this server does currently, so it isn't too exciting, but it is deployed. You can check on its status by navigating to your [EC2 instances](https://us-east-2.console.aws.amazon.com/ec2/v2/home?region=us-east-2#Home:) page. (That link corresponds to the entered region of `us-east-2` from `main.tf` above, so make sure you're in your region before wondering where your instance is.)

To destroy the infrastructure, use the aptly-named `terraform destroy`.

## Building up complexity

From here, you can have your server spit back a simple, "Hello, world," set up security groups to specify incoming and outgoing traffic restrictions, set up a load balancer, or implement an auto-scaling group to automatically manage the scaling of instances.

From here, you can do a myriad of tasks, such as:

- Have the instance spit back a simple, "Hello, world"
- Set up security groups to specify incoming and outgoing traffic restrictions
- Stand up a load balancer to distribute traffic over multiple instances with health checking
- Implement an auto scaling group to manage dynamic resource needs and self-heal clusters

A great resource for going through all of this is to check out Gruntwork co-founder Yevgeniy Brikman's [Terraform tutorial](https://blog.gruntwork.io/a-comprehensive-guide-to-terraform-b3d32832baca) over on the Gruntwork blog.

## Variable management

One particular pain point I noticed while going through the above was that of variable management. While what I configured did not require many variables, it got me thinking about how variables are managed and if that management followed the Don't Repeat Yourself (DRY) principle.

On the Amazon Partner Network Blog, Josh Campbell and Brandon Chavis illustrate [variable management in Terraform](https://aws.amazon.com/blogs/apn/terraform-beyond-the-basics-with-aws/) utilizing modules. What immediately struck me was that every module (specified as a subdirectory) had to manage a `variables.tf` file and there was a redundant interplay of input and output variables in various places.

In their example, utilizing `webapp_elb_name` from the `load_balancers` module required the following chain:

1. `main.tf` - define the name of the variable, `webapp_elb_name`, and where it comes from (`module.load_balancers.webapp_elb_name`)
2. `load_balancers/webapp-elb.tf` - the output variable `webapp_elb_name` is tied to a value that is defined inside this configuration file
3. `autoscaling_groups/variables.tf` - the input variable `webapp_elb_name` is declared for use in this module
4. `autoscaling_groups/webapp-asg.tf` - the `load_balancers` variable of the `aws_autoscaling_group` resource is set to an array containing the variable known to the file as `webapp_elb_name`

At first glance, step 3 seems like it could be extraneous. Couldn't we just output the variable from step 2 and immediately use it in step 4?

By default, variables are in the local scope. So, this can explain why variables must be designated as output or input variables. It certainly seems like a good idea to explicitly call out those variables that can be utilized outside the file in which they were defined, but it seems like it might be acceptable to allow variables that are already designated as output variables to be used as input variables elsewhere without having to call that fact out.

In searching for a solution to the issue, it seems Gruntwork (funnily enough, given that they wrote the resource recommended in the previous section) set out to do just this. They have a great post covering how to [keep your Terraform code DRY](https://davidbegin.github.io/terragrunt/use_cases/keep-your-terraform-code-dry.html) by utilizing `.tfvars` files and remote configurations. It's well worth a read to see how to handle the complexity that arrives with variable management in larger projects.

## Custom validation rules

I happened to stumble upon this and just enjoy when policies can be enforced so to prevent mishaps in the first place.

This is an `experimental` feature and requires explicit opt-in through the following:

```HCL
terraform {
    experiments = [variable_validation]
}
```

You can use it to ensure variables are being set correctly and provide error messages to assist others in correcting those issues.

With a quick example, the Terraform docs cover [custom validation rules](https://www.terraform.io/docs/configuration/variables.html#custom-validation-rules). To save you the click, I'll reproduce their example below.

```HCL
variable "image_id" {
    type        = string
    description = "The id of the machine image (AMI) to use for the server."

    validation {
        condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
        error_message = "The image_id value must be a valid AMI id, starting with \"ami-\."
    }
}
```

Expressions for conditions may only reference the variable that condition applies to. However, if the condition can fail (not return `false`), it must be wrapped by `can()`.

## Additional tips

- Variables can be set, in order of increasing priority, by:
    - environment variables
    - `.tfvars`
    - `.tfvars.json`
    - `*.auto.tfvars`, `*.auto.tfvars.json`
    - `-var`, `-var-file`

- `terraform fmt` - lines up all those `=` so you don't have to (works in current directory only)
- `terraform validate` - checks files in the current directory for syntax and consistency

I'm excited to keep learning about Terraform and figuring out the breadth and depth that it has. It also would pair well with other tools, such as Packer, Docker, and Kubernetes.