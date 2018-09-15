Tutorial
========

A step by step tutorial for installing and launching a denzel deployment.

Prerequisites
-------------

| Python 3 and docker are necessary for denzel.
| Optionally, you can set up a `virtualenv`_.

1. `Install docker`_
2. (Optional) `Install virtualenv`_ and create one to work on.

.. note::
    | If opting for ``virtualenv`` understand that the deployment will not be using this env - it will only be used to install denzel on it.
    | The deployment itself will use the interpreter and packages installed into the containers (more on this further down).

.. _`Install docker`: https://docs.docker.com/install/
.. _`virtualenv`: https://virtualenv.pypa.io/en/stable/
.. _`Install virtualenv`: https://virtualenv.pypa.io/en/stable/installation/


.. _`install`:

Installation
------------

| If you are using a virtualenv, make sure you have it active.
| To install denzel, simply run

.. code-block:: bash

    $ pip install denzel

| After this command completes, you should have the ``denzel`` :doc:`cli` available.
| Once available you can always call ``denzel --help`` for the help menu.


.. _toy_model:

Toy Model
---------

| Before actually using denzel, we need a trained model saved locally.
| If you already have one, you can skip this sub-section.
|
| For this tutorial, we are going to use `scikit-learn's default SVM classifier`_ and train it on the `Iris dataset`_.
| The following script downloads the dataset, trains a model and saves it to disk

.. code-block:: python3

    import pandas as pd
    from sklearn.svm import SVC
    from sklearn.model_selection import train_test_split
    import pickle

    # -------- Load data --------
    IRIS_DATA_URL = "https://archive.ics.uci.edu/ml/machine-learning-databases/iris/iris.data"
    IRIS_DATA_COLUMNS = ['sepal-length', 'sepal-width', 'petal-length', 'petal-width', 'class']

    iris_df = pd.read_csv(IRIS_DATA_URL,
                          names=IRIS_DATA_COLUMNS)


    # -------- Spit train and test --------
    features, labels = iris_df.values[:, 0:4], iris_df.values[:, 4]

    test_fraction = 0.2
    train_features, test_features, train_labels, test_labels = train_test_split(
        features, labels,
        test_size=test_fraction)

    # -------- Train and evaluate --------
    model = SVC()
    model.fit(X=train_features, y=train_labels)

    print(model.score(X=test_features, y=test_labels))
    >> 0.9666666666666667

    # -------- Save for later --------
    SAVED_MODEL_PATH = '/home/creasy/saved_models/iris_svc.pkl'
    with open(SAVED_MODEL_PATH, 'wb') as saved_file:
        pickle.dump(
            obj=model,
            file=saved_file)


| Great, we have a model trained to populate the deployment!


.. _`scikit-learn's default SVM classifier`: http://scikit-learn.org/stable/modules/svm.html#svm-classification
.. _`Iris dataset`: https://archive.ics.uci.edu/ml/datasets/Iris


Starting a denzel Project
-------------------------

| To start a project, you first have to run the command :ref:`startproject`, as a result denzel will build for you the following skeleton

.. code-block:: bash

    $ denzel startproject iris_classifier
    Successfully built iris_classifier project skeleton
    $ cd iris_classifier
    $ tree
    .
    |-- Dockerfile
    |-- __init__.py
    |-- app
    |   |-- __init__.py
    |   |-- assets
    |   |   `-- info.txt
    |   |-- logic
    |   |   |-- __init__.py
    |   |   |-- assets
    |   |   `-- pipeline.py
    |   `-- tasks.py
    |-- docker-compose.yml
    |-- logs
    `-- requirements.txt

| To make denzel fully operational, the only files we'll ever edit are:
| 1. ``requirements.txt`` - Here we'll store all the pip packages our system needs
| 2. ``app/assets/info.txt`` - Text file that contains information about our model and system
| 3. ``app/logic/pipeline.py`` - Here we will edit the body of the :doc:`pipeline`

.. tip::

    | A good practice will be to edit only the body of functions in ``pipeline.py`` and if you wish to add your own custom functions that will be called from within ``pipeline.py``, you should put them on a separate file inside the ``app/logic`` directory and import them.


Requirements
------------

| When we've built our toy model, we used ``scikit-learn`` so before anything we want to specify this requirement in the ``requirements.txt`` file.
| Open your favorite file editor, and append ``scikit-learn``, ``numpy`` and ``scipy`` as requirements - don't forget to leave a blank line in the end.
| Your ``requirements.txt`` should look like this

