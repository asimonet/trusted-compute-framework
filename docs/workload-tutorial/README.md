# TCF Worker Application Development Tutorial

This tutorial describes how to build a trusted workload application.
We begin by copying files under directory (folder) `templates/` to
a new directory, `hello_world/workload/`.
Then we show how to modify the files to create a workload application.

The example we create will be a workload application that takes a name as
input and echos back "Hello *name*".

Under directory `hello_world/` are the desired results of modifying the
template files, `hello_world/stage_1` and, with further modifications,
`hello_world/stage_2`

The directory structure for this tutorial is as follows:

* [README.md](README.md) This file
* [templates/](templates/) Templates to copy to create a workload application
  * [CMakeLists.txt](templates/CMakeLists.txt) CMake file to build this application
  * [logic.h](templates/logic.h) Header file defining worker-specific code
  * [logic.cpp](templates/logic.cpp) C file for worker-specific code
  * [plug-in.h](templates/plug-in.h) Header file defining generic plug-in code
  * [plug-in.cpp](templates/plug-in.cpp) C file for generic plug-in code
* [hello_world/](hello_world/) Example workload application
  * [stage_1/](hello_world/stage_1/) Intermediate results from modifying
    template files
    * [CMakeLists.txt](hello_world/stage_1/CMakeLists.txt) Modified to build worker
    * [logic.h](hello_world/stage_1/logic.h)
    * [logic.cpp](hello_world/stage_1/logic.cpp)
    * [plug-in.h](hello_world/stage_1/plug-in.h) Modified to define worker framework
    * [plug-in.cpp](hello_world/stage_1/plug-in.cpp) Modified to implement worker framework
  * [stage_2/](hello_world/stage_2/) Final results from adding worker code
    * [CMakeLists.txt](hello_world/stage_1/CMakeLists.txt)
    * [logic.h](hello_world/stage_1/logic.h) Modified with worker definitions added
    * [logic.cpp](hello_world/stage_1/logic.cpp) Modified with worker code added
    * [plug-in.h](hello_world/stage_1/plug-in.h)
    * [plug-in.cpp](hello_world/stage_1/plug-in.cpp) Modified to call worker

## Prerequisites

Before beginning this tutorial, review the following items:

* Review the base class `WorkloadProcessor`,
  which any workload class inherits, at
  [$TCF_HOME/common/sgx_workload/workload_processor.h](../../common/sgx_workload/workload_processor.h)

  Observe the following:
  * Each workload must implement method `ProcessWorkOrder()`
    (see file [templates/plug-in.cpp](templates/plug-in.cpp))
  * Each workload class definition must include macro
    `IMPL_WORKLOAD_PROCESSOR_CLONE()`
    (see file [templates/plug-in.h](templates/plug-in.h))
  * Each workload class implementation must include macro
    ` REGISTER_WORKLOAD_PROCESSOR()`
    (see file [templates/plug-in.cpp](templates/plug-in.cpp))

* Review the generic command line client at
  [$TCF_HOME/examples/apps/generic_client/](../../examples/apps/generic_client/)
  * This component is optional, but it can be useful because it allows early
    testing of the workload without first creating a custom requester
    application
  * The client application accepts only strings as input parameters and
    assumes that all outputs are also provided as strings.
    If other data types are needed (numbers, binaries), then create
    a custom test application
    (potentially by modifying this application)
  * The Workload ID (`hello-world`) and input parameters (a name string)
    sent by the client must match the workload requirements for this worker

As a best practice, this tutorial separates the actual workload-specific logic
from the the TCF plumbing required to link the workload to the TCF framework
into separate files.

## Tutorial

