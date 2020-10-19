# Overview

This is the deployment code for the Cashflow project. Cashflow project is historically divided in 2 parts:

1. Cashflow I (CF1): CF1 is detailed here [wiki](https://wiki.swedbank.net/display/CINT/Cash+Flow+prediction). It has been deployed in June 2019, before AMMF establishement.
2. Cashflow II (CF2): In CF2 the objective is to forecast the balance time series 3 months ahead of time, for 2.5 M customers of scope. The selected method consists in a deep learning 1-dimensional convolutional neural network (1D CNN), as it is multivariate and rather robust against sharp variations in the time series. CF2 is detailed in the [wiki](https://wiki.swedbank.net/display/CINT/Cashflow+2nd+part%3A+Description+of+the+deep+learning+solution).

By Feb 2020, the current goal is to deploy CF2 according to AMMF requirements on synthetic data. When this is done, then CF2 will be tested on real data (time series generated within CF1, available in DP layer in ODL). When this is working, then CF1 will be connected to CF2, as a feature engineering function that will be called in the training.py file. (May 2020)

Here, we briefly summarize the main sections in the script. For details about definition of the CF2 model, labeling and main source tables, please refer to the project's [wiki](https://wiki.swedbank.net/display/CINT/Cashflow+2nd+part%3A+Description+of+the+deep+learning+solution).  
 
## Dataset Description

Within CF2 framework, the input data are the cross-account balance-like time series of all customers of scope. For training or scoring is an hdfs file containing 3 columns:

- primaryaccountholder: the customer id
- transactiondate: a sorted array of at least 365 day dates of the balance time series
- balance: the corresponding values of the time series

## Training 
The [training.py](model_modules/training.py) produces the following artifacts:

- trained model file: saved_model.pb, saved in HDFS format. 
- file for evaluation

## Evaluation
Evaluation is also performed in [scoring.evaluate](model_modules/scoring.py) and it returns the following metrics:

- R2 at 3 months horizon, and R2 at 1 month horizon, in the file evaluation.json.

## Scoring
The [scoring.py](model_modules/scoring.py) loads the model and metadata and accepts the unseen dataframe for prediction. 

## TODO

The general task list is described in [wiki](https://wiki.swedbank.net/display/CINT/Cashflow+2nd+part%3A+Description+of+the+deep+learning+solution). Here list of code-wise tasks.

**A. Cashflow II tasks:**
- BUG [solved]: json file not readable from file in AIlab when using nested json objects.
- [done]: modify the data.json and test the scoring.py code.
- BUG [partially solved]: logging not working. The thing is that logging in distributed system is tricky. AMMF team does not have a best practice now, so shifting back to simple print (logging would save a file to docker, which would not be accessible to us, while simple print out will print to spark log, which is available)
- [done]: put the functions of scoring.py (not evaluate and score) into utils.py and test that scoring.py is still working. There will be a bug with pyarrow/pysparkUDF...
- TODO [tried without success]: optimize step 1 of CF2: so far a few hours needed, but could be improved (tweak UDF or pandas_UDF)
- BUG [partially solved]: when running scoring.py, separately all parts complete, but when ran together, it fails... Likely it needs to give back memory between each part. Try spark.catalog.clearCache() ? Solution found: it was due to error not finding tf model saved (conflict between different cores). Hack is to fix one core per executor...
- [done] test full code (without commented steps), both training.py and serving.py
- [done] automate the key of tables (so far synthetic data hard-coded)
- [done] harmonize the AIlab code and Cloud code inside training.py and scoring.py. Should we create 1 version for both platforms, or 2 separate versions? It was decided to separate the codes, one for each platform.
- [done] correct the path of the parquet files (json index hard-coded so far in the training and scoring codes)
- [done] correct and test the evaluation function!  
- [done] need to copy the model from the current model id (in training.py) and puts the copy into a current type of sub-folder, with an invariant name, so that the scoring will query that folder instead of the id one...
- [TODO]: add hyperparameters tuning!  
  - [done] learn how to use experiments
  - [done] save training data to feature store, then to "training dataset" folder (using feature store) as a tfrecord format, and load that in tensorflow
  - [done] do same for validation dataset
  - [done] retrain the model using best hyperparameters and save the model to hdfs
  - [TODO] load the model in the scoring.py in the pandas_udf and check that the scoring is done properly
  - [TODO] refactor the code to have a "final" version with HP tuning
- BUG [partially solved]: in the training code, when increasing the number of epochs, Petastorm library returns an error. Look into this.
- [TODO] test the full code on different time ranges
- [TODO] add the is_customer_regular metric!!!
- [optional] add unit tests... big task! 

**B. Cashflow I tasks:**
- TODO: bring the 4 tables to AILAB (so far 3 available)
- TODO: incorporate the steps of CF1 into the training.py of CF2
- IMPEDIMENT: are we allowed to use data in AILAB for non-AML case like Cashflow? This needs to be solved
- TODO: build small datasets (i.e. subsets of the 4 tables) for development and testing purpose.

