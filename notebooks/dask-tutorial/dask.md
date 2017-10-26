
# 1. Dask Overview

Dask is a flexible parallel computing library for analytics. Parallel computing is a type of computation in which many calculations or the execution of processes are carried out simultaneously. Large problems can often be divided into smaller ones, which can then be solved at the same time.

Dask is composed of two components:

1. Dynamic task scheduling optimized for computation. This is similar to Airflow, Luigi, Celery, or Make, 
   but optimized for interactive computational workloads.
2. “Big Data” collections like parallel arrays, dataframes, and lists that extend common interfaces like NumPy, 
   Pandas, or Python iterators to larger-than-memory or distributed environments. These parallel collections run 
   on top of the dynamic task schedulers.


Dask emphasizes the following virtues:

1. Familiar: Provides parallelized NumPy array and Pandas DataFrame objects.
2. Flexible: Provides a task scheduling interface for more custom workloads and integration with other projects.
3. Native: Enables distributed computing in Pure Python with access to the PyData stack.
4. Fast: Operates with low overhead, low latency, and minimal serialization necessary for fast numerical algorithms
5. Scales up: Runs resiliently on clusters with 1000s of cores
6. Scales down: Trivial to set up and run on a laptop in a single process
7. Responsive: Designed with interactive computing in mind it provides rapid feedback and diagnostics to aid humans


The layout of the tutorial will be as follows:

a) Foundations: an explanation of what Dask is, how it works, and how to use lower-level primitives to set up computations.             
             Casual users may wish to skip this section, although we consider it useful knowledge for all users.
b) Distributed: information on running Dask on the distributed scheduler, which enables scale-up to distributed settings and  
             enhanced monitoring of task operations. The distributed scheduler is now generally the recommended engine for 
             executing task work, even on single workstations or laptops.
c) Collections: convenient abstractions giving a familiar feel to big data.
       bag: Python iterators with a functional paradigm, such as found in func/iter-tools and toolz - generalize 
            lists/generators to big data; this will seem very familiar to users of PySpark's RDD
     array: massive multi-dimensional numerical data, with Numpy functionality
     dataframes: massive tabular data, with Pandas functionality

## Prepare artificial data

 From the repo directory


```python
python prep.py
```

This will generate all the necessary files for the rest of the tutorials.

You will need the following core libraries:


```python
conda install numpy pandas h5py Pillow matplotlib scipy toolz pytables fastparquet

```

And a recently updated version of Dask


```python
conda install dask distributed
```

The following is useful for task graph visualization:


```python
conda install graphviz
```


```python
# in directory dask-tutorial/
# this takes a little while
%run prep.py
```

# 2. How Dask Works

As Python programmers, you probably already perform certain tricks to enable computation of larger-than-memory datasets, parallel execution or delayed/background execution. Perhaps with this phrasing, it is not clear what we mean, but a few examples should make things clearer. The point of Dask is to make simple things easy and complex things possible!

Aside from the detailed introduction, we can summarize the basics of Dask as follows:

    process data that doesn't fit into memory by breaking it into blocks and specifying task chains
    parallelize execution of tasks across cores and even nodes of a cluster
    move computation to the data rather than the other way around, to minimize communication overheads

All of this allows you to get the most out of your computation resources, but program in a way that is very familiar: for-loops to build basic tasks, Python iterators, and the Numpy (array) and Pandas (dataframe) functions for multi-dimensional or tabular data, respectively.

Dask allows you to construct a prescription for the calculation you want to carry out. That may sound strange, 
but a simple example will demonstrate that you can achieve this while programming with perfectly ordinary Python 
functions and for-loops.


```python
from dask import delayed

@delayed
def inc(x):
    return x + 1

@delayed
def add(x, y):
    return x + y
```

Here we have used the delayed annotation to show that we want these functions to operate lazily - to save the set of inputs and execute only on demand.

# 3. Dask Bag mimics iterators, Toolz, and PySpark

Dask-bag excels in processing data that can be represented as a sequence of arbitrary inputs. We'll refer to this as "messy" data, because it can contain complex nested structures, missing fields, mixtures of data types, etc. The functional programming style fits very nicely with standard Python iteration, such as can be found in the itertools module.

Messy data is often encountered at the beginning of data processing pipelines when large volumes of raw data are first consumed. The initial set of data might be JSON, CSV, XML, or any other format that does not enforce strict structure and datatypes. For this reason, the initial data massaging and processing is often done with Python lists, dicts, and sets.

These core data structures are optimized for general-purpose storage and processing. Adding streaming computation with iterators/generator expressions or libraries like itertools or toolz let us process large volumes in a small space. If we combine this with parallel processing then we can churn through a fair amount of data.

Dask.bag is a high level Dask collection to automate common workloads of this form. In a nutshell

dask.bag = map, filter, toolz + parallel execution

You can create a Bag from a Python sequence, from files, from data on S3, etc..


```python
# each element is an integer
import dask.bag as db
b = db.from_sequence([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
```


```python
# each element is a text file of JSON lines
import os
b = db.read_text(os.path.join('data', 'accounts.*.json.gz'))
```


```python
# Requires `s3fs` library
# each element is a remote CSV text file
b = db.read_text('s3://dask-data/nyc-taxi/2015/yellow_tripdata_2015-01.csv')
```

Bag objects hold the standard functional API found in projects like the Python standard library, toolz, or pyspark, including map, filter, groupby, etc..

As with Array and DataFrame objects, operations on Bag objects create new bags. Call the .compute() method to trigger execution.


```python
def is_even(n):
    return n % 2 == 0

b = db.from_sequence([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
c = b.filter(is_even).map(lambda x: x ** 2)
c
```