.. code-block:: text

    # ---------------------------------------------------------------
    #                           USER GUIDE
    # Remember this has to be a lightweight service;
    # Keep that in mind when choosing which libraries to use.
    # ---------------------------------------------------------------
    scikit-learn
    numpy
    scipy


| Take heed to the comment at the top of the file. Keep your system as lean as possible using light packages and operations.

.. note::

    | The packages specified in ``requirements.txt`` will be installed only once, on the first call of the :ref:`launch` command.
    | If you wish to add packages later you can always use the :ref:`pinstall` command.


.. _`api_interface`:

Define Interface (API)
----------------------

| Our end users will need to know what is the JSON scheme our API accepts, so we will have to define what is the accepted JSON scheme for the :ref:`predict_endpoint` endpoint.
| In our :ref:`toy model <toy_model>`, we have four features the model expects: 'sepal-length', 'sepal-width', 'petal-length' and 'petal-width'.
| Since we are going to return an :ref:`async response <tasks_and_synchrony>`, we also need to make sure we include a callback URI in the scheme.
| Finally we'll want to support batching, so the following JSON scheme should suffice

.. code-block:: json

    {
        "callback_uri": <callback_uri>,
        "data": {<unique_id1>: {"sepal-length": <float>,
                                "sepal-width": <float>,
                                "petal-length": <float>,
                                "petal-width": <float>},
                 <unique_id2>: {"sepal-length": <float>,
                                "sepal-width": <float>,
                                "petal-length": <float>,
                                "petal-width": <float>},
                 ...}
    }

| Also let's include a documentation of this interface and the model version in our ``app/assets/info.txt`` file that will be available to the end user in the :ref:`info_endpoint` endpoint.
| For example we might edit ``info.txt`` to something like this

.. parsed-literal::

    # =====================  DEPLOYMENT  ======================

        ██████╗ ███████╗███╗   ██╗███████╗███████╗██╗
        ██╔══██╗██╔════╝████╗  ██║╚══███╔╝██╔════╝██║
        ██║  ██║█████╗  ██╔██╗ ██║  ███╔╝ █████╗  ██║
        ██║  ██║██╔══╝  ██║╚██╗██║ ███╔╝  ██╔══╝  ██║
        ██████╔╝███████╗██║ ╚████║███████╗███████╗███████╗
        ╚═════╝ ╚══════╝╚═╝  ╚═══╝╚══════╝╚══════╝╚══════╝
                             |project_version|

    # ========================  MODEL  ========================

    Model information:
        Version: 1.0.0
        Description: Iris classifier

    For prediction, make a POST request for /predict matching the following scheme

    {
        "callback_uri": "http://alonzo.trainingday.com/stash",
        "data": {<unique_id1>: {"sepal-length": <float>,
                                "sepal-width": <float>,
                                "petal-length": <float>,
                                "petal-width": <float>},
                 <unique_id2>: {"sepal-length": <float>,
                                "sepal-width": <float>,
                                "petal-length": <float>,
                                "petal-width": <float>},
                 ...}
    }

| Looks great, now end users can see this info using GET requests!


Launch (partial project)
------------------------

| Even though we haven't completed all our tasks to make the deployment fully functional, now is a good time for a sanity check.
| What we have now is a skeleton, an editted ``info.txt`` and ``requirements.txt`` files and we can launch our API, without the functionality of the :ref:`predict_endpoint` endpoint (yet).
| Inside project directory run:

.. code-block:: bash

    $ denzel launch

    Creating network "irisclassifier_default" with the default driver
    Pulling redis (redis:3.2.11)...
    3.2.11: Pulling from library/redis
    4d0d76e05f3c: Pull complete
    cfbf30a55ec9: Pull complete
    82648e31640d: Pull complete
    1d925e96c510: Pull complete
    .
    .


| If this is the first time you launch a denzel project, the necessary images will be downloaded and built.
| What is going on in the background is necessary for building the containers that will power the deployment.
| This might take a few minutes, so sit back and enjoy an `Oscar winning performance by the man himself`_.

.. _`Oscar winning performance by the man himself`: https://youtu.be/6KrNpxODiDA

.. note::

    By default denzel will occupy port 8000 for the API and port 5555 for monitoring. If one of them is taken, denzel will let you know and you can opt for other ports - for more info check the :ref:`launch` command documentation.

