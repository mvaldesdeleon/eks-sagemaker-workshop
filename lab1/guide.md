# Lab 1

## Overview

For this lab, you will build, deploy and run a Kubeflow Pipeline that integrates with SageMaker leveraging the SageMaker Components for Kubeflow Pipelines.

You can find more information about this here: [https://docs.aws.amazon.com/sagemaker/latest/dg/usingamazon-sagemaker-components.html](https://docs.aws.amazon.com/sagemaker/latest/dg/usingamazon-sagemaker-components.html)

## Start

Using the personal access link provided, log into the Event Engine platform. Click on the "AWS Console" button, and then on the "Open AWS Console" button to access the AWS Management Console. Please remain in `eu-west-1` throughout the entire lab.

Once in the AWS Management Console, navigate to [Amazon SageMaker](https://eu-west-1.console.aws.amazon.com/sagemaker/home?region=eu-west-1#/). From the left-hand side menu, find "Notebook" and select "Notebook instances".

You should find a single Notebook Instance provisioned for you, which you will be using as your workstation. Click on "Open JupyterLab" under "Actions" to access your Notebook instance and get started by navigating to the directory of the first lab:

```sh
cd ~/SageMaker/eks-sagemaker-workshop/lab1
```

## Upload MNIST dataset

The pipeline for this lab will train a classification model using k-means clustering with the MNIST dataset.

All the assets for the pipeline will be stored in S3, on a bucket that was created as part of your environment. The name of this S3 bucket is available in the `AWS_S3_BUCKET_NAME` environment variable.

Download the MNIST dataset and upload it to your S3 bucket:

```sh
aws s3 cp s3://ee-assets-prod-eu-west-1/modules/cb9c6461a4f440c69d8681bc5a2975a1/v1/lab1/mnist.pkl.gz .
aws s3 cp mnist.pkl.gz s3://$AWS_S3_BUCKET_NAME/lab1/source_data/mnist.pkl.gz
```

## Review the Kubeflow Pipeline definition

The Kubeflow Pipeline definition you will deploy is in the `mnist_classification_pipeline.py` file. Take a moment to review the Pipeline, its stages and parameters, and the dependency graph. The Pipeline consists of a data Processing job, a Hyperparameter Tuning job, a Training job. The resulting model artifact is then imported into SageMaker, and used to deploy an Inference endpoint as well as running a Batch transform job.

## Compile and export the Pipeline artifact

Before you compile the Kubeflow Pipeline, you will need to upload the data processing script to your S3 bucket:

```sh
aws s3 cp kmeans_preprocessing.py s3://$AWS_S3_BUCKET_NAME/lab1/processing_code/kmeans_preprocessing.py
```

You can now compile the Kubeflow Pipeline, and upload the resulting artifact to your S3 bucket:

```sh
dsl-compile --py mnist_classification_pipeline.py --output mnist_classification_pipeline.tar.gz
aws s3 cp mnist_classification_pipeline.tar.gz s3://$AWS_S3_BUCKET_NAME/lab1/pipeline/mnist_classification_pipeline.tar.gz
```

## Download the Pipeline artifact

You will need to download a copy of this file to your local computer, so you can import it into the Kubeflow Pipelines UI.

From the AWS Management Console, navigate to [S3](https://s3.console.aws.amazon.com/s3/home?region=eu-west-1). You should find a single S3 bucket. Select it and continue to navigate to the pipeline artifact location: `lab1/pipeline/mnist_classification_pipeline.tar.gz`. Locate the "Object actions" button on the top-right corner, and select "Download", under "Download actions". Save this file locally and take note of its location.

## Import the Pipeline into Kubeflow

You can access the Kubeflow Dashboard from the Application Load Balancer that was created as part of the Kubeflow deployment. You can retrieve the hostname directly from the associated `ingress` resource:

```sh
kubectl get ingress -n istio-system
```

Copy the value under `ADDRESS` and paste it on your browser. You should see the Kubeflow Dahsboard login page. The credentials will be provided to you during the lab. Log in, and click through the initial configuration steps. Do not change the `workshop` username.

From the left-hand side menu, select "Pipelines", and then click on the "+ Upload pipeline" button.

Provide a name and a description, select "Upload a file", click on "Choose file" and select the pipeline artifact file you downloaded on the previous step. Click "Create" to create the pipeline.

## Configure Service Account and IAM Role for Kubeflow Pipelines

Before you can run the Kubeflow Pipeline, you will need to create an IAM Role that Kubeflow Pipelines can use to communicate with SageMaker. You will then [associate this IAM Role with the Kubernetes service account](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) that Kubeflow Pipelines will use when deploying the Pods for your Pipeline. This lets you provide AWS access with Pod-level granularity, rather than assigning the permissions directly to the underlying EC2 compute nodes and thus to all Pods running on said nodes.

Start by retrieving the OIDC Issuer URL for the cluster's identity provider:

```sh
aws eks describe-cluster --name $AWS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
```

You will also need to retrieve your AWS account number:

```sh
aws sts get-caller-identity --query Account --output text
```

Update the following definitions with the values you obtained. You will need to remove the schema (`https://`) from the OIDC Issuer URL:

```sh
OIDC_URL="oidc.eks.eu-west-1.amazonaws.com/id/0123456789ABCDEF0123456789ABCDEF"
AWS_ACCOUNT="123456789012"
KUBEFLOW_USERNAME="workshop"
```

Create the trust policy file. This policy specifies that this IAM Role can only be assumed from the EKS cluster, and only by the specific Service Account that Kubeflow Pipelines will use.

```sh
cat <<EOF > trust.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT}:oidc-provider/${OIDC_URL}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_URL}:aud": "sts.amazonaws.com",
          "${OIDC_URL}:sub": "system:serviceaccount:${KUBEFLOW_USERNAME}:default-editor"
        }
      }
    }
  ]
}
EOF
```

You can now create the IAM Role, attach the [AmazonSageMakerFullAccess](https://console.aws.amazon.com/iam/home?region=eu-west-1#/policies/arn:aws:iam::aws:policy/AmazonSageMakerFullAccess) managed policy to it, and retrieve its ARN:

```sh
aws iam create-role --role-name kfp-sagemaker-pod-role --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name kfp-sagemaker-pod-role --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam get-role --role-name kfp-sagemaker-pod-role --output text --query 'Role.Arn'
```

Annotate the Kubernetes service account with the IAM Role ARN. Be sure to replace `<role-arn>` with the ARN of the IAM Role you just created.

```sh
kubectl annotate -n workshop serviceaccount default-editor eks.amazonaws.com/role-arn='<role-arn>'
```

## Retrieve the SageMaker Execution Role ARN

The IAM Role you just created will be used by Kubeflow Pipelines to create the individual SageMaker tasks, but each of these tasks will in turn assume a different IAM Role called the [SageMaker Execution Role](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html). This is the role, for example, that must have access to the S3 bucket containing the training data, and where the model artifacts will be stored.

A SageMaker Execution Role has been created for you already, and you will need to retrieve its ARN. To do so, navigate to the [IAM](https://console.aws.amazon.com/iam/home?region=eu-west-1) console.

From the left-hand side menu, select "Roles", and search for "SageMakerExecutionRole". You should get a single result.

Select this role, inspect its permissions and compare them with the permissions provided for the Kubeflow Pipelines role on the previous step.

Once you're done, take note of the Role ARN.

You will also need the name of your S3 Bucket, which you can retrieve from its respective environment variable:

```sh
echo $AWS_S3_BUCKET_NAME 
```

## Run the Kubeflow Pipeline

Switch back to the Kubeflow Dashboard. From the left-hand side menu, select "Pipelines", then select the pipeline you created previously. Click on the "+ Create experiment" button.

Provide a name and a description, and click "next".

Scroll down to the "Run parameters" section, and provide the SageMaker Execution Role ARN, as well as your S3 Bucket name. Click "Start" to start run the pipeline.

**IMPORTANT: Make sure to provide the SageMaker Execution Role ARN and not the ARN of the Kubeflow Pipelines role you created**

You can now alternate between the Kubeflow Pipelines UI and the [Amazon SageMaker](https://eu-west-1.console.aws.amazon.com/sagemaker/home?region=eu-west-1#/) console. As the pipeline run progresses, you will be able to see the respective SageMaker jobs being created and executed. The relevant outputs from each stage will be available directly from the Kubeflow Pipelines UI.

The pipeline run will take about 20 minutes to complete.

## Test the Inference endpoint

From the Kubeflow Pipelines UI, locate the "SageMaker - Deploy Model" stage, and select it. Within the pop-over window, scroll down to "Output arficats" and locate the Endpoint Name, which should look like this: `Endpoint20201207170530-M49T`.

Update the following definition with this value:

```sh
INFERENCE_ENDPOINT_NAME="Endpoint20201207170530-M49T"
```

Create the test script, using `boto3` to invoke the inference endpoint directly:

```sh
cat <<EOF > test_inference.py
#!/usr/bin/env python3

import pickle, gzip, numpy, json
import io
import boto3

ENDPOINT_NAME='$INFERENCE_ENDPOINT_NAME'

# Load the dataset
with gzip.open('mnist.pkl.gz', 'rb') as f:
    train_set, valid_set, test_set = pickle.load(f, encoding='latin1')

# Simple function to create a csv from our numpy array
def np2csv(arr):
    csv = io.BytesIO()
    numpy.savetxt(csv, arr, delimiter=',', fmt='%g')
    return csv.getvalue().decode().rstrip()

runtime = boto3.client('sagemaker-runtime')

payload = np2csv(train_set[0][30:31])

response = runtime.invoke_endpoint(EndpointName=ENDPOINT_NAME,
                                   ContentType='text/csv',
                                   Body=payload)
result = json.loads(response['Body'].read().decode())
print(result)
EOF
```

Assign the right permissions and run the test script:

```sh
chmod +x test_inference.py

./test_inference.py
```