# MLOps Explained

This document was created to help me understand MLOps and all of the steps involved in productionalizing Machine Learning. I am utilizing Databricks and the functionality provided by that platform.

## Table of Contents

- [Data Science Modeling and Exploration](#data-science-modeling-and-exploration)
- [Data Science Workflow Steps (Dev vs Prod)](#data-science-workflow-steps-dev-vs-prod)
- [DevOps vs MLOps](#devops-vs-mlops)
- [New Dynamics](#new-dynamics)
  - [Code](#code)
  - [Model, Parameters, Evaluation Metric Results](#model-parameters-evaluation-metric-results)
  - [Training/Test Data](#trainingtest-data)
    - [Features](#features)
    - [Training](#training)
    - [Evaluation](#evaluation)
- [How To Address These New Dynamics](#how-to-address-these-new-dynamics)
  - [Code](#code-1)
  - [Model, Parameters, Evaluation Metric Results](#model-parameters-evaluation-metric-results-1)
  - [Training/Test Data](#trainingtest-data-1)
- [MLFlow](#mlflow)
   - [MLFlow for Development](#mlflow-for-development)
   - [MLFlow for Production](#mlflow-for-production)
- [Deployment](#deployment)
- [Post-Deployment Observability](#post-deployment-observability)
- [Retraining a Model](#retraining-a-model)
- [Summary](#summary)

## Data Science Modeling and Exploration

This is ideally the only work that a Data Scientist should do. A lot of exploration and experimentation needs to be done to figure out which ML Model should be used for the problem, what parameters value should be supplied, and which features are most useful to help the model make a prediction. This process is iterative.  

## Data Science Workflow Steps (Dev vs Prod)

 1. Preprocess Data - Transform data into a usable format for ML Model  
    a. Dev: This includes feature creation and data cleaning. Multiple iterations and experimentation.  
    b. Prod: Utilize an official feature table and apply any adjustments. (Feature table is loaded on a schedule by Data Engineering)
 2. Create Training Dataset (and Test Dataset if applicable)  
    a. Dev: Test multiple different splits on dataset for optimal split  
    b. Prod: Intakes established configuration for split and applies
 3. Train the ML Model on Training Dataset
 4. Evaluate the ML Model Predictions (if applicable)  
    a. Dev: Iterate through multiple parameter testing to compare evaluation results   
    b. Prod: Run evaluation and have deployment check
     - First Deploy: rmse < BUSINESS_THRESHOLD, fail
     - Retrain: rmse < CURRENT_PROD_MODEL_RMSE, fail
 5. Serve the Model  
    a. Dev: Often PoC to demonstrate  
    b. Prod: Register the model to a Model Registry
      - A separate serving service can then pull the latest model to utilize for predictions

## DevOps vs MLOps

Continuous Deployment needs to trigger all of the Data Science Production Workflow steps. We are not just deploying code, we are deploying a Machine Learning Model.

 1. Primary deployment is the training, evaluation, and deployment of a model  
    a. This is MLOps
 2. Secondary deployment is any code changes to the repo that do not affect or need a primary deployment (such as adding a utility script)  
    a. This is classic DevOps

Both need to be able to occur.

## New Dynamics

To understand why MLOps has been introduced, we have to understand the new dynamics that Machine Learning Models bring into Development Operations.

### Code

Machine Learning code that is used to to create, train, evaluate, and deploy Models should all be stored within source control and follow best practices in regards to code organization/structure and have tests written to help ensure correct functionality. The project should utilize CI/CD to ensure reliable changes can be made.

### Model, Parameters, Evaluation Metric Results

A new aspect introduced to code reliability is Machine Learning Models. We need to store the Model, the parameters that were used to create the model, and the evaluation metric results of the model. These are the inputs/outputs that can be manipulated to introduce better predictions. Storing these allows us to understand if a new model is expected to perform better.

### Training/Test Data

Another new aspect introduced with Machine Learning Models is its reliance on training and test data. If the training data is different across model training, the prediction and evaluation results are highly likely to be impacted. This means we need to store and link training/test datasets to the model (and its inputs/outputs) to be able to understand model differences. This also helps for debugging.

----
To effectively integrate Machine Learning Models into Development Operations, we have to address the need to store Models, the training/test data utilized during training, and the parameters and evaluation metric results associated. We have to treat these new aspects of developoment the same way we would treat code to reliably deploy, debug, and rollback.

## How To Address These New Dynamics

### Code

This is classic DevOps. We need to ensure code has a proper CI/CD pipeline with scalable organization and written tests for reliability. This means NO NOTEBOOKS. At least for production code. Notebook away for experimentation Data Scientists. Databricks-Connect within an IDE such as VS empowers local project development and testing of code before merging.

### Model, Parameters, Evaluation Metric Results

We need to utilize a model tracking tool, such as MLFlow. MLFlow has a lot of functionality to support version control of Models. We'll talk about MLFlow in the next section.

### Training/Test Data

To ensure reproducibility, we need to store each training/test/test_predictions dataset separately for each training session. WE CANNOT OVERWRITE THESE TABLES. To make this scalable, we'll implement some checks and have two separate schemas: features and training.

#### Features

This schema holds the official and robust feature tables that can be utilized or augmented to create training/test datasets. These feature tables should have constraint enforcement, be of high quality, and be updated by Data Engineering on a schedule.

#### Training

This schema holds each training/test/test_predictions dataset that is created for production deployments. When a model is trained for deployment, it will create its training/test dataset. Both sets will have a hash created to represent the data contents and to ensure we don't create duplicate tables. This tablename/hash will be associated with the model and the model parameters.   

This allows us to store the training/test datasets that were used to train the model before its deployment, giving us the ability to investigate/debug or retrain the model on the same data with different parameters if needed. We can then implement a cleanup job to drop old training/test datasets that are no longer needed.  

#### Evaluation
We also save the test predictions as a dataset during the evaluation stage. We associate this with the model_id and model_version obtained from MLFlow. This allows us to log the model predictions in production and compare distribution against our test predictions to detect any drift to signal for possible re-training. Retrain = new model version and new predictions table.

## MLFlow

### MLFlow for Development

MLFlow is an open source tool that empowers full Machine Learning development by storing key information and empowers model logging with this format:
 - Experiment (ML problem you're solving, basic structure of code)
    - Run (Different parameters, variations of the same code)
        - Associate:
            - Artifacts (nice to haves, like graphs)
            - Datasets
            - Model
            - Metrics (evaluation metrics)
            - Parameters
            - Tags

MLFlow tracking is the ability to track/log the model runs that a Data Scientist trains. This allows for effective development by easily comparing different model runs and to easily see which features are important and which parameters provide better results.

### MLFlow for Production

MLFlow can tie important information to a model, such as a Git Commit Sha, Test/Training/Test_Predications datasets, and metric evaluation results. This is critical for debugging and for pipeline automation. MLFlow can be utilized to:
 1. Associate Training/Test/Test_Predictions with a specific model and version.  
 2. Register a model to the MLFlow Model Registry to easily be pulled by a production service.   
 3. Grab accuracy metrics for the production model to then compare against any retraining. This allows us to determine if a retrained model is more accurate and should be deployed to the registry. 
 4. Log and compare the current production Model's predictions against it's associated test predictions (by MLFlow's association logic) to detect drift.  
 5. Have specific production model performance metrics logged for monitoring

## Deployment
To deploy a Machine Learning Model in an idempotent (~99.9%) way, we need these components:
 - Model Type
 - Seed Value
 - Parameter Values
 - Feature Table and/or preprocessing logic

Once these values are provided to a central code repository and merged, these steps occur:  
 1. Regular CI/CD  
    a. Tests Run  
    b. Deploy code to production environment
 2. Model Deployment Process (Ran from the central repo code as a job)   
    a. Create the Model  
    b. Create and save the training/test dataset utilizing preprocessing logic    
    c. Train the model  
    d. Evaluate the model and save the predictions dataset!  
    e. Regiser the model to the model registry  
    f. Trigger the serving layer service to pull the latest model

> MLFlow has a model registry, similar to a container registry, that makes it easy to deploy models. Once a model has been registered within the registry, we can simply alias/tag it as the current production model. 

A separate service utilizes MLFlow client code to pull the latest model for the Experiment and use that for its predictions. This decouples the model deployment and the serving layer. This allows for easy rollback by simply aliasing/tagging a different model in the registry as current and refreshing the serving service to use that model instead.

## Post-Deployment Observability

Most will stop once a model is deployed. This is naive. A model will most likely start to drift in accuracy and need to be retrained or modified in the future. There are three avenues we can take here:   
 1. We can re-train our model on new production data at a scheduled cadence, say weekly. This is proactive approach when the system is mature and stable.  
 2. We log the results of the production model and create alerting for drift. We can then get alerted or trigger an automatic retrain & deploy.
 3. A hybrid approach of both

I recommend staying away from automatic re-training until the process is mature. Predictions should definitely be logged and distributions checked against the test_predictions data to detect for drift though. This allows us to be proactive and be alerted of any issues.

## Retraining a Model

When retraining a model, we utilize the same model type and parameters, but we now have new data to train on. We can implement a check in the evaluation stage to compare accuracy metrics from the production model that was deployed to the new models. If the new model isn't more accurate, we simply don't deploy.

## Summary

To reliably productionalize Machine Learning Models, a central repository needs created. This repository has the model type, seed, parameters, and preprocessing logic stored in configuration to reliably create an ML Model and serve it to production systems. This ML Pipeline can be utilized for model deployment to an MLFlow registry, detect production model drift, and retraining of specific models when necessary.  

All Data Science Development is mostly done outside of the repo until proper configuration is determined, then they simply need to adjust configuration in the ML Pipeline repo and merge their code to trigger a new model deployment.