This tutorial creates a workload application in two phases:
1. [Create generic plug-in logic](#phase1)
2. [Incrementally add workload-specific logic](#phase2)

### <a name="phase1"></a>Phase 1: TCF Plug-in Code

For the first phase copy and modify template code to create TCF plug-in code.
This code contains TCF framework code to invoke worker-specific logic that
will be created next in [Phase 2](#phase2).

* From the top-level TCF source repository directory, `$TCF_HOME`,
  create a new workload directory and change into it:
  ```bash
  mkdir -p examples/apps/hello_world/workload
  cd examples/apps/hello_world/workload
  ```

* Copy five template files to the newly-created directory:
  ```bash
  cp ../../../../docs/workload-tutorial/templates/* .
  ```

* Change placeholders `$CLASS_NAME$` in files `plug-in.h` and `plug-in.cpp`
  to an appropriate workload class name, `HelloWorld`

* Change placeholder `$WORKLOAD_STRING_ID$` in file `plug-in.cpp` to an
  appropriate workload ID, `hello-world` (note the dash, `-`)

* Change placeholder `$WORKLOAD_STATIC_NAME$` in file `CMakeLists.txt`
  to an appropriate name, `hello_world` (note the underscore, `_`)

* To include the new workload into the build,
  add this line to the end of
  [$TCF_HOME/examples/apps/CMakeLists.txt](../../examples/apps/CMakeLists.txt) :

  ```
  ADD_SUBDIRECTORY(hello_world/workload)
  ```

* To link the new workload library into the build, add these lines to
  the end of
  [$TCF_HOME/tc/sgx/trusted_worker_manager/enclave/CMakeLists.txt](../../tc/sgx/trusted_worker_manager/enclave/CMakeLists.txt) :
  ```bash
  # Add $WORKLOAD_STATIC_NAME$ workload
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-L,${TCF_TOP_DIR}/examples/apps/build/$WORKLOAD_STATIC_NAME$/workload")
  TARGET_LINK_LIBRARIES(${PROJECT_NAME} -Wl,--whole-archive -l$WORKLOAD_STATIC_NAME$ -Wl,--no-whole-archive)
  ```
  Replace `$WORKLOAD_STATIC_NAME$` with `hello_world` so it becomes:
  ```bash
  # Add hello_world workload
  SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-L,${TCF_TOP_DIR}/examples/apps/build/hello_world/workload")
  TARGET_LINK_LIBRARIES(${PROJECT_NAME} -Wl,--whole-archive -lhello_world -Wl,--no-whole-archive)
  ```

* Rebuild the framework (see [$TCF_HOME/BUILD.md](../../BUILD.md)).
  It should include the new workload (with the hard-coded placeholder string)

* Load the framework and use the generic command line utility to test the
  newly-added workload:
  ```bash
  ../generic_client/generic_client.py --uri "http://localhost:1947" \
      --workload_id "hello-world" --in_data "Dan"
  ```

* The Hello World worker should return the string `Error: under construction`
  as the result (hard-coded placeholder string in `plug-in.cpp`):
  ```
  [17:35:06 INFO    utility.utility]
  Decryption result at client - Error: under construction
  [17:35:06 INFO    __main__]
  Decrypted response:
   [{'index': 0, 'dataHash':
     '60AEBCFC13614F392352DC5683486C05F5519C927FA35DC254204CA0E5045348',
     'data': 'Error: under construction', 'encryptedDataEncryptionKey': '',
     'iv': ''}]
  ```

To see what the updated source files should look like, refer to the files in
directory [hello_world/stage_1/](docs/workload-tutorial/hello_world/stage_1/).


### <a name="phase2"></a>Phase 2: Worker-specific Code

Now that we have TCF plug-in framework code, we incrementally add
worker-specific logic and call it from the plug-in code.
As a best practice, we separate worker-specific logic from the
TCF framework code that calls it into separate files.
In this example we name the worker-specific function `ProcessHelloWorld()`.

* Add the `ProcessHelloWorld()` function definition to `logic.h`:
  ```cpp
  extern std::string ProcessHelloWorld(std:string in_str);
  ```

* Add the `ProcessHelloWorld()` function implementation to `logic.cpp`
  ```cpp
  std::string ProcessHelloWorld(std::string in_str) {
      return "Hello " + in_str;
  }
  ```

  For this example, the worker-specific logic is trivial. Usually
  the logic is much more complex so it is in a separate file to
  separate it from TCF-specific plug-in code

* Modify the `ProcessWorkOrder()` method in `plug-in.cpp`
  to call `ProcessHelloWorld()`.  That is, change:

  ```cpp
  // Replace the dummy implementation below with invocation of
  // actual logic defined in logic.h and implemented in logic.cpp.
  result_str.assign("Error: under construction");
  ```
  to

  ```cpp
  // Process the input data
  result_str = ProcessHelloWorld(ByteArrayToString(
      wo_data.decrypted_data));
  ```

* Rebuild the framework (see [$TCF_HOME/BUILD.md](../../BUILD.md)).
  It should now include the new workload

* Load the framework and use the generic command line utility to test the
  newly-added workload:
  ```bash
  ../generic_client/generic_client.py --uri "http://localhost:1947" \
      --workload_id "hello-world" --in_data "Dan"
  ```

* The Hello World worker should return a string
  `Hello name` where `name` is the string sent in the first
  input parameter
  ```
  [17:47:48 INFO    utility.utility] Decryption result at client - Hello Dan
  [17:47:48 INFO    __main__]
  Decrypted response:
  [{'index': 0, 'dataHash':
    '02D0D64CA3F5BC43B29304DA25AE9D240A48DE0374C3296A564CE55FB63E0B8C',
    'data': 'Hello Dan', 'encryptedDataEncryptionKey': '', 'iv': ''}]
  ```

To see what the updated source files should look like, refer to the files in
directory [hello_world/stage_2/](docs/workload-tutorial/hello_world/stage_2/).

