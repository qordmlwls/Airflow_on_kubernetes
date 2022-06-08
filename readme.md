# Airflow on Kubernetes
- This project is for testing Airflow on Kubernetes, KubernetesPodOperator

## Create docker image and push 

```
cd scripts/docker
docker build -t airflow .
```

> Rename docker image and push your registry(DockerHub)


```
docker push [your docker image name]
```

## Change docker image and tag

```
cd scripts/kube
vi deploy.sh
```


> change AIRFLOW_IMAGE and AIRFLOW_TAG into your docker image and tag

```
cd scripts/kube/build 
vi airflow.yaml
```

> Change all image into your docker image and tag 

## Deploy in kubernetes 

```
cd scripts/
./kube/deploy.sh -d persistent_mode
```

> Make sure kubernetes should be run before deployment 

## Check pods and service

```
kubectl get svc
kubectl get pods
```

## KubernetesPodOperator - run linear regression 

> First, you should make your own docker image and make train.py in the image

> train.py example

```
from sklearn import datasets
import pandas as pd
from sklearn import linear_model
boston_house_prices = datasets.load_boston()
data = pd.DataFrame(boston_house_prices.data)
data.columns = boston_house_prices.feature_names
data['Price'] = boston_house_prices.target

linear_regression = linear_model.LinearRegression()
linear_regression.fit(X=pd.DataFrame(data['RM']), y = data['Price'])
prediction = linear_regression.predict(X=pd.DataFrame(data['RM']))
print('a value: ' , linear_regression.intercept_)
print('b value: ' , linear_regression.coef_)
```

> After that, push your docker image to DockerHub

```
docker push [your train example docker image]
```

> Then, enter your airflow webserver pod

```
kubectl exec -it [your airflow pod name] -c webserver -- /bin/bash
cd root/airflow/dags
vi example_kubernetes_operator.py
```

> Then change example_kubernetes_operator.py into below code

```
# -*- coding: utf-8 -*-
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
"""
This is an example dag for using the KubernetesPodOperator.
"""
from airflow.utils.dates import days_ago
from airflow.utils.log.logging_mixin import LoggingMixin
from airflow.models import DAG
from kubernetes.client import models as k8s
#from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import ( KubernetesPodOperator )
log = LoggingMixin().log

try:
    # Kubernetes is optional, so not available in vanilla Airflow
    # pip install 'apache-airflow[kubernetes]'
    from airflow.contrib.operators.kubernetes_pod_operator import KubernetesPodOperator

    default_args = {
        'owner': 'Airflow',
        'start_date': days_ago(2)
    }

    with DAG(
        dag_id='example_kubernetes_operator',
        default_args=default_args,
        schedule_interval=None
    ) as dag:

        tolerations = [
            {
                'key': "key",
                'operator': 'Equal',
                'value': 'value'
            }
        ]

        k = KubernetesPodOperator(
            namespace='airflow-example',
            image="your train example docker image",
            cmds=["python3", "train.py"],
            # arguments=["echo", "10"],
            # labels={"foo": "bar"},
            name="airflow-test-pod",
            in_cluster=True,
            task_id="task",
            get_logs=True,
            is_delete_operator_pod=False,
            tolerations=tolerations,
            
        )

except ImportError as e:
    log.warning("Could not import KubernetesPodOperator: " + str(e))
    log.warning("Install kubernetes dependencies with: "
                "    pip install 'apache-airflow[kubernetes]'")
```

> Change KubernetesPodOperator() image into your train example docker image

> Finally, run example_kubernetes_operator.py on airflow webserver then you can find log

![Untitled (12)](https://user-images.githubusercontent.com/43153661/172519530-ca8cc4a6-b965-492f-b3b8-b73048681df3.png)