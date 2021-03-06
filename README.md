# SAVer
SAVer (**S**VM **A**bstract **Ver**ifier) is an abstract interpretation based tool for proving properties of SVMs, in particular we aim at proving robustness or vulnerability properties of classifiers.

Given a point *x* and perturbation function *P*, SAVer symbolically computes an overapproximation of *P(x)*, the region of (possibly infinite) points which corresponds to perturbations of *x*, and runs an abstract version of the SVM on it, returning a superset of the labels associated to points in *P(x)*. Whenever such set contains a single label, the concrete SVM classifier is definitively robust on point *x* for perturbation *P*.
![Example of provable robustness](https://raw.githubusercontent.com/svm-abstract-verifier/saver/master/doc/images/example-right.svg?sanitize=true)

When SAVer returns more than one label, it may happen either due to the classifier really being not robust, or because of a loss of precision induced by the abstraction process.
![Example of not provable robustness](https://raw.githubusercontent.com/svm-abstract-verifier/saver/master/doc/images/example-top.svg?sanitize=true)

More information can be found on [Robustness Verification of Support Vector Machines](http://www.math.unipd.it/~ranzato/papers/sas19a.pdf).

## Requirements
 - Any C99-compatible C compiler

## Installation
To install SAVer you need to clone or download the source files from this repository and compile them. There are no additional requirement nor dependencies:

    git clone https://github.com/svm-abstract-verifier/saver.git
or:

    wget https://github.com/svm-abstract-verifier/saver/archive/master.zip
    unzip master.zip
then:

    cd saver/src
    make
    make install
The executable file will be available under `saver/bin/saver`.

Every piece of code is documented using [Doxygen](http://www.doxygen.nl/). If you have Doxygen installed and wish to generate the documentation pages (HTML), run:

    cd saver/src
    make doc
Documentation will be available under `saver/doc/html/index.html`.

## Usage
Run `saver` without arguments for a quick online help message. Full syntax is:

    bin/saver <path to classifier> <path to dataset> [abstraction] [perturbation] [perturbation parameters]
Where
 - *path to classifier*: path to classifier file; must be in the appropriate format (see below)
 - *path to dataset*: path to dataset; must be in the appropriate format (see below)
 - *abstraction*: optional, tells which abstract domain to use for analysis; must be one of *interval*, *raf*, *hybrid* (default: *hybrid*)
 - *perturbation*: optional, type of perturbation; must be one of *l_inf*, *frame*, *from_file* (default: *l_inf*)
 - *perturbation parameters*: list of parameter specific for the type of perturbation

You can find data sets and trained SVM classifiers in our [data-collection repository](https://github.com/svm-abstract-verifier/data-collection/).

## How to read results
SAVer will display both per-sample analysis results and a global summary as in the following example:

                      classifier                  dataset   id epsilon  label  concrete  abstract
    ../classifiers/mnist-svm.dat ../datasets/test-set.csv    0    0.01      7         7       7,9
    ../classifiers/mnist-svm.dat ../datasets/test-set.csv    1    0.01      2         2         2
    ../classifiers/mnist-svm.dat ../datasets/test-set.csv    2    0.01      1         1       1,7
    ../classifiers/mnist-svm.dat ../datasets/test-set.csv    3    0.01      0         0         0
    ../classifiers/mnist-svm.dat ../datasets/test-set.csv    4    0.01      4         4       4,9
    ../classifiers/mnist-svm.dat ../datasets/test-set.csv    5    0.01      1         1       1,7
    ...
    ../classifiers/mnist-svm.dat ../datasets/test-set.csv 9999    0.01      6         6         6
    [SUMMARY]  Size  Epsilon  Avg. Time (ms)  Correct  Robust  Cond. robust
    [SUMMARY] 10000     0.01          53.719     9837    4707          4703
The summary section (last two rows) displays a header and some statistics (from left to right):
 - number of samples in the dataset
 - magnitude of perturbation (same given as input)
 - average analysis time per sample, in milliseconds
 - number of samples on which classifier computed the correct label with respect to the one prescribed in the dataset (correctness)
 - number of samples on which classifier assigned the same labels to every point of an adversarial region, regardeless of correctness (stability)
 - number of samples on which classifier is both correct and stable at the same time (standard notion of robustness)
 
The per-sample section contains one row for every sample in the dataset. Every row shows, from left to right:
 - path to the classifier file
 - path to the dataset file
 - index of the sample in the dataset (starting from 0)
 - magnitude of the perturbation, same given as input
 - label of the sample prescribed by the dataset
 - labels of the sample computed by the classifier
 - sound superset of labels of points in the adversarial region of the sample

When reading last column keep in mind that SAVer's analysis is:
 - **sound**: if a label is not in the list, it is guaranteed that not point in the adversarial region can have that label
 - **incomplete**: if a label is in the list, it is **not** guaranteed that there exists a point in the adversarial region which has that label

## Example
    bin/saver ../data/mnist/svm-rbf-60k.dat ../data/mnist/test-set-normalized.csv raf l_inf 0.01
    
                          classifier                                 dtaset    id  epsilon  label  concrete  abstract
    ../classifiers/mnist-rbf-60k.dat  ../datasets/mnist-test-normalized.csv     0  0.010000     7         7       7,9
    ../classifiers/mnist-rbf-60k.dat  ../datasets/mnist-test-normalized.csv     1  0.010000     2         2         2
    ../classifiers/mnist-rbf-60k.dat  ../datasets/mnist-test-normalized.csv     2  0.010000     1         1       1,7
    ../classifiers/mnist-rbf-60k.dat  ../datasets/mnist-test-normalized.csv     3  0.010000     0         0         0
    ...
    ../classifiers/mnist-rbf-60k.dat  ../datasets/mnist-test-normalized.csv  9999  0.010000     6         6         6
    [SUMMARY]  Size  Epsilon  Avg. Time (ms)  Correct  Robust  Cond. robust
    [SUMMARY] 10000     0.01          53.719     9837    4707          4703

will evaluate whether the classifier in `../data/mnist/svm-rbf-60k.dat` is robust on each point taken from the data set in `../data/mnist/test-set-normalized.csv`, with respect to an L-infinity perturbation of magnitude 1% using the RAF abstract domain for the analysis. Analysis will reveal that classifier computes the correct label 98.37% of the times, exhibits a stable behavior (i.e. it consistenlty assigns the same label to points in the same adversarial region) 47.07% of the times, and it is both correct and stable 47.03% of the times. There exist `4707 - 4703 = 4` samples in the dataset on which classifier is provably stable, but does not output the expected label: by checking the per-sample results it is possible to identify those samples.

## Classifier format
SAVer supports OVO SVM classifier files in the following format:

    ovo
    <feature space size>
    <number of classes>
    <kernel type and parameters>
    <class 1> <number of support vectors for class 1>
    <class 2> <number of support vectors for class 2>
    ...
    <class N> <number of support vectors for class N>
    <alpha coefficients>
    <support vectors>
    <biases>
Where alpha, support vectors and biases are stored as described by [Support Vector Machines - scikit-learn](https://scikit-learn.org/stable/modules/svm.html#multi-class-classification) documentation. You may also want to check [our data collection repository](https://github.com/svm-abstract-verifier/data-collection) for some examples, in particular the easy-to-read [Iris linear SVM classifier](https://github.com/svm-abstract-verifier/data-collection/blob/master/models/iris/svm-linear.dat.zip).

In particular, every SVM trained with `scikit-learn` and `libsvm` is compatible with this format.

## Data set format
See [dedicated section on our data-collection repository](https://github.com/svm-abstract-verifier/data-collection#dataset-format), from which you can also download some ready-to-use [datasets](https://github.com/svm-abstract-verifier/data-collection/tree/master/datasets) and [models](https://github.com/svm-abstract-verifier/data-collection/tree/master/models).

## Perturbations
SAVer supports hyperrectangle perturbations read from a text file. Each line of the file will be matched with the corresponding sample in the dataset, i.e. i-esim sample will be tesed using i-esim row, every row describing a hyperrectangle through pairs of lower and upper bounds, one pair for each component. For example, a perturbation file for *N* samples in a space size of *K* will look like:

    sample1_l1 sample1_u1 sample1_l2 sample1_u2 ... sample1_lK sample1_uK
    sample2_l1 sample2_u1 sample2_l2 sample2_u2 ... sample2_lK sample2_uK
    ...
    sampleN_l1 sampleN_u1 sampleN_l2 sampleN_u2 ... sampleN_lK sampleN_uK
Note that a hyperrectangle region defined in this way does not need to contain the sample it is associated to.