| Once done if everything went right you should see the end of the output looking like this:

.. code-block:: bash

    Creating irisclassifier_redis_1   ... done
    Creating irisclassifier_api_1     ... done
    Creating irisclassifier_denzel_1  ... done
    Creating irisclassifier_monitor_1 ... done

| This indicates that all the containers (services) were created and are up.

.. tip::

    You can always check the status of the services using the :ref:`status` command. Normally all should be up or all down.

| For sanity check, assuming you have deployed locally, open your favorite browser and go to http://localhost:8000/info . You should see the contents of ``info.txt``
| At any time, you can stop all services using the :ref:`stop` command and start them again with the :ref:`start` command.
| From this moment forward we shouldn't use the :ref:`launch` command as a project can and needs to be launched once.
| If for any reason you wish to relaunch a project (for changing ports for example) you'd have to first :ref:`shutdown` and then :ref:`launch` again.


Pipeline Methods
----------------

| Now is the time to fill the body of the :doc:`pipeline methods <pipeline>`. They are all stored inside ``app/logic/pipeline.py``.
| Open this file in your favorite IDE as we will go through the implementation of these methods.

^^^^^^^^^^^^^^
``load_model``
^^^^^^^^^^^^^^

| :ref:`pipeline_load_model` is the method responsible for loading our saved model into memory and will keep it there as long as the worker lives.
| So our model will be accessible for reading, we must copy it into the project directory, preferably to ``app/assets``
| Once copied there, the assets directory should be as follows:

.. code-block:: bash

    $ cd app/assets/
    $ ls -l

    total 8
    -rw-rw-r-- 1 creasy creasy 1623 Sep 14 14:35 info.txt
    -rw-rw-r-- 1 creasy creasy 3552 Sep 14 08:55 iris_svc.pkl

| Now if we'll look at ``app/logic/pipeline.py`` we will find the skeleton of :ref:`pipeline_load_model`.
| Edit it so it loads the model and returns it, it should look something like:


.. code-block:: python

    import pickle

    .
    .

    def load_model():
        """
        Load model and its assets to memory
        :return: Model, will be used by the predict and process functions
        """
        with open('./app/assets/iris_svc.pkl', 'rb') as model_file:
            loaded_model = pickle.load(model_file)

        return loaded_model


.. note::

    | When using paths on code which is executed inside the containers (like the pipeline methods) the current directory is always the project main directory (where the ``requirements.txt`` is stored). Hence the saved model prefix above is ``./app/assets/...``.

| When we edit the pipeline methods, the changes do not take effect until we restart the services.
| As we just edited a pipeline method, we should run the :ref:`restart` command so the changes apply.
| Navigate back into the project main directory and run ``denzel restart`` and after the services have restarted the changes will take effect.
| To verify all went well you can examine the logs by running the :ref:`logs` command - if anything went wrong we will see it there.


^^^^^^^^^^^^^^^^
``verify_input``
^^^^^^^^^^^^^^^^

| Now that we took care of the model loading, next we will implement the methods that actually apply to the request sent by the end user.
| The :ref:`pipeline_verify_input` function is responsible for making sure the JSON data received matches the :ref:`interface we defined <api_interface>`.
| In order to do that, lets edit the :ref:`pipeline_verify_input` method to do just that:

.. code-block:: python

    .
    .
    FEATURES = ['sepal-length', 'sepal-width', 'petal-length', 'petal-width']

    def verify_input(json_data):
        """
        Verifies the validity of an API request content

        :param json_data: Parsed JSON accepted from API call
        :type json_data: dict
        :return: Data for the the process function
        """

        # callback_uri is needed to sent the responses to
        if 'callback_uri' not in json_data:
            raise ValueError('callback_uri not supplied')

        # Verify data was sent
        if 'data' not in json_data:
            raise ValueError('no data to predict for!')

        # Verify data structure
        if not isinstance(json_data['data'], dict):
            raise ValueError('jsondata["data"] must be a mapping between unique id and features')

        # Verify data scheme
        for unique_id, features in json_data['data'].items():
            feature_names = features.keys()
            feature_values = features.values()

            # Verify all features needed were sent
            if not all([feature in feature_names for feature in FEATURES]):
                raise ValueError('For each example all of the features [{}] must be present'.format(FEATURES))

            # Verify all features that were sent are floats
            if not all([isinstance(value, float) for value in feature_values]):
                raise ValueError('All feature values must be floats')

        return json_data

