### https://docs.aws.amazon.com/sagemaker/latest/dg/usingamazon-sagemaker-components.html

cd SageMaker/eks-sagemaker-workshop/lab1

aws s3 cp s3://ee-assets-prod-us-east-1/modules/cb9c6461a4f440c69d8681bc5a2975a1/v1/lab1/mnist.pkl.gz .
aws s3 cp mnist.pkl.gz s3://$AWS_S3_BUCKET_NAME/lab1/source_data/mnist.pkl.gz

aws s3 cp kmeans_preprocessing.py s3://$AWS_S3_BUCKET_NAME/lab1/processing_code/kmeans_preprocessing.py
dsl-compile --py mnist_classification_pipeline.py --output mnist_classification_pipeline.tar.gz

aws s3 cp mnist_classification_pipeline.tar.gz s3://$AWS_S3_BUCKET_NAME/lab1/pipeline/mnist_classification_pipeline.tar.gz

# Go to S3 in the management console and download this file...
# echo "https://s3.console.aws.amazon.com/s3/object/$AWS_S3_BUCKET_NAME?region=$AWS_REGION&prefix=lab1/pipeline/mnist_classification_pipeline.tar.gz"
# https://s3.console.aws.amazon.com/s3/object/mod-cb9c6461a4f440c6-buildoutput-4jkh7sgyuugb?region=eu-west-1&prefix=lab1/pipeline/mnist_classification_pipeline.tar.gz


# Go to the Kubeflow Dashboard
# kubectl get ingress -n istio-system -o jsonpath="{..hostname}"
# Log in, confirm the "workshop" user name
# Import the new pipeline from file

### https://docs.aws.amazon.com/eks/latest/userguide/specify-service-account-role.html

aws eks describe-cluster --name $AWS_CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
## https://oidc.eks.eu-west-1.amazonaws.com/id/4E615CF252F46934096A9F0A035EA6BB
aws sts get-caller-identity --query Account --output text
## 654300125888

OIDC_URL="OIDC_URL_WITHOUT_HTTPS://"
AWS_ACCOUNT="ACCOUNT_NUMBER"
KUBEFLOW_USERNAME="workshop"

# Run this to create trust.json file

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

# Create the role, attach the policy and retrieve the ARN
aws iam create-role --role-name kfp-sagemaker-pod-role --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name kfp-sagemaker-pod-role --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam get-role --role-name kfp-sagemaker-pod-role --output text --query 'Role.Arn'
## arn:aws:iam::654300125888:role/kfp-sagemaker-pod-role

kubectl edit -n workshop serviceaccount default-editor

# Add the following annotation

# metadata:
#   annotations:
#     eks.amazonaws.com/role-arn: <role-arn>

# Save and exit with
# :wq

# Fetch S3 Bucket name
echo $AWS_S3_BUCKET_NAME 
## mod-cb9c6461a4f440c6-buildoutput-4jkh7sgyuugb

# Continue from the Kubeflow Dashboard
# Create Experiment
# Provide RoleARN (SageMaker Execution Role!) and S3 Bucket
### https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html
# Alternate between Kubeflow Dashboard and SageMaker Console

# Wait until everything completes. Get the Endpoint Name from Kubeflow
## Endpoint20201207170530-M49T


# Inference test script

cat <<EOF > test_inference.py
#!/usr/bin/env python

import pickle, gzip, numpy, json
import io
import boto3

ENDPOINT_NAME='<endpoint-name>'

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

chmod +x test_inference.py

./test_inference.py