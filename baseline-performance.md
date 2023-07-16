# Baseline Human Activity Recognition Classifier Performance

The purpose of the Capture-24 dataset is to serve as a training dataset for developing Human Activity Recognition (HAR) classifiers. 
These classifiers enable the identification of activities performed by individuals wearing smartwatch accelerometers during unmonitored free-living, such as in the UK Biobank physical activity monitoring study. 
Our goal is to ensure the use of the best possible model for accurate activity identification, with generalizable performance across different individuals and behaviors.

The [baseline-performance.ipynb](baseline-performance.ipynb) notebook outlines the implementation of a series of Python scripts used to determine and evaluate the performance of various machine learning (ML) pipelines in identifying activity levels in the Capture-24 dataset. 
The activity levels considered are simplified to the Walmsley labels, including sleep, sedentary behavior, light activity, and moderate-vigorous physical activity (MVPA).

The flow diagram below illustrates the pipeline for the HAR model, developed and trained to identify activities from raw accelerometer data.

![Flow diagram of baseline HAR model](baseline_model_pipeline.png)
*Flow diagram of the baseline HAR model*

The numbered stages of this analysis present various possibilities and considerations, which are explained below.

## Stage 1: Axis Invariance

The Axivity AX3 device, depicted below, has specific axes for recording positive values of acceleration in the x, y, and z directions.

<img src="wrist_accelerometer.jpg" alt="Axivity AX3 device with displayed axes" width="300" />

*Axivity AX3 device with displayed axes*

Although these axes may hold true for the majority of the dataset, we aim to make our models axis invariant to ensure their utility for other datasets. 
Several reasons justify this approach:

1. Participants may wear the device on different hands.
1. Participants may use different versions of the Axivity AX3 device, which could have different axes configurations.
1. Participants may wear a different brand of smart watch, with different axes configurations, such as Fitbit, Apple watch, etc.

To ensure that the generated model is not dependent on a specific axis configuration, two possible approaches were explored:

**Approach 1:** Continuing the analysis using only the acceleration magnitude, which is the Euclidean norm of the accelerometer signal. 
This approach discards any axis-specific properties.

**Approach 2:** Augmenting the training data by rotating/flipping it across all axes. 
Note that the evaluation metrics are computed only for the non-augmented data.

## Stage 2: Data Preparation

### Windowing

The full axis-invariant accelerometer signal is divided into 30-second windows with no overlap. 
The activity label for each window is determined based on the most frequent label observed within that time window.

### Feature Extraction

For deep learning models used in this analysis, the feature extraction is learned, eliminating the need for handcrafted features. 
However, for non-deep learning models, handcrafted signal features have been extracted from each data window. 
The list of these signal features can be found in [feature-list.md](feature-list.md).

### Data Preparation

The windowed accelerometer data may require additional steps for appropriate input preparation for the ML model. 
In the case of deep learning models, the windowed accelerometer data may be downsampled to match the expected input shape of the model. 
On the other hand, the extracted handcrafted features are standardized to have the same mean and standard deviation among the training population.

## Stage 3: Activity Classification

ML models are chosen and trained to classify activity labels based on the prepared accelerometer data. 
To ensure balanced training, sampling or weighting of loss functions is implemented, preventing the model from being biased solely by the frequency of certain activity labels. 
Note that for stages 3 and 4, a portion of the training data may need to be split into training and validation sets. 
This occurs when a model in either stage 3 or 4 requires a validation dataset. 
Group shuffle split is used to ensure no leakage of participant data.

The table below shows the models used in activity classification along with the percentage of the validation set used:

| Model                                     | Validation Set Used |
|-------------------------------------------|---------------------|
| Balanced random forest                    | 0%                  |
| Logistic regression                       | 0%                  |
| Support vector machines                   | 0%                  |
| XGBoost                                   | 0%                  |
| Signal CNN                                | 20%                 |
| Signal Resnet18                           | 20%                 |
| Signal pre-trained Resnet18               | 20%                 |
| Spectogram pre-trained Resnet101          | 20%                 |
| GramianAngularField pre-trained Resnet101 | 20%                 |

## Stage 4: Predicted Label Smoothing

The trained activity model assumes that windows of data from the same participant are independent and identically distributed (i.i.d.). 
However, this assumption does not hold in reality, as the likelihood of a given activity level depends on previous activity levels. 
To incorporate some temporal information in label prediction and introduce smoothing effects, we can employ a model to learn this dependency. 
Several approaches were considered:

| Model                         | Validation Set Used |
|-------------------------------|---------------------|
| No smoothing                  | 0%                  |
| Modal smoothing               | 0%                  |
| Linear regression             | 20%                 |
| Hidden Markov Model (HMM)     | 20%                 |
| Long Short-Term Memory (LSTM) | 20%                 |

## Stage 5: Model Evaluation

Each model is evaluated using 10-fold stratified group cross-validation. 
Groups are defined by participants to prevent data leakage between training and testing. Stratification ensures a representative distribution of labels across each fold.

Performance metrics are reported by comparing true labels against the predicted labels generated by the model pipeline, with participant data held out in the test set across the 10 folds. 
Performance metrics are reported as the mean and standard deviation per participant across the entire study, deviating from the convention of using per-window performance averaged across folds. 
This approach aims to assess the model's generalizability. 
Per-window performance metrics may hide poor performance in individuals who provide few samples of specific activity behavior.

The performance metrics reported in this work are:

1. Accuracy
1. Macro F1 score
1. Cohen's kappa score
1. Balanced accuracy
