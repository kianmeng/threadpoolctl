# Thread-pool Controls [![Build Status](https://dev.azure.com/joblib/threadpoolctl/_apis/build/status/joblib.threadpoolctl?branchName=master)](https://dev.azure.com/joblib/threadpoolctl/_build/latest?definitionId=1&branchName=master) [![codecov](https://codecov.io/gh/joblib/threadpoolctl/branch/master/graph/badge.svg)](https://codecov.io/gh/joblib/threadpoolctl)

Python helpers to limit the number of threads used in the
threadpool-backed of common native libraries used for scientific
computing and data science (e.g. BLAS and OpenMP).

Fine control of the underlying thread-pool size can be useful in
workloads that involve nested parallelism so as to mitigate
oversubscription issues.

## Installation

- For users, install the last published version from PyPI:

  ```bash
  pip install threadpoolctl
  ```

- For contributors, install from the source repository in developer
  mode:

  ```bash
  pip install -r dev-requirements.txt
  flit install --symlink
  ```

  then you run the tests with pytest:

  ```bash
  pytest
  ```

## Usage

### Command Line Interface

Get a JSON description of thread-pools initialized when importing python
packages such as numpy or scipy for instance:

```
python -m threadpoolctl -i numpy scipy.linalg
[
  {
    "filepath": "/home/ogrisel/miniconda3/envs/tmp/lib/libmkl_rt.so",
    "prefix": "libmkl_rt",
    "user_api": "blas",
    "internal_api": "mkl",
    "version": "2019.0.4",
    "num_threads": 2,
    "threading_layer": "intel"
  },
  {
    "filepath": "/home/ogrisel/miniconda3/envs/tmp/lib/libiomp5.so",
    "prefix": "libiomp",
    "user_api": "openmp",
    "internal_api": "openmp",
    "version": null,
    "num_threads": 4
  }
]
```

The JSON information is written on STDOUT. If some of the packages are missing,
a warning message is displayed on STDERR.

### Python Runtime Programmatic Introspection

Introspect the current state of the threadpool-enabled runtime libraries
that are loaded when importing Python packages:

```python
>>> from threadpoolctl import threadpool_info
>>> from pprint import pprint
>>> pprint(threadpool_info())
[]

>>> import numpy
>>> pprint(threadpool_info())
[{'filepath': '/home/ogrisel/miniconda3/envs/tmp/lib/libmkl_rt.so',
  'internal_api': 'mkl',
  'num_threads': 2,
  'prefix': 'libmkl_rt',
  'threading_layer': 'intel',
  'user_api': 'blas',
  'version': '2019.0.4'},
 {'filepath': '/home/ogrisel/miniconda3/envs/tmp/lib/libiomp5.so',
  'internal_api': 'openmp',
  'num_threads': 4,
  'prefix': 'libiomp',
  'user_api': 'openmp',
  'version': None}]

>>> import xgboost
>>> pprint(threadpool_info())
[{'filepath': '/home/ogrisel/miniconda3/envs/tmp/lib/libmkl_rt.so',
  'internal_api': 'mkl',
  'num_threads': 2,
  'prefix': 'libmkl_rt',
  'threading_layer': 'intel',
  'user_api': 'blas',
  'version': '2019.0.4'},
 {'filepath': '/home/ogrisel/miniconda3/envs/tmp/lib/libiomp5.so',
  'internal_api': 'openmp',
  'num_threads': 4,
  'prefix': 'libiomp',
  'user_api': 'openmp',
  'version': None},
 {'filepath': '/home/ogrisel/miniconda3/envs/tmp/lib/libgomp.so.1.0.0',
  'internal_api': 'openmp',
  'num_threads': 4,
  'prefix': 'libgomp',
  'user_api': 'openmp',
  'version': None}]
```

In the above example, `numpy` was installed from the default anaconda channel and comes
with MKL and its Intel OpenMP (`libiomp5`) implementation while `xgboost` was installed
from pypi.org and links against GNU OpenMP (`libgomp`) so both OpenMP runtimes are
loaded in the same Python program.

The state of these libraries is also accessible through the object oriented API:

```python
>>> from threadpoolctl import ThreadpoolController, threadpool_info
>>> from pprint import pprint
>>> import numpy
>>> controller = ThreadpoolController()
>>> pprint(controller.info())
[{'architecture': 'Haswell',
  'filepath': '/home/jeremie/miniconda/envs/dev/lib/libopenblasp-r0.3.17.so',
  'internal_api': 'openblas',
  'num_threads': 4,
  'prefix': 'libopenblas',
  'threading_layer': 'pthreads',
  'user_api': 'blas',
  'version': '0.3.17'}]

>>> controller.info() == threadpool_info()
True
```

### Setting the Maximum Size of Thread-Pools

Control the number of threads used by the underlying runtime libraries
in specific sections of your Python program:

```python
>>> from threadpoolctl import threadpool_limits
>>> import numpy as np

>>> with threadpool_limits(limits=1, user_api='blas'):
...     # In this block, calls to blas implementation (like openblas or MKL)
...     # will be limited to use only one thread. They can thus be used jointly
...     # with thread-parallelism.
...     a = np.random.randn(1000, 1000)
...     a_squared = a @ a
```

The threadpools can also be controlled via the object oriented API, which is especially
useful to avoid searching through all the loaded shared libraries each time. It will
however not act on libraries loaded after the instanciation of the
`ThreadpoolController`:

```python
>>> from threadpoolctl import ThreadpoolController
>>> import numpy as np
>>> controller = ThreadpoolController()

>>> with controller.limit(limits=1, user_api='blas'):
...     a = np.random.randn(1000, 1000)
...     a_squared = a @ a
```

### Restricting the limits to the scope of a function

`threadpool_limits` and `ThreadpoolController` can also be used as decorators to set
the maximum number of threads used by the supported libraries at a function level. The
decorators are accessible through their `wrap` method:

```python
>>> from threadpoolctl import ThreadpoolController, threadpool_limits
>>> import numpy as np
>>> controller = ThreadpoolController()

>>> @controller.wrap(limits=1, user_api='blas')
... # or @threadpool_limits.wrap(limits=1, user_api='blas')
... def my_func():
...     # Inside this function, calls to blas implementation (like openblas or MKL)
...     # will be limited to use only one thread.
...     a = np.random.randn(1000, 1000)
...     a_squared = a @ a
...
```

### Known Limitations

- `threadpool_limits` can fail to limit the number of inner threads when nesting
  parallel loops managed by distinct OpenMP runtime implementations (for instance
  libgomp from GCC and libomp from clang/llvm or libiomp from ICC).

  See the `test_openmp_nesting` function in [tests/test_threadpoolctl.py](
  https://github.com/joblib/threadpoolctl/blob/master/tests/test_threadpoolctl.py)
  for an example. More information can be found at:
  https://github.com/jeremiedbb/Nested_OpenMP

  Note however that this problem does not happen when `threadpool_limits` is
  used to limit the number of threads used internally by BLAS calls that are
  themselves nested under OpenMP parallel loops. `threadpool_limits` works as
  expected, even if the inner BLAS implementation relies on a distinct OpenMP
  implementation.

- Using Intel OpenMP (ICC) and LLVM OpenMP (clang) in the same Python program
  under Linux is known to cause problems. See the following guide for more details
  and workarounds:
  https://github.com/joblib/threadpoolctl/blob/master/multiple_openmp.md

- Setting the maximum number of threads of the OpenMP and BLAS libraries has a global
  effect and impacts the whole Python process. There is no thread level isolation as
  these libraries do not offer thread-local APIs to configure the number of threads to
  use in nested parallel calls.


## Maintainers

To make a release:

Bump the version number (`__version__`) in `threadpoolctl.py`.

Build the distribution archives:

```bash
pip install flit
flit build
```

Check the contents of `dist/`.

If everything is fine, make a commit for the release, tag it, push the
tag to github and then:

```bash
flit publish
```

### Credits

The initial dynamic library introspection code was written by @anton-malakhov
for the smp package available at https://github.com/IntelPython/smp .

threadpoolctl extends this for other operating systems. Contrary to smp,
threadpoolctl does not attempt to limit the size of Python multiprocessing
pools (threads or processes) or set operating system-level CPU affinity
constraints: threadpoolctl only interacts with native libraries via their
public runtime APIs.