| In the verification process implementation, you may throw any object that inherits from ``Exception`` and the message attached to it will be sent back to the user in case he tackles that exception.

.. tip::

    For JSON scheme verification, you can consider using the `jsonschema`_ library.

    .. _`jsonschema`: https://github.com/Julian/jsonschema


^^^^^^^^^^^
``process``
^^^^^^^^^^^

| The output of the :ref:`pipeline_verify_input` and :ref:`pipeline_load_model` methods are the input to the :ref:`pipeline_process` method.
| The model object itself is not always necessary, but it is there if you want to have some kind of loaded resource available for the processing, in this tutorial we won't use the model in this method.
|
| Now we are in possession of the JSON data, and we are already sure it has all the necessary data for making predictions.
| Our model though, does not accept JSON, it expects four floats as input, so in this method we will turn the JSON data into model ready data.
| For our use case, we should edit the function to look as follows:

.. code-block:: python

    .
    .
    import numpy as np
    .
    .

    def process(model, json_data):
        """
        Process the json_data passed from verify_input to model ready data

        :param model: Loaded object from load_model function
        :param json_data: Data from the verify_input function
        :return: Model ready data
        """

        # Gather unique IDs
        ids = json_data['data'].keys()

        # Gather feature values and make sure they are in the right order
        data = []
        for features in json_data['data'].values():
            data.append([features[FEATURES[0]], features[FEATURES[1]], features[FEATURES[2]], features[FEATURES[3]]])

        data = np.array(data)
        """
        data = [[float, float, float, float],
                [float, float, float, float]]
        """

        return ids, data


^^^^^^^^^^^
``predict``
^^^^^^^^^^^

| The output of :ref:`pipeline_process` and :ref:`pipeline_load_model` are the input to the :ref:`pipeline_predict` method.
| The final part of a request lifecycle is the actual prediction that will be sent back as response.
| In our example in order to do that we would edit the method to look as follows:

.. code-block:: python

    def predict(model, data):
        """
        Predicts and prepares the answer for the API-caller

        :param model: Loaded object from load_model function
        :param data: Data from process function
        :return: Response to API-caller
        :rtype: dict
        """

        # Unpack the outputs of process function
        ids, data = data

        # Predict
        predictions = model.predict(data)

        # Pack the IDs supplied by the end user and their corresponding predictions in a dictionary
        response = dict(zip(ids, predictions))

        return response

.. warning::

    The returned value of the :ref:`pipeline_predict` function must be a dictionary. This is necessary because denzel will parse it into JSON to be sent back to the end user.

| And... That's it! Denzel is ready to be fully operational.
| Don't forget, after all these changes we must run ``denzel restart`` so they will take effect.
| For reference, the full ``pipeline.py`` file should look like this

.. code-block:: python

    import pickle
    import numpy as np

    FEATURES = ['sepal-length', 'sepal-width', 'petal-length', 'petal-width']

    # -------- Handled by api container --------
    def verify_input(json_data):
        """
        Verifies the validity of an API request content

        :param json_data: Parsed JSON accepted from API call
        :type json_data: dict
        :return: Data for the the process function
        """

        # callback_uri is needed to sent the responses to
        if 'callback_uri' not in json_data:
            raise ValueError('callback_uri not supplied')

        # Verify data was sent
        if 'data' not in json_data:
            raise ValueError('no data to predict for!')

        # Verify data structure
        if not isinstance(json_data['data'], dict):
            raise ValueError('jsondata["data"] must be a mapping between unique id and features')

        # Verify data scheme
        for unique_id, features in json_data['data'].items():
            feature_names = features.keys()
            feature_values = features.values()

            # Verify all features needed were sent
            if not all([feature in feature_names for feature in FEATURES]):
                raise ValueError('For each example all of the features [{}] must be present'.format(FEATURES))

            # Verify all features that were sent are floats
            if not all([isinstance(value, float) for value in feature_values]):
                raise ValueError('All feature values must be floats')

        return json_data


    # -------- Handled by denzel container --------
    def load_model():
        """
        Load model and its assets to memory

        :return: Model, will be used by the predict and process functions
        """
        with open('./app/assets/iris_svc.pkl', 'rb') as model_file:
            loaded_model = pickle.load(model_file)

        return loaded_model


    def process(model, json_data):
        """
        Process the json_data passed from verify_input to model ready data

        :param model: Loaded object from load_model function
        :param json_data: Data from the verify_input function
        :return: Model ready data
        """

        # Gather unique IDs
        ids = json_data['data'].keys()

        # Gather feature values and make sure they are in the right order
        data = []
        for features in json_data['data'].values():
            data.append([features[FEATURES[0]], features[FEATURES[1]], features[FEATURES[2]], features[FEATURES[3]]])

        data = np.array(data)

        return ids, data


    def predict(model, data):
        """
        Predicts and prepares the answer for the API-caller

        :param model: Loaded object from load_model function
        :param data: Data from process function
        :return: Response to API-caller
        :rtype: dict
        """

        # Unpack the outputs of process function
        ids, data = data

        # Predict
        predictions = model.predict(data)

        # Pack the IDs supplied by the end user and their corresponding predictions in a dictionary
        response = dict(zip(ids, predictions))

        return response


Using the API to Predict
------------------------

| Now is the time to put denzel into action.
| To do that, we must first have some URI to receive the responses (remember, we are using :ref:`async responses <tasks_and_synchrony>`).
| You can do that by using `waithook`_ which is an in browser service for receiving HTTP requests, just what we need - just follow the link, choose a "Path Prefix" (for example ``john_q`` and press "Subscribe".
| Use the link that will be generated for you (http://waithook.com/?path=<chosen_path_prefix>) and keep the browser open as we will receive the responses to the output window.
| Next we need to make an actual POST request to the :ref:`predict_endpoint` endpoint. We will do that using `curl`_ through the command line.

.. tip::
    | There are more intuitive ways to create HTTP requests than `curl`_. For creating requests through UI you can either use `Postman`_, or through Python using the `requests`_ package.

| Let's launch a predict request, for two examples from the test set:

.. tabs::

    .. code-tab:: bash

        $ curl --header "Content-Type: application/json" \
        > --request POST \
        > --data '{"callback_uri": "http://waithook.com/?path=john_q",'\
        > '"data": {"a123": {"sepal-length": 4.6, "sepal-width": 3.6, "petal-length": 1.0, "petal-width": 0.2},'\
        > '"b456": {"sepal-length": 6.5, "sepal-width": 3.2, "petal-length": 5.1, "petal-width": 2.0}}}' \
        http://localhost:8000/predict

    .. code-tab:: python

        import requests

        headers = {
            'Content-Type': 'application/json',
        }

        data = {
          "callback_uri": "http://waithook.com/?path=john_q",
          "data": {"a123": {"sepal-length": 4.6, "sepal-width": 3.6, "petal-length": 1.0, "petal-width": 0.2},
                   "b456": {"sepal-length": 6.5, "sepal-width": 3.2, "petal-length": 5.1, "petal-width": 2.0}}
        }

        response = requests.post('http://localhost:8000/predict', headers=headers, data=data)


| If the request has passed the :ref:`pipeline_verify_input` method, you should immediately get a response that looks something like:

.. code-block:: json

    {"status":"success","data":{"task_id":"19e39afe-0729-43a8-b4c5-6a60281157bc"}}

| This means that the task has already entered the task queue and will next go through :ref:`pipeline_process` and :ref:`pipeline_predict`.
| At any time, you can view the task status by sending a GET request to the :ref:`status_endpoint` endpoint.
| If you examine waithook in your browser, you will see that a response was already sent back with the prediction, it should looks something like:

.. code-block:: json

    {
      "method": "POST",
      "url": "/john_q",
      "headers": {
        "User-Agent": "python-requests/2.19.1",
        "Connection": "close",
        "X-Forwarded-Proto": "http",
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Content-Length": "49",
        "Content-Type": "application/json",
        "Host": "waithook.com",
        "X-Forwarded-for": "89.139.202.80"
      },
      "body": "{\"a123\": \"Iris-setosa\", \"b456\": \"Iris-virginica\"}"
    }

| In the ``"body"`` section, you can see the returned predictions.
| If you got this response it means that all went well and your deployment is fully ready.

.. _`waithook`: http://waithook.com/
.. _`curl`: https://curl.haxx.se/docs/manual.html
.. _`Postman`: https://www.getpostman.com/
.. _`requests`: http://docs.python-requests.org/en/master/

Monitoring
----------

| Denzel comes with a built in UI for monitoring the tasks and workers.
| To use it, once the system is up go to the monitor port (defaults to 5555) on the deployment domain. If deployed locally open your browser and go to http://localhost:5555
| You will be presented with a UI that looks something like:

.. figure:: _static/monitor_ui.png

    Example of Flower's monitoring UI

| This dashboard is generated by `Flower`_, and gives you access to examine the worker status, tasks status, tasks time and more.


.. _`Flower`: https://flower.readthedocs.io/en/latest/

Debugging
---------

| Life is not all tutorials and sometime things go wrong.
| Debugging exceptions is dependent of where the exception is originated at.

^^^^^^^^^^^^^^^^^^^^^^^^^
``load_model`` Exceptions
^^^^^^^^^^^^^^^^^^^^^^^^^

| If anything went wrong with the :ref:`pipeline_load_model` method, you will only able to see the traceback and exception on the logs.
| Specifically, the denzel service log is where the model is loaded - to view the logs use the :ref:`logs` command.


^^^^^^^^^^^^^^^^^^^^^^^^^^^
``verify_input`` Exceptions
^^^^^^^^^^^^^^^^^^^^^^^^^^^

| This method is executed in the API container. If anything goes wrong in this method you will get it as an immediate response to your ``/predict`` POST request.
| For example, lets say we make the same POST request as we did before, but we opt out one of the features in the data.
| Given the code we supplied ``verify_input`` we should get the following response

.. code-block:: json

    {
     "title": "Bad input format",
     "description": "For each example all of the features [['sepal-length', 'sepal-width', 'petal-length', 'petal-width']] must be present"
    }

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``process`` & ``predict`` Exceptions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

| :ref:`pipeline_process` and :ref:`pipeline_predict` both get executed on the denzel container. If anything goes wrong inside of them it will be most likely only visible when querying for task status.
| For example, if we would forget to import ``numpy as np`` even though it is in use in the :ref:`pipeline_process` method - we will get a ``"SUCCESS"`` response for our POST (because we passed the :ref:`pipeline_verify_input` function).
| But the task will fail after entering the :ref:`pipeline_process` method - to see the reason, we should query the :ref:`status_endpoint` and we would see the following:

.. code-block:: json

    {
     "status": "FAILURE",
     "result": {"args":["name 'np' is not defined"]}
    }

| In general it is best to keep an eye on the logs (``denzel logs``) and examine statuses (through the API or through the monitoring UI) of tasks when looking for bugs.


Deployment
----------

| Since denzel is fully containerized it should work on any machine as long as it has docker and Python3 installed.
| After completing all the necessary implementations for deployment covered in this tutorial it is best to check that the system can be launched from scratch.
| To do that, we should :ref:`shutdown` while purging all images, and relaunch the project - don't worry no code is being deleted during shutdown.
| Go to the main project directory and run the following:

.. code-block:: bash

    $ denzel shutdown --purge

    Stopping irisclassifier_denzel_1  ... done
    Stopping irisclassifier_monitor_1 ... done
    Stopping irisclassifier_api_1     ... done
    Stopping irisclassifier_redis_1   ... done
    Removing irisclassifier_denzel_1  ... done
    Removing irisclassifier_monitor_1 ... done
    Removing irisclassifier_api_1     ... done
    Removing irisclassifier_redis_1   ... done
    Removing network irisclassifier_default
    Removing image redis:3.2.11
    Removing image denzel
    Removing image denzel
    ERROR: Failed to remove image for service denzel: 404 Client Error: Not Found ("No such image: denzel:latest")
    Removing image denzel
    ERROR: Failed to remove image for service monitor: 404 Client Error: Not Found ("No such image: denzel:latest")

    $ denzel launch
    Creating network "irisclassifier_default" with the default driver
    Pulling redis (redis:3.2.11)...
    3.2.11: Pulling from library/redis
    .
    .

.. note::

    | The "ERROR: Failed to remove...." can be safely ignored. This is a result of the ``--purge`` flag that tells denzel to remove the denzel image.
    | Since the image is used by three different containers, it will successfully delete it on the first container but fail on the other two.


| By now denzel will rebuild everything from zero, but all the edited files and assets will still be present.
| After the relaunching is done check again that all endpoints are functioning as expected - just to make sure.
| If all is well your system is ready to be deployed wherever, on a local machine, a remote server or a docker supporting cloud service.
| Do deploy it elsewhere simply copy all the contents of the project directory to the desired destination, :ref:`install denzel <install>` and call ``denzel launch`` from within that directory.