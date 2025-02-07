# Mango: A parallel black-box optimization library

**Mango** is a python library for parallel optimization over complex search spaces. Currently Mango is intended to find the optimal hyperparameters for machine learning algorithms.
Mango internally uses parallel implementation of multi-armed bandit bayesian optimizer based on gaussian process. Some of the salient features of Mango are:
- Ability to easily define complex search spaces which are compatible with the scikit-learn random search and gridsearch functions.
- Internally uses state of art optimizer which allows to sample a batch of values in parallel for evaluation.
- Objective function can be arbitrary complex which can be scheduled on local, cluster  or cloud infrastructure.
- The ease of usage was kept in mind with ability to plugin new distributions for search space and new optimizer algorithms.

## Index
1. [ Mango Setup ](#setup)
2. [ Simple Example ](#simpleexample)
3. [ Tune Hyperparameters of KNeighborsClassifier ](#knnexample)
4. [ Domain Space of Variables](#DomainSpace)
5. [ More on Objective Function](#ObjectiveFunction)
6. [ Controlling Mango Configurations](#MangoConfigurations)
7. [ Schedule Objective Function on Celery](#Celery)
8. [ Tune Hyperparameters of Facebook Prophet ](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/Prophet_Classifier.ipynb)
9. [ Tune Hyperparameters of xgboost XGBRegressor ](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/Xgboost_Example.ipynb)
10. [ Tune Hyperparameters of xgboost XGBClassifier ](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/Xgboost_XGBClassifier.ipynb)
11. [Tune Hyperparameters of SVM](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/SVM_Example.ipynb)
12. [ More Examples](https://gitlab.com/arm-research/isr/mango/tree/master/mango/examples)
13. [ Contact & Questions ](#contactDetails)

<a name="setup"></a>
# 1. Mango Setup
```
Clone the Mango repository, and from the Mango directory.
$ pip3 install -r requirements.txt
$ python3 setup.py install
```

<!--
- Mango requires scikit-learn and is develped for python 3, some other packages are installed which required to optimize xgboost classifiers and fbprophet.
!-->

Testing the installation.
```
$ cd PATH/mango/mango/test
$ pytest
```


<a name="simpleexample"></a>
# 2. Simple Example
Mango is very easy to use. The given example finds optimal value of the identity function whose input is a single variable between 1 and 1000.
More examples are available in the directory *mango/examples*.

```python
from mango.tuner import Tuner

param_dict = {"a": range(1,1000)} # Search space of variables
             
def objectiveFunction(args_list): # Identity Objective Function
    a = args_list[0]['a']
    return [a]

tuner_identity = Tuner(param_dict, objectiveFunction) # Initialize Tuner

results = tuner_identity.maximize() # Run Tuner
print('Optimal value of a:',results['best_params'],' and objective:',results['best_objective'])
```

Sample output of above example.

```
Optimal value of a: {'a': 999}  and objective: 999
```
More details about this simple example are available [here.](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/Getting_Started.ipynb)

<a name="knnexample"></a>
# 3. Mango Example to Tune Hyperparameters of KNeighborsClassifier

```python
from mango.tuner import Tuner

from scipy.stats import uniform

# n_neighbors can vary between 1 and 50, with different choices of algorithm
param_dict = {"n_neighbors": range(1, 50),
              'algorithm' : ['auto', 'ball_tree', 'kd_tree', 'brute']
             }
             

# Objective function for KNN
from sklearn import datasets
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import cross_val_score

X, Y = datasets.load_breast_cancer(return_X_y=True)

def objectiveKNN(args_list): # arg_list is list of hyperpar values
    global X,Y # Data is loaded only once
    evaluations = []
    for hyper_par in args_list:
        clf = KNeighborsClassifier(**hyper_par)
        accuracy  = cross_val_score(clf, X, Y, scoring='accuracy').mean()
        evaluations.append(accuracy)
    return evaluations

tuner_knn = Tuner(param_dict, objectiveKNN)
results = tuner_knn.maximize()

print('best parameters:',results['best_params'])
print('best accuracy:',results['best_objective'])
```
Sample output of above example. Note output may be different for your program.

```
best parameters: {'algorithm': 'auto', 'n_neighbors': 11}
best accuracy: 0.931486122714193
```

<a name="DomainSpace"></a>
# 4. Domain Space of Variables
The domain space defines the search space of variables from which optimal values are chosen. Mango allows definitions of domain space
to define complex search spaces. Domain space definitions are compatible with the [RandomizedSearchCV](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.RandomizedSearchCV.html) of the
scikit-learn. The parameter dictionary concept of scikit-learn is used. Dictionary with parameters names (string) as keys and distributions or 
lists of parameters to try. Distributions must provide a rvs method for sampling (such as those from scipy.stats.distributions). 
If a list is given, it is sampled uniformly. We have defined a new [loguniform](https://gitlab.com/arm-research/isr/mango/blob/master/mango/domain/distribution.py) distribution by extending the scipy.stats.distributions constructs.
**loguniform** can be used in SVM classifiers where *C* parameter can have search space defined using loguniform distribution. In these definitions categorical/discrete variables are defined using list of strings. List of ints is considered
as the variable having interger values. Some of the sample domain space definitions are shown.

```python
from scipy.stats import uniform
param_dict = {"a": uniform(0, 1), # uniform distribution
              "b": range(1,5), # Integer variable
              "c":[1,2,3], # Integer variable
              "d":["-1","1"] # Categorical variable
             }
```

```python
from scipy.stats import uniform
param_dict = {"learning_rate": uniform(0.01, 0.5),
              "gamma": uniform(0.5, 0.5),
              "max_depth": range(1,14),
              "n_estimators": range(500,2000),
              "subsample": uniform(0.7, 0.3),
              "colsample_bytree":uniform(0.3, 0.7),
              "colsample_bylevel":uniform(0.3, 0.7),
              "min_child_weight": range(1,10)}
              
```

```python
from mango.domain.distribution import loguniform
param_dict = {"changepoint_prior_scale": loguniform(-3, 1),
              'seasonality_prior_scale' : loguniform(1, 2)
             }
```

```python
from scipy.stats import uniform
from distribution import loguniform
param_dict = {"kernel": ['rbf'],
              "gamma": uniform(0.1, 4),
              "C": loguniform(-7, 8)}
```


<a name="ObjectiveFunction"></a>
# 5. More on Objective Function
The objective function has the structure as shown below.

```python
def objective_function(args_list):
    evaluations = []
    for hyper_par in args_list:
        result =  evaluate_function_on_hyper_par
        evaluations.append(result)
    return evaluations
```
The objective function is called with the input list of hyper parameters. Each element of the list is the dictionary 
which is a sample drawn from the domain space of variables. Mango expects the objective function to return the list of
evaluations which has the same size as the args_list. Each value of evaluations list is the function evaluated at hyper parameters
of args_list in the same order. A rich set of objective function is shown in the [examples](https://gitlab.com/arm-research/isr/mango/tree/master/mango/examples). The size of args_list is controlled by the batch_size configuration parameter of Mango. By default
batch_size is 1. The configuration parameters of Mango are explained in the [Mango Configurations](#MangoConfigurations) section. 

<a name="MangoConfigurations"></a>
# 6. Controlling Mango Configurations

The default configuration parameters used by the Mango as below:
```python
{'param_dict': ...,
 'userObjective': ...,
 'domain_size': 5000,
 'initial_random': 1,
 'num_iteration': 20,
 'objective': 'maximize',
 'batch_size': 1}
```
The configuration parameters are explained:
- domain_size: The size which is explored in each iteration by the gaussian process. Generally, a larger size is prefered if higher dimensional functions are optimized. More on this will be added with details about the internals of bayesian optimization.
- initial_random: The number of random samples tried.
- num_iteration: The total number of iterations used by Mango to find the optimal value.
- objective: Default objective of maximize is used. minimize objective is not supported yet. minimize objective can be acheived by evaluating the negative of the function.
- batch_size: The size of args_list passed to the objective function for parallel evaluation. For larger batch sizes, Mango internally uses intelligent sampling to decide the optimal samples to evaluate.

The default configuration parameters can be modified as shown below. Only the parameters whose values need to adjusted can be passed as the dictionary.
```python
conf_dict = dict()
conf_dict['batch_size'] = 5
conf_dict['num_iteration'] = 40
conf_dict['domain_size'] = 10000
conf_dict['initial_random'] = 3

tuner_user = Tuner(param_dict, objective_Xgboost,conf_dict) 

# Now tuner_user can be used as shown in other examples.
```

<a name="Celery"></a>
# 7. Schedule Objective Function on Celery
User defined objective function can be scheduled on local, cluster  or cloud infrastructure. The objective function scheduler
is completely independent of the Mango. This design was chosen to enable the scheduling of varied resource objective 
function according to developer needs. We have included examples using [Celery](http://www.celeryproject.org/). In the sample
examples celery workers are used to evaluate the objective function in parallel. These examples assumes celery is installed and workers
are running. Default celery configurations can be modified in the [file](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/classifiers/celery.py).

- [KNN example using celery workers](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/KNN_Celery.ipynb)
- [Prophet example using celery workers](https://gitlab.com/arm-research/isr/mango/blob/master/mango/examples/Prophet_Celery.ipynb)

More examples will be included to show scheduling of objective function using local threads/processes. By default examples schedule
the objective function on the local machine itself.


<a name="contactDetails"></a>
# More Details
Details about specifying parameter/variable domain space, user objective function and internals of Mango will be added.
Please stay tuned. For any questions feel free to reach out to Sandeep Singh Sandha (iotresearch@arm.com)