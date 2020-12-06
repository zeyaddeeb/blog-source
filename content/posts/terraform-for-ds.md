---
date: 2020-12-05
title: "Data Scientists should starting using Terraform"
---

TL; DR:  Data scientists should manage their data infrastructure  with code. 
By doing so you can avoid being reliant on engineering departments and also keep better tabs on your costs.

I've had amazing opportunities throughout my career to work with all kinds of data scientists. I've found that not all of us are alike. You don't have to remember by heart how to test for [heteroscedasticity](https://en.wikipedia.org/wiki/Heteroscedasticity) in a linear regression model to be a good data scientist. Heck, you barely even need to remember the concept itself. Some data scientists are good with figuring out the best activation function for that type of neural network,  some are good with deployment and at automatically re-training and fine-tuning your models, and others are good with explaining the model's results and visualizing the outputs. Encountering a data scientist who can do, and likes doing, all of these things, would be a rare unicorn indeed.

If you're the kind of data scientist who likes managing their own data infrastructure, keep reading to learn how to deploy a scalable model.

Quick blanket disclaimer before diving into Terraform: I'm not saying here that [HCL (HashiCorp Configuration Language)](https://github.com/hashicorp/hcl) is the only way to manage infrastructure. I have my own frustrations with [Terraform](https://www.terraform.io/) at times, but overall it's a reliable and battle tested way of getting infrastructure up and running. There are many different tools available to help you accomplish the same goal, so if you're already using something like [Pulumi](https://www.pulumi.com/) that's great!

So what is Terraform, and why should you care? From the creators:

> Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

That's a lot to unpack, but for now I'll speak from a data scientist's perspective on why we should be using Terraform more and more.

Let's say you want to build a model that predicts whether a celebrity's name is mentioned in a some text (Name Entity Recognition). For example:

Input: `Tom Cruise is overrated`  
Output: `[text: Tom Cruise, label: Celebrity, confidence: 0.95]`

If you're a cool kid who already uses [Pytorch](https://pytorch.org/), you'd likely find yourself completing a laundry list that looks like this:

- write your model
- write your data loader
- write your training function

...of course all this happens only *after* you figure out how to do [`CyclicLR`](https://pytorch.org/docs/stable/optim.html#torch.optim.lr_scheduler.CyclicLR), because you got all hot and bothered by that one paper that mentioned it on [Arxiv](http://arxiv.org/).
At the end of all this trouble, you'll end up with something that looks like this (whatever code that is):

{{< highlight python >}}

import torch.nn as nn
import torch.nn.functional as F

class TransformerModel(nn.Module):
...

def train():
    model.train() # Turn on the train mode
    total_loss = 0.
    start_time = time.time()
    src_mask = model.generate_square_subsequent_mask(bptt).to(device)
    for batch, i in enumerate(range(0, train_data.size(0) - 1, bptt)):
        data, targets = get_batch(train_data, i)
        optimizer.zero_grad()
...
{{< /highlight >}}

Once you're done training and evaluating the model and plotting all the charts on how the loss/accuracy/mcc is performing between training/holdout datasets, if everything looks good, then you're ready to deploy this thing and run it on 100 million documents.

You move on to [`Deploying PyTorch Models in Production`](https://pytorch.org/tutorials/intermediate/flask_rest_api_tutorial.html) section of the Pytorch documentation. Here data scientists will have different opinions. Some will say, "this deployment stuff is not my job," and unceremoniously throw this task over the fence to someone with the word `engineer` in their title, instead opting to write some documentation on what is going on with `with torch.no_grad()` for predictions and the different [`LabelEncoder`](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.LabelEncoder.html) used to convert the labels, etc... Conversely, if you're actually interested in seeing what lies beyond that metaphorical fence, and think, "hey, this deployment stuff is interesting," then keep reading.

### Model deployment strategies

Regardless of what type of deployment strategy you choose, such as FlaskAPI or a fancy FastAPI web service, even if you are a hardcore data scientist who wants to run torchscript and the `C++` torchserve implementation to server your model, you'll still need to run this code somewhere that is always available and accessible to other services. Most companies now have a cloud environment of some sort, so let's assume we are using AWS (Amazon Web Services), just for simplicity's sake. Terraform can support all the major clouds (aka providers), and you can always check the list of clouds supported [here](https://www.terraform.io/docs/providers/type/major-index.html).

Having AWS (or any other cloud) gives you access to wide variety of services. It's not an understatement to say if you can spend all the money, the sky is the limit, check out the list of products offered [here](https://aws.amazon.com/products/). The core services a data scientist might use are:

- S3: data storage, where you can store your `model.pt` for example
- EC2: a server that will run your code
- Lambda: same idea as EC2 but you don't get to manage the server directly (aka. serverless execution)
- RDS: database (postgres, mysql) to store metadata about your models
- Redshift/EMR: data warehousing and big data stuff
- EKS: if you are into kubernetes like all the cool kids
- Sagemaker: a managed service for Machine Learning (it's worth while checking it out)

### Setting Up Your Model Infrastructure with Terraform & AWS

Installing terraform is a breeze. If you have Linux/MacOS, I recommend checking out [`tfenv`](https://github.com/tfutils/tfenv) to manage all the different versions of terraform.

Installation:

```bash
$ brew install tfenv && tfenv install latest
```

If you have a different system, checkout the install instructions for your operating system.

Now you can check which version of terraform you have by running

```bash
$ terraform version
# Terraform v0.13.5 --> This will be the latest version
```

Knowing which terraform version you've got is very important. It's like using Python2 vs Python3 in some cases and not all the features carry over, I'll assume you are using `>0.12` in the demo here.

Next, to set up AWS credentials, you'll need an AWS account, so you can talk to your administrator to create one, or, if you are the administrator, you'll know what to do. Once you have an account, you can get your AWS Access Key and Access Secret for AWS IAM page, and run the following commands:

```bash
$ pip install awscli && aws configure
```

Follow the setup instructions and then run a quick test:

```bash
$ aws s3 ls # this will list all s3 buckets in the region

2020-12-05 66:66:66 cool-bucket
```

Before diving deeper into how to deploy your celebrity named entity model, let's take a light example to get our feet wet.

### Introductory example: Setting up an S3 Bucket with Terraform

The [AWS terraform documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) is pretty extensive. Since Terraform supports multiple clouds, you always want to make sure you have the right provider.

To create a terraform deployment for any AWS service, you can break your code into multiple files or you can dump it all in one file. I'll be doing the one file example here for readability. Regardless of the method you choose, you always want to save your code to `main.tf`. While the name of your file is not that important, the extension `tf` is a must.

{{< highlight terraform >}}
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1" # you can change if you are in a different country/region
}

resource "aws_s3_bucket" "bucket" {
  bucket = "my-super-cool-tf-bucket"
  acl    = "private"

  tags = {
    Name        = "machine-learning" # tags are important for cost tracking
    Environment = "prod"
  }
}

{{< /highlight >}}

There are three main commands to run terraform in your terminal

`terraform validate` : will make sure your syntax is validated  
`terraform plan` : will produce the output of what is about to get executed  
`terraform apply` : will execute your code and create the resources you defined  

```bash
$ aws s3 ls | grep my-super-cool-tf-bucket

2020-12-05 66:66:66 my-super-cool-tf-bucket
```

### Deploying your Named Entity Model using Terraform

We'll be using AWS lambda to deploy the model because we want to reduce costs and ensure high concurrency. If you are trying to run predictions on 100 million documents, you need to do it in a scalable manner without breaking the bank. To achieve full scrooge status, I have previously set up infrastructure with GPU which costs a paltry $20/hour. But that was pretty  exhausting, so if you're looking for a quicker and easier approach, lambda's a good bet.

First off, you'll need to store your model coefficients on that s3 bucket you just created, you can do that by running

```bash
$ aws cp model.pt s3://my-super-cool-tf-bucket
```

You can also load your tokenizer and other files needed for running your predict function.

We will be using [`EFS`](https://aws.amazon.com/efs/) as a data store for the model since some of those models can be big and will not work with the Lambda limits otherwise.

Create a file called `providers.tf`

{{< highlight terraform >}}
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1" # you can change if you are in a different country/region
}

{{< /highlight >}}

Now we need to create the EFS in a file called `efs.tf` (this will also create a VPC along), if you have an existing VPC, [check out the data source on how to pull existing VPC information](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/vpc)

{{< highlight terraform >}}

resource "random_pet" "vpc_name" {
  length = 2
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = random_pet.vpc_name.id
  cidr = "10.10.0.0/16"

  azs           = ["us-east-1"]
  intra_subnets = ["10.10.101.0/24"]

  tags = {
    Name        = "machine-learning" # tags are important for cost tracking
    Environment = "prod"
  }

}

resource "aws_efs_file_system" "model_efs" {}

resource "aws_efs_mount_target" "model_target" {
  file_system_id  = aws_efs_file_system.shared.id
  subnet_id       = module.vpc.intra_subnets[0]
  security_groups = [module.vpc.default_security_group_id]

  tags = {
    Name        = "machine-learning" # tags are important for cost tracking
    Environment = "prod"
  }

}

resource "aws_efs_access_point" "lambda_ap" {
  file_system_id = aws_efs_file_system.shared.id

  posix_user {
    gid = 1000
    uid = 1000
  }

  root_directory {
    path = "/lambda"
    creation_info {
      owner_gid   = 1000
      owner_uid   = 1000
      permissions = "0777"
    }
  }

  tags = {
    Name        = "machine-learning" # tags are important for cost tracking
    Environment = "prod"
  }
}

{{< /highlight >}}

Now we need to create a DataSync to load the model from `s3` to the `efs` volume or `datasync.tf`

{{< highlight terraform >}}

resource "aws_datasync_location_s3" "s3_loc" {
  s3_bucket_arn = "arn" # copy the bucket arn you created in the previous step
}

resource "aws_datasync_location_efs" "efs_loc" {
  efs_file_system_arn = aws_efs_mount_target.model_target.file_system_arn

  ec2_config {
    security_group_arns = [module.vpc.default_security_group_id]
    subnet_arn          = module.vpc.intra_subnets[0]
  }
}

resource "aws_datasync_task" "model_sync" {
  name                     = "named-entity-model-sync-job"
  destination_location_arn = aws_datasync_location_s3.efs_loc.arn
  source_location_arn      = aws_datasync_location_nfs.s3_loc.arn

  options {
    bytes_per_second = -1
  }

  tags = {
    Name        = "machine-learning" # tags are important for cost tracking
    Environment = "prod"
  }
}

{{< /highlight >}}

We are almost done, create the lambda that runs the prediction, you will need your predict files and the requirements that needs to be installed (I'll assume you have a `model.py` and a `requirements.txt` files), now we can create `lambda.tf`

You'll need to make your code adapt the lambda syntax and load the model from `/mnt/shared-storage`, so `torch.load('/mnt/shared-storage/model.pt')`

and you'll need to handle the event `http/sns` (you can read more in the documentation)

```python
def predict(event, ctx):
   ...
```

To deploy the model, place the files in the same foler

{{< highlight terraform >}}

resource "random_pet" "lambda_name" {
  length = 2
}

module "lambda" {
  source = "terraform-aws-modules/lambda/aws"

  function_name = random_pet.lambda_name.id
  description   = "Named Entity Recognition Model"
  handler       = "model.predict"
  runtime       = "python3.8"

  source_path = "${path.module}"

  vpc_subnet_ids         = module.vpc.intra_subnets
  vpc_security_group_ids = [module.vpc.default_security_group_id]
  attach_network_policy  = true

  file_system_arn              = aws_efs_access_point.lambda.arn
  file_system_local_mount_path = "/mnt/shared-storage"

  tags = {
    Name        = "machine-learning" # tags are important for cost tracking
    Environment = "prod"
  }

  depends_on = [aws_efs_mount_target.model_target]
}

{{< /highlight >}}

You should be good to go, I glossed over some details on how to run the lambda, but you'll be able to stack-overflow your way through. Once you're done, cleanup is simple. All you have to do is run `terraform destroy` and it will remove all your resources.

### Key takeaways

1. You can prototype model training easily without having to rely on engineering departments. This could mean you can move more quickly if those teams are handling large backlogs, or maybe it means you get other cooks out of your kitchen to keep things tidy.
2. You get added control over your budget and cost tracking by using tags. Giving your team a solid ROI on data science costs helps justify your existence, which feels pretty good.

### References

- [https://aws.amazon.com/blogs/aws/new-a-shared-file-system-for-your-lambda-functions/](https://aws.amazon.com/blogs/aws/new-a-shared-file-system-for-your-lambda-functions/)
- [https://thenewstack.io/tutorial-host-a-serverless-ml-inference-api-with-aws-lambda-and-amazon-efs/](https://thenewstack.io/tutorial-host-a-serverless-ml-inference-api-with-aws-lambda-and-amazon-efs/)
- [https://github.com/terraform-aws-modules/terraform-aws-lambda/blob/master/examples/with-efs/main.tf](https://github.com/terraform-aws-modules/terraform-aws-lambda/blob/master/examples/with-efs/main.tf)
