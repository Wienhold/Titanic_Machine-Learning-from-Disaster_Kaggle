# Machine Learning Engineer for MS Azure - Capstone Project
# Titanic - Machine Learning from Disaster (Kaggle Competition)

After gaining some experience in working with Azure ML during my Udacity nanodegree I now want to finish the course with my capstone project.
As I always love a challenge, I chose to work on the Titanic Kaggle competition project. We want to generate a model which predicts with high accuracy if a specific individual is likely to survive the disaster or not. The goal is to get a response in for of a CSV which can directly be submitted to Kaggle.
This task should be achieved via two potential routes, using Azure's AutoML as well as Hyperdrive tuning algorithms.

## Dataset

### Overview
The dataset used in this project was downloaded from [Kaggle](https://www.kaggle.com/competitions/titanic/overview), where I also registered for the competition and later can submit my predictions. We are provided a [train](https://www.kaggle.com/competitions/titanic/data?select=train.csv) and a [test](https://www.kaggle.com/competitions/titanic/data?select=test.csv) dataset with 891 and 418 records, respectively.
We are given the following 12 columns, here some examplary data:
![image](https://user-images.githubusercontent.com/98894580/171413959-3d7347e6-3a4e-42f6-a359-697bc56b30ff.png)

As we can already see from the first few records, there are some missing values. Our first task will be to handle those.

### Feature Completion
Descriptive statistics for the combined train+test dataset were received from Minitab statistical software. 
![image](https://user-images.githubusercontent.com/98894580/171394293-547806cf-0809-4a83-b508-0528e1dd18bb.png)

Here we can see that we have three incomplete feature columns: 418 missing values for `Age`, two missing values for `Embarked` and 1 missing value for `Fare`.
The later should be less relevant for our predictions, so we will just fill `Embarked` with a random selection from the three options and `Fare` with the mean from the rest of the dataset (33.2955).
But the missing age values pose a relevant portion of all records in the dataset, so a more elaborate solution should be developed.
Using  Minitab's Predictive Analytics Module, a CART regression analysis was performed with Pclass, SibSp and Parch (ticket class, N of parents & spouses, N of parents & children) as categorical variables, aiming for least squared error and using a 10-fold cross validation.
As we see in the tree diagram, six terminal nodes were found.
![image](https://user-images.githubusercontent.com/98894580/171398053-62786b3e-5159-4df3-b9cb-a0cf9901a97a.png)
![image](https://user-images.githubusercontent.com/98894580/171398164-4ff00769-6c8a-45a1-9a24-ea1600c988d7.png)

It is obvious when we look at the residual plots that there is still a lot of deviation, which potentially could be decreased by further optimization. But for now, these values are sufficient to us and will be used for model development.
![image](https://user-images.githubusercontent.com/98894580/171398483-5b75645f-34fc-40f6-ae8e-cbe3c9f3b1b9.png)


### Task
From the given features for each record we want to make a prediction if this person did survive the Titanic disaster or not. Therefore relevant features should be identified to help us with this task.
Used features:
- Pclass: integer, ticket class (1, 2, 3), proxy for wealth
- Sex: string, male or female
- Age: float, age in years
- Sibsp: integer, number of siblings and/or spouses aboard (brother, sister, stepbrother, stepsister, husband, wife)
- Parch: integer, number of parents and/or children aboard (mother, father, daughter, son, stepdaughter, stepson)
- Fare: float, amoung of money paid for the ticket
- Embarked: string, port of embarkation	(C = Cherbourg, Q = Queenstown, S = Southampton)

Discarded features:
- PassengerId: integer, internal key
- Name: string
- Ticket: string, ticket numbers as various formats
- Cabin: string, cabin number, many missing values

Target:
- Survived: binary, ground truth

So we will use 7 features for training and prediction.

### Access
The training dataset as well as the testing dataset are registered within MS Azure. Furthermore, respective filtered versions with only the relevant features specified above as well as with the target "Survived" for the training dataset were registered as well. As the Hyperdrive run itself does not include automatic feature encoding like AutoML, one-hot encoding using pandas' ```get_dummies()``` method was applied in addition.
![image](https://user-images.githubusercontent.com/98894580/172067898-068f671e-47d6-44a5-9fe7-2107d48fb222.png)


## Automated ML
For the AutoML experiment we choose the following settings. They include the options for deep learning and for early stopping to save computation time for less promising approaches, as well as the 80%-20% train-validation split and the definition of accuracy as the primary metric. We also want to find a single promising model, so we disable ensembles.
```
automl_settings = {
  "max_concurrent_iterations": 5,
  "max_cores_per_iteration": -1,
  "enable_dnn": True,
  "enable_early_stopping": True,
  "validation_size": 0.2,
  "primary_metric" : 'accuracy',
  "enable_voting_ensemble": False,
  "enable_stack_ensemble": False
  }

automl_config = AutoMLConfig(
  compute_target=compute_target,
  task = "classification",
  training_data=dataset_filtered,
  label_column_name="Survived",
  path = project_folder,
  featurization= 'auto',
  debug_log = "automl_errors.log",
  **automl_settings
  )
```

### Results
After submission the experiment took about 20 min to complete. A total of 50 models were tested, with the best sucessful run reaching an accuracy of 82.1%. Some models returned failures and therefore weren't considered.
![image](https://user-images.githubusercontent.com/98894580/172328429-93e8f12f-00ca-4a60-88a5-b36b93fabfc2.png)
![image](https://user-images.githubusercontent.com/98894580/172328996-56cf7239-3812-44d7-b47c-56e35d69a86d.png)

![image](https://user-images.githubusercontent.com/98894580/172327316-a2cc1ff5-2dcc-4ea0-a09b-f576ead9d40b.png)
![image](https://user-images.githubusercontent.com/98894580/172327418-63478dff-a185-45c8-80b2-4c94800b86ee.png)


The resulting best model was an XGBoost classifier with data preparation via a sparse normalizer.
![image](https://user-images.githubusercontent.com/98894580/172328700-a754540b-65a4-45d0-94c4-8c12d88dac0f.png)
![image](https://user-images.githubusercontent.com/98894580/172329525-6ebe4f47-6b89-4347-a470-ce7a7feea469.png)

A view into the explanations tells us about the impact of each feature on the chance of survival. Interestingly, sex and wealth (Pclass) are by far the most important factors to survive the disaster.
![image](https://user-images.githubusercontent.com/98894580/172329284-62ffd662-1f19-4def-b200-1e9402769838.png)

## Model Deployment

The model was then deployed as an endpoint. Therefore, an ```InferenceConfig``` script as well as a ```score.py``` script for handling submitted data were generated.
The data can be submitted via a post request to the scoring uri, the composition of the request can be seen here:
![image](https://user-images.githubusercontent.com/98894580/172406512-30986e3d-d486-459d-815c-967411c2e7a7.png)
The response is a simple list of binary values, in this case ```[1,0]```, indicating that the first individual is predicted to survive while the second individual won't.
For the ease of handling, an additional function to predict directly from pandas dataframes was included.

![image](https://user-images.githubusercontent.com/98894580/172068345-48ea8514-16ed-442a-9c61-5fcf9f947a49.png)

The 418 records from the testing dataset were submitted to the predictor, the result was again registered as a new dataset together with the respective indices from Kaggle. The results were also saved as a CSV file in the format required for Kaggle submission. The file was downloaded and submitted to Kaggle, scoring 0.73444 on the leaderboard.
![image](https://user-images.githubusercontent.com/98894580/172068628-1901fe07-3644-4e6c-bfb2-48cbf90657c6.png)

As this has been my first Kaggle competition I was quite happy with the result, but of course there is always some room for improvement.
Especially promising should be the addition of further feature processing and engineering steps. For this purpose, non-linear transformations could be employed for breaking the correlation between different parameters. Some features like age could be transformed from continuous variables to categorical ones (e.g. "young", "adult", "elder"). Further information could be gathered on the individual status by taking into account titles and ranks separated from the names as well as using the family information to better approach the individual fare.
On the machine learning side, of course extended training sessions as well as the utilization of ensembles could further improve results.

## Hyperparameter Tuning
After successfully employing AutoML for survivor prediction, I wanted to give a custom model with optimized hyperparameters a try. Therefore, a Hyperdrive setup was generated including a ```train.py``` script for the actual model generation and training. I decided for SKLearn's ```GradientBoostingRegressor``` as a base model and used Hyperdrive to use various values for maximum depth and learning rate by ```RandomParameterSampling```. For saving GPU time, a ```BanditPolicy``` was employed for early stopping and the training was limited to 40 runs.
The parameters selected for optimization are ```learning_rate``` and ```max_depth```, they should be selected via ```choice``` from a provided list. ```learning_rate``` as a generally important hyperparameter for all ML experiments should be varied to optimally reach the globally best metric and to avoid the pitfalls of getting stuck in local error minima at a too low as well as to overshoot at a too high learning rate. ```max_depth``` was selected due to its high importance in reaching good differentiation specifically for the gradient regression method. The maximum depth limits the number of nodes in the tree and should therefore be tuned for best performance.
Of course various further parameters could be tuned like for example ```n_estimators```, ```max_leaf_nodes``` or various options for ```max_features```, but these were not considered in this first try to keep computation time low.

### Results
The training took about 42 min and resulted in the best model giving an accuracy of only 44.4%.
Probably ```GradientBoostingRegressor``` was not the best choice for solving this challenge. Furthermore, more than two hyperparameters could have been passed for optimization. Besides or even subsequent to the ```RandomParameterSampling``` run, further methods like ```BayesianParameterSampling``` or ```GridParameterSampling``` could improve results.
![image](https://user-images.githubusercontent.com/98894580/172069157-cf33a4b4-f24f-4b81-9b1e-014828b63a76.png)

![image](https://user-images.githubusercontent.com/98894580/172069286-233ec926-52fe-473a-881d-d2ccae8a3afe.png)

Here we can see a screenshot of the RunDetails widget showing the progress of the training runs of the different experiments, including the top performing model.
![image](https://user-images.githubusercontent.com/98894580/172343095-f8783ac4-4c10-4415-851b-e6538665ce1a.png)

![image](https://user-images.githubusercontent.com/98894580/172069193-b3e9aeae-21b7-44ab-8532-e0d0222a26cd.png)

As this result was not very promising, deploying an endpoint for submission of the testing data was skipped for this model.

More promising hyperparameter tuning approaches could start from the XGBoost model resulting from the AutoML run instead of the ```GradientBoostingRegressor``` and involve all tunable hyperparameters. As these would result in a large searchspace, they should potentially be analyzed via ```RandomParameterSampling```, potentially followed by further refinement via a ```BayesianParameterSampling``` strategy.

## Screen Recording
[Presentation on YouTube](https://www.youtube.com/watch?v=eQK2soYqhtU)
