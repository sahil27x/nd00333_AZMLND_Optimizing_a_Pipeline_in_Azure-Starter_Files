# Optimizing an ML Pipeline in Azure

## Overview
This project is part of the Udacity Azure ML Nanodegree.
In this project, we build and optimize an Azure ML pipeline using the Python SDK and a provided Scikit-learn model.
This model is then compared to an Azure AutoML run.

## Summary
This dataset contains data about individuals applying for bank loans. The task we set out to accomplish here is to develop a model that,
based on the information provided about each individual, predicts whether they will subscribe to a service.

The best performing model was found to be a voting ensemble with 91.8% accuracy. However, many models were of comparable accuracy.

## Scikit-learn Pipeline
The Scikit-learn pipeline obtains the provided data from the provided URL. Following data download, a number of data cleaning steps are carried out including:
- Removing NAs from the dataset.
- One-hot encoding job titles, contact, and education variables.
- Encoding a number of other categorical variables.
- Encoding months of the year.
- Encoding the target variable.

Once the data has been prepared it is split into a training and test set. A test set size of 33% of total entries was selected as a compromise between ensuring adequate representation in the test data and providing sufficient data for model training. 

The classification method used here is logistic regression. Logistic regression uses a fitted logistic function and a threshold. The parameters available within the training script are C (which indicates the regularization strength i.e. preference for sparser models) and maximum number of iterations.

Azure's Hyperdrive service was used for hyperparameter tuning with the following key elements:

#### Parameter sampling
I decided to use random parameter sampling. Compared to the other techniques available, random sampling does not require pre-specified values (like grid search) and can make full use of all available nodes (unlike bayesian parameter sampling). 

Random Parameter sampling supports early termination of low-performance runs. Benefit- A simple random sample allows to learn about an entire population much faster and more efficiently than collecting data from every member of the population. Some users do an initial search with random sampling and then refine the search space to improve results. Random parameter searching is preferable to gridsearch also in that it is highly unlikely that the specified values in gridsearch are optimal, while there is a chance that the ones obtained randomly are closer to ideal values. Combined with the fact that I have very little background in using logistic regression random parameter sampling enables a much more broad search over the parameter space and with it's capabilities to make use of concurrency would outperform bayesian sampling for large jobs.

#### Early stopping policy
I selected the BanditPolicy stopping policy because it allows one to select a cut-off at which models reporting metrics worse than the current best model are terminated. This allows a relatively intuitive method to screen models, only retaining those with similar or better performance. This policy offers a little more flexibility than truncation and median stopping.

BanditPolicy defines an early termination policy based on slack criteria, and a frequency and delay interval for evaluation. Slack factor - The amount of slack allowed with respect to the best performing training run. This factor specifies the slack as a ratio. Slack Amount - The amount of slack allowed with respect to the best performing training run. This factor specifies the slack as an absolute amount.

## AutoML
I defined the following configuration for the AutoML run:

```
automl_config = AutoMLConfig(
    experiment_timeout_minutes = 30,
    task = 'classification',
    primary_metric = 'accuracy',
    training_data = ds,
    label_column_name = 'y',
    n_cross_validations = 7,
    compute_target = compute_cluster,
    iterations = 5)
```
_experiment_timeout_minutes = 30_

This is an exit criterion and is used to define how long, in minutes, the experiment should continue to run. To help avoid experiment time out failures, I used the minimum of 30 minutes.

_task = 'classification'_

This defines the experiment type which in this case is classification.

_primary_metric = 'accuracy'_

The metric that Automated Machine Learning will optimize for model selection. we have used accuracy as the primary metrics.

_training_data = ds_

The training data to be used within the experiment. It should contain both training features and a label column.

_label_column_name = 'y'_

The name of the label column.

_n_cross_validations = 7_

How many cross validations to perform when user validation data is not specified.

_compute_target = compute_cluster_

The Azure Machine Learning compute target to run the Automated Machine Learning experiment on.

_iterations = 5_

The total number of different algorithm and parameter combinations to test during an automated ML experiment.


## Pipeline comparison
**Comparison of the two models and their performance. Differences in accuracy & architecture - comments**


| HyperDrive Model | |
Best Run Id:  HD_da1813a9-c46b-4696-8976-b9755a019132_7
Accuracy: 0.9074


| AutoML Model | |
| id | AutoML_7e62a313-77dd-4e54-be40-a453d45c60ce |
| Accuracy | 0.9163 |

The two models performed very similarly in terms of accuracy, with the hyperdive model achieving 90.7% accuracy and the autoML model achieving 91.6% accuracy. The difference in accuracy could come down to slight variations in the cross-validation process. The pipelines make use of the same data cleansing process, however autoML tests a number of scalers in combination with models, adding a preprocessing step prior to model training. Architecturally, the models are quite different. Logistic regression (90.7% accurate; tuned with hyperdrive) effectively makes use of a fitted logistic function with a threshold to carry out binary classification. The voting ensemble classifier (91.6% accurate; selected via autoML) makes use of a number of individual classifiers and, in this case, averages the class probabilities of each classifier to make a prediction. 


The autoML pipeline is very similar to the Scikit-learn pipeline described above with several notable differences:
- The data are retrieved from the provided URL.
- The data are cleaned using the same process as described above. 
- The data are **not** split into train and test sets.
- The variables and target dataframes are merged prior to the autoML process.
- The joined dataset is used as input in the autoML configuration and the autoML run is processed locally.

The best model selected by autoML was a voting ensemble (~91.6% accurate). The Voting Ensemble model selected used a slight amount of l1 regularization, meaning that some penalty was placed the number of non-zero model coefficients. Additionally, the voting method was soft voting (as compared to hard), where all models' class probabilities are averaged and the highest probablility selected to make a prediction. Although the learning rate scheduling for gradient descent is specified as 'invscaling' (i.e. inverse scaling), the scaling factor power is 0 indicating that the learning rate is constant in this case.

## Future work
In the future it might be helpful to explore more feature engineering steps prior to training. Also, many of the AutoML runs use a scaler prior to model training and evaluation. Our options are limited for AutoML as it does all the selection and the fine tuning work for us. We can probably collect more data and vary the cross_validations and change the experiment_timeout so that AutoML tries out several models and preprocessing steps to come up with the model having the best performance. The encoded data does not really benefit from this scaling, so selectively scaling continuous variables instead of all, might be helpful. Also, running AutoML for much longer would likely find better models in this case. Furthermore, exploring hyperdrive with a broader variety of classification models would also be informative. Moreover, if the data set is imbalanced then Accuracy is not always the right metric to be considered. There are some other metrics which you may also consider:
AUC
Precision
Recall

