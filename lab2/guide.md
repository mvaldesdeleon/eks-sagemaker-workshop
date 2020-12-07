## Lab 2 on using Amazon SageMaker components in Kubeflow Pipelines

This lab contains two examples: `kfp-sagemaker-script-mode.ipynb` and `kfp-sagemaker-custom-container.ipynb`.

Both examples implement the pipeline below using the mentioned Kubeflow components. It is recommended to start with the script-mode example using the Tensorflow framework container:  
https://github.com/aws/deep-learning-containers/blob/master/available_images.md  
The custom-container example extends this container and uses the custom docker container instead.


## First component:
https://github.com/kubeflow/pipelines/tree/master/components/aws/sagemaker/hyperparameter_tuning  
Runs an Amazon SageMaker hyperparameter tuning job to optimize the following hyperparameters:

* learning-rate: [0.0001, 0.1] log scale
* optimizer : [sgd, adam]
* batch-size: [32, 128, 256]
* model-type: [resnet, custom model]

**Input**: N/A <br>
**Output**: best set of hyperparameters

## Second component:
Custom python function operator using kfp.components.func_to_container_op
In the current step the best set of hyperparameters are taken and the number of epochs of the trainig job is increased.

**Input**: best hyperparameters <br>
**Output**: best hyperparameters with updated epochs (20)

## Third component:
https://github.com/kubeflow/pipelines/tree/master/components/aws/sagemaker/train  
Run an Amazon SageMaker training job using the best set of hyperparameters with a higher number of epochs (we still keep this number relatively low here to reduce training time).

**Input**: best hyperparameters with updated epochs (20) <br>
**Output**: training job name

## Fourth component:
https://github.com/kubeflow/pipelines/tree/master/components/aws/sagemaker/model   
Create an Amazon SageMaker model artifact containing the serving code and the model itself.

**Input**: training job name <br>
**Output**: model artifact name

## Fifth component:
https://github.com/kubeflow/pipelines/tree/master/components/aws/sagemaker/deploy   
Deploy a model on an Amazon SageMaker enpoints.

**Input**: model artifact name <br>
**Output**: N/A

