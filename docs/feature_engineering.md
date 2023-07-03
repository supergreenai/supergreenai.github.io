# Feature Engineering

Error Analysis requires features that may be very different from the features used by the ML model being analyzed. These features are used for high level and fine-grained error segmentation, but not necessarily for prediction.

## Business Segments

Business segmentation plays a vital role in two key areas. Firstly, it allows data science to communicate in business terms, the business areas that need investment, helping decision-makers sponsor error analysis activity. Secondly, it helps explain some of the variety of the data, which often leads to challenges. To address these aspects, invest time in creating categorical columns or meta-features that describe relevant segments and sub-segments.  \




* Industries: Different data sources and mix of data exist across industries.
* Geography: Varied regulations and network latencies impact data.
* Clients/Channels: Data integration, quality, and legal contracts vary.
* Tier: Incentives, risks, and data volumes differ across tiers.
* Offering: Individual product/service groups have unique data needs.

## Demographics and Bias

Demographic data, such as age, gender, income, or race, is often excluded from prediction models' training sets to prevent bias. In many cases this is regulatory mandated. However, in certain cases, the model does pick up bias, for example by using zip code information which is correlated to certain neighborhoods which could be correlated to a specific demographic. If the banned demographic data (such as race) is used for error analysis purposes only, it can help identify discrimination. This manifests as specific demographics with higher model prediction errors.

## Data Origin

Prediction errors in the ML model can be caused by changes in data sources or the data generation process, which in turn affect the features used by the model. This non-stationarity makes it more difficult to accurately predict the target variable, resulting in prediction errors. Therefore, features that describe the origin of the data can help explain errors:



* External data source (clients, channels, partners, data providers)
* Data processing subsystem name that had a failure or timeout.

## Label Origin

Labeling mistakes not only adds noise to the training set, but also adds noise to the test-set error metrics. Labeling mistakes can be identified if test-set prediction errors correlate with the origin of the labels. Treating labeling information as an explanatory variable rather than a target variable, removes many limitations. Such features can be:


* A detailed label that was originally simplified for use as a target variable.
* Label source such as the name of the labeler or the labeling function.

For example, the label for the malware prediction model is a boolean flag that indicates if the machine was infected or not. While the explanatory variables used for error analysis could flag the types of the infecting malwares (Ransomware, Spyware, Adware, etc…).


## Retrospective information

Data that becomes apparent too late to be used in production and was not used as a labeling function can still be leveraged for error analysis. In most cases this data point is too noisy or unreliable.

For example, if the machine "disappeared" (probably formatted) it might indicate that it was previously infected by malware, but it could also mean a laptop was repurposed by IT department, or any number of things. This indication is just too noisy to be used as a malware label indicator. Nevertheless, if it correlates well with false positive prediction errors, it can explain the disparity as a malware just destroyed some laptops it was infecting.

## Confounding Training Features

It is important to consider model features for explaining model errors. However, including all features used in the original model during error analysis produces too many results, which are mostly duplicates.

Let's consider a scenario where a malware prediction ML model model struggles with tablets but performs well on laptops. Features that are derived from the device form factor, like hard drive capacity and display size, may not be helpful.  A result such as `"form_factor='tablet'"` is much more informative than `"drive_capacity_gb <= 128 AND display_inches <= 12". `The device form_factor is a confounding variable that needs more attention.

In order to include model features in the error analysis use a two step approach:



1. Perform a high-level error analysis using only confounding features (aka form_factor)
2. Once such a model feature is found to explain an error concentration, use it to partition the analysis. For example, inside the `"form_factor='tablet'"`partition, drive_capacity and display_size are no longer explaining errors. So there would be no harm including them in the error analysis.

## (Non)Features still in Research

Certain ML model features may not be ready for production due to various reasons. However, if these features explain prediction errors, then it would be a strong motivator for prioritizing their deployment.

## (Non)Features that introduce Data Leakage

Features that provide access to future information or the target label might exhibit high predictive power during research but may perform poorly in production. However, they can be valuable for error analysis.  
 
For example, consider a feature that averages the probability for malware infection in the current week. This feature contains forward looking information (up to 1 week ahead), and does not really exist in production (where only information on the past exists). However, a large increase in this figure can indicate a fundamental shift in malware attacks and thus correlate with prediction errors.

## (Non)Features that may cause Overfit

Raw high cardinality data, such as IP addresses or user IDs, can cause the model to overfit on the train set data. Even when applied to a transformation such as target encoding (or reputation) the result could overfit on a specific incident that tarnished the reputation, or have difficulties in production given new IDs without prior reputation. However, these features can be useful for error analysis, aligning with the Pareto principle where a small portion of feature values often contributes to a significant portion of errors.
 
For example, let's try to predict malware infections by evaluating the organization's maturity based on its size and industry. Now imagine a model is running in production and a new big organization from a well regulated industry has been onboarded. However, this org has no employee security awareness training, and does not deploy regular OS patch updates. And as a result their machines are infected more than its peers. As a result the prediction model is too optimistic resulting in false negatives errors for that organization. In this case, the organization ID correlates well with prediction errors and is a great explaining variable for error analysis. 


## (Non)Features due to selection and Correlation

In general any feature that has [predictive power](https://github.com/8080labs/ppscore#) over the label could be a candidate for error analysis. Some features may end up having very low or zero importance, or filtered out during feature selection. If this happens although the feature has good predictive power, it means the feature was rejected due to correlation with other features. 


To illustrate this, let's consider the prediction of malware infections based on two features related to security patches:

* Number of missing security patches
* Time since last security update installed

Obviously, these two features are highly correlated as a security patch installation on a machine would reset both features to zero. Based on the available training data, the prediction accuracy remains the same with or without the "time since update" feature, and the model is trained and deployed to production without that feature. Time goes by and Windows 8 reaches end-of-support which means no further security patches. Consequently, even if there are zero missing security patches, the system becomes vulnerable to malware infections. The production model starts exhibiting an increase in false negative errors, as malware infects machines with zero missing security patches.

During the process of error analysis, we use the "time since update" as an explanatory variable, despite it not being included as a feature in the model. We observe correlation between machines that have not installed a security update recently and model prediction false negative errors. The underlying reason is that machines running a non-supported Windows version, have zero missing security patches, even though they have not been updated for a long time. As a result, we decide for the next model training to add a new boolean feature that indicates if the OS has reached end-of-support or not.