```python
# blocking form: wait for completion (which is very fast in this case)
c.compute()
```

For more details on Dask Bag check https://dask.pydata.org/en/latest/bag.html

# 4. Concurrent Features by Dask

Dask supports a real-time task framework that extends Python’s concurrent.futures interface. This interface is good 
for arbitrary task scheduling, like dask.delayed, but is immediate rather than lazy, which provides some more flexibility 
in situations where the computations may evolve over time. These features depend on the second generation task scheduler 
found in dask.distributed (which, despite its name, runs very well on a single machine).

As we saw in Foundations, Dask allows you to simply construct graphs of tasks with dependencies. In fact, if you skip forward, you will find that graphs can also be created automatically for you using functional, Numpy or Pandas syntax on data collections. None of this would be very useful, if there weren't also a way to execute these graphs, in a parallel and memory-aware way. Dask comes with four available schedulers:

    a) dask.threaded.get: a scheduler backed by a thread pool
    b) dask.multiprocessing.get: a scheduler backed by a process pool
    c) dask.async.get_sync: a synchronous scheduler, good for debugging
    d) distributed.Client.get: a distributed scheduler for executing graphs on multiple machines.

Here is a simple program for dask.distributed library:


```python
from dask.distributed import Client
client = Client('scheduler:port')

futures = []
for fn in filenames:
    future = client.submit(load, fn)
    futures.append(future)

summary = client.submit(summarize, futures)
summary.result()
```

For more details on Concurrent Features by Dask check https://dask.pydata.org/en/latest/futures.html

# 5. Dask Array 

Dask arrays implement a subset of the NumPy interface on large arrays using blocked algorithms and task scheduling.
These behave like numpy arrays, but break a massive job into tasks that are then executed by a scheduler. The default 
scheduler uses threading but you can also use multiprocessing or distributed or even serial processing (mainly for debugging). You can tell the dask array how to break the data into chunks for processing.


```python
import dask.array as da
f = h5py.File('myfile.hdf5')             
x = da.from_array(f['/big-data'], chunks=(1000, 1000))
x - x.mean(axis=1).compute()
```

For more details on Dask Array check https://dask.pydata.org/en/latest/array.html

# 6. Dask DataFrame

A Dask DataFrame is a large parallel dataframe composed of many smaller Pandas dataframes, split along the index. 
These pandas dataframes may live on disk for larger-than-memory computing on a single machine, or on many different 
machines in a cluster. Dask.dataframe implements a commonly used subset of the Pandas interface including elementwise 
operations, reductions, grouping operations, joins, timeseries algorithms, and more. It copies the Pandas interface 
for these operations exactly and so should be very familiar to Pandas users. Because Dask.dataframe operations merely 
coordinate Pandas operations they usually exhibit similar performance characteristics as are found in Pandas. To run the following code, save 'Student_Name_ID_Email.csv' file in your machine.


```python
import pandas as pd                     
df = pd.read_csv('data.csv')      
d = df.groupby(df.HID).Serial_No.mean()
print(d)
```

    HID
    101     1
    102     2
    104     3
    105     4
    106     5
    107     6
    109     7
    111     8
    201     9
    202    10
    Name: Serial_No, dtype: int64
    


```python
import dask.dataframe as dd
df = dd.read_csv('data.csv')
dt = df.groupby(df.HID).Serial_No.mean().compute()
print (dt)
```

    HID
    101     1.0
    102     2.0
    104     3.0
    105     4.0
    106     5.0
    107     6.0
    109     7.0
    111     8.0
    201     9.0
    202    10.0
    Name: Serial_No, dtype: float64
    

For more details on Dask DataFrame check https://dask.pydata.org/en/latest/dataframe.html

# 7. Dask DataFrame Storage

Efficient storage can dramatically improve performance, particularly when operating repeatedly from disk.

Decompressing text and parsing CSV files is expensive. One of the most effective strategies with medium data is to use a binary storage format like HDF5. 


```python
# be sure to shut down other kernels running distributed clients
from dask.distributed import Client
client = Client()
```

Create data if we don't have any


```python
from prep import accounts_csvs
accounts_csvs(3, 1000000, 500)
```



First we read our csv data as before.

CSV and other text-based file formats are the most common storage for data from many sources, because they require minimal pre-processing, can be written line-by-line and are human-readable. Since Pandas' read_csv is well-optimized, CSVs are a reasonable input, but far from optimized, since reading required extensive text parsing.



```python
import os
filename = os.path.join('data', 'accounts.*.csv')
filename
```


```python
import dask.dataframe as dd
df_csv = dd.read_csv(filename)
df_csv.head()
```



HDF5 and netCDF are binary array formats very commonly used in the scientific realm.

Pandas contains a specialized HDF5 format, HDFStore. The dd.DataFrame.to_hdf method works exactly like the pd.DataFrame.to_hdf method.



```python
target = os.path.join('data', 'accounts.h5')
target
```


```python
%time df_csv.to_hdf(target, '/data')
```


```python
df_hdf = dd.read_hdf(target, '/data')
df_hdf.head()
```

For more information of Dask DataFrame Storage, click http://dask.pydata.org/en/latest/dataframe-create.html

# Useful Resources:

1. https://dask.pydata.org/en/latest/
2. http://matthewrocklin.com/blog/work/2017/10/16/streaming-dataframes-1
3. http://people.duke.edu/~ccc14/sta-663-2017/18A_Dask.html
4. https://www.kdnuggets.com/2016/09/introducing-dask-parallel-programming.html
5. https://pypi.python.org/pypi/dask/
6. https://www.hdfgroup.org/2015/03/hdf5-as-a-zero-configuration-ad-hoc-scientific-database-for-python/
7. https://github.com/dask/dask-tutorial
