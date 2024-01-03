# Databases: Web Application Integration with a Cassandra NoSQL Database 
```
Adviser: Melvin Bautista:
Authors: Louis Anthony Bernante, Joanna Lorraine Castaño, Lorenzo Lucin Jr., Joshua Villa
```
This case study provides an extensive overview of the procedures involved in successfully implementing and operating a web application seamlessly linked to a NoSQL database. This document provides a comprehensive description, including objectives, an architecture diagram, workflows, and a discussion of the solution's features. The methodology section offers a systematic procedure, leading the reader through the process of setting up a ready-made Kubernetes cluster, creating the web app and its Docker image, deploying and configuring the Cassandra database image, as well as orchestrating Jenkins, Prometheus, Grafana, and Splunk. The Runbook expressly covers probable issues and how to solve them. 

Although having some knowledge of distributed systems and Kubernetes is advantageous, our goal is to enable readers with different degrees of expertise. A glossary is included in the final section of this document for convenient reference.
<br>
### Learning Objectives

[x] **Deployment and Integration:** Gain the ability to integrate and deploy a web application that utilizes a Cassandra NoSQL database on a dynamic Kubernetes framework.

[x] **Automation with Jenkins CI/CD:** Investigate the automation capabilities through the implementation of Jenkins CI/CD solutions, which optimize the deployment workflow.

[x] **Logging and Monitoring:** Operationalize logging and monitoring solutions utilizing Splunk and Prometheus, to ensure that the database and web application operate constantly.

[x] **Ops Simulation:** Deliberately introduce errors to the system to replicate operational accidents and routine maintenance scenarios.

<br>

## System Architecture

![](https://drive.google.com/uc?export=view&id=11B4mXvH01VnQhilh8F4uHxbtfElXHrPM)



<br>

## Methodology

### A. Accessing the Kubernetes Cluster
Kubernetes is a free and open-source tool for managing containerized applications. In the technical world, Kubernetes is called an "orchestration engine" as it orchestrates the deployment, scaling, and handling of the application/system, much like a maestro with its orchestra.
In this case study, we used Kubernetes as our orchestration platform for deploying our web app and database, along with services such as Jenkins, Prometheus, Grafana, and Splunk.
As you go through this document, you will learn more about the services I mentioned above. For now, look at the steps below to access the Kubernetes cluster.

**DISCLAIMER**: We used MacOS when creating this solution. For other operating systems, ignore steps 1 and 2. Refer to the [Official Kubernetes Documentation](https://kubernetes.io/docs/tasks/tools/) for installing kubectl in [Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) or [Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).

- **Step 1** Install HomeBrew
```
 $ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
- **Step 2** Install the kubectl CLI in your local
```
$ brew install kubernetes-cli 
```
- **Step 3** Check if the Installation is successful
```
$ kubectl version –client 
```
- **Step 4** Export kubeconfig as root user
```
# export KUBECONFIG=~/a2t-pe-cs1-F-kubeconfig.yaml
```
NOTE: the file a2t-pe-cs1-F-kubeconfig.yaml is provided to us. For privacy concerns, we will not share this file's contents.<br>
**Step 5** Test if kubectl is working
```
$ kubectl get nodes
```
<br>

### B. Deploying and Configuring Jenkins
Jenkins is an open-source automation tool that enables continuous integration and delivery (CI/CD). It is designed to automate stages of software development, including processes such as testing, building, and deployment. Jenkins optimizes the development process by automating repetitive tasks, enabling a smooth and effective workflow for developers throughout various software development and delivery stages.

For this solution, we used Jenkins to build and deploy the web app, and as a tool for managing codebase changes and redeployment. Specifically, Jenkins is responsible for building the web app Docker image and deploying it to the Kubernetes cluster. If there are changes in the application's codebase in our GitHub repository, Jenkins will trigger workflows/pipelines to rebuild and redeploy the image.

- **Step 1** Create Jenkins Deployment using a yaml file.You may name it anything you prefer, but for readability we'll name this **jenkins-deployment.yaml**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk17
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        emptyDir: { }
```
Deploy this to the Kubernetes cluster by running the command below:
```
$ kubectl create –f jenkins-deployment.yaml
```
You may check if Jenkins was successfully deployed by running:
```
$ kubectl get deployments.apps
```
- **Step 2** Create Jenkins Service using yaml file. For readability, we'll name it **jenkins-service.yaml**.

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2023-12-30T08:29:35Z"
  name: jenkins
  namespace: default
  resourceVersion: "4246268"
  uid: 352e6f91-28ba-4bef-9091-808b5a01c0a7
spec:
  clusterIP: 10.128.25.41
  clusterIPs:
  - 10.128.25.41
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30979
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: jenkins
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Then, run the following command:
```
$ kubectl create –f jenkins-service.yaml
```
- **Step 3** Access the UI by looking for the host node's *external IP* and through the configured *nodePort* in the jenkins-service.yaml

```
$ kubectl get pods -o wide

$ kubectl get nodes –o wide
```
In our case, we can access the Jenkins web UI at *139.162.36.121:30979*.

- **Step 4** You will be asked for the Administrator Password. To access this, inspect the jenkin pod's logs. You can see the full pod's name from running the *get pods* command in the previous step.
```
$ kubectl logs jenkins-<some string>
```
Look for this line in the logs:
```
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
```
- **Step 5** Install plugins. You will be redirected to the plugins page upon successful authentication. 
<br>

### C. Deploying Cassandra

Cassandra is a NoSQL database. Its query language is very similar to SQL, however its structure is entirely different. It is designed to handle large amounts of data with high reliability and scalability. This makes Cassandra an optimal database choice for scalable solutions.

We'll be deploying Cassandra with a Statefulset. This method is taken from the [official Kubernetes Documentation](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/).

- **Step 1** Create a headless Service for Cassandra. We'll name this file as **cassandra-service.yaml**
The following Service is used for DNS lookups between Cassandra Pods and clients within your cluster:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cassandra
  name: cassandra
spec:
  clusterIP: None
  ports:
  - port: 9042
  selector:
    app: cassandra
```
- **Step 2** Build a Cassandra ring with three pods using a StatefulSet. Creating a Cassandra ring using a StatefulSet in Kubernetes involves configuring and deploying Cassandra instances in a scalable and persistent manner. 

  - **Step 2.a** Create the manifest file for the statefulset. We will name this file cassandra-statefulset.yaml
    ```
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
    name: cassandra
    labels:
        app: cassandra
    spec:
    serviceName: cassandra
    replicas: 3
    selector:
        matchLabels:
        app: cassandra
    template:
        metadata:
        labels:
            app: cassandra
        spec:
        terminationGracePeriodSeconds: 1800
        containers:
        - name: cassandra
            image: louis5566/cassandra:amd64
            imagePullPolicy: Always
            ports:
            - containerPort: 7000
            name: intra-node
            - containerPort: 7001
            name: tls-intra-node
            - containerPort: 7199
            name: jmx
            - containerPort: 9042
            name: cql
            resources:
            limits:
                cpu: "500m"
                memory: 1Gi
            requests:
                cpu: "500m"
                memory: 1Gi
            securityContext:
            capabilities:
                add:
                - IPC_LOCK
            lifecycle:
            preStop:
                exec:
                command: 
                - /bin/sh
                - -c
                - nodetool drain
            env:
            - name: MAX_HEAP_SIZE
                value: 512M
            - name: HEAP_NEWSIZE
                value: 100M
            - name: CASSANDRA_SEEDS
                value: "cassandra-0.cassandra.default.svc.cluster.local"
            - name: CASSANDRA_CLUSTER_NAME
                value: "K8Demo"
            - name: CASSANDRA_DC
                value: "DC1-K8Demo"
            - name: CASSANDRA_RACK
                value: "Rack1-K8Demo"
            - name: POD_IP
                valueFrom:
                fieldRef:
                    fieldPath: status.podIP
            readinessProbe:
            exec:
                command:
                - /bin/bash
                - -c
                - /ready-probe.sh
            initialDelaySeconds: 15
            timeoutSeconds: 5
            volumeMounts:
            - name: cassandra-data
            mountPath: /cassandra_data
    volumeClaimTemplates:
    - metadata:
        name: cassandra-data
        spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: linode-block-storage
        resources:
            requests:
            storage: 10Gi
    ---
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
    name: linode-block-storage
    annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
        {"allowVolumeExpansion":true,"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{"lke.linode.com/caplke-version":"v1.28.3-001"},"name":"linode-block-storage"},"provisioner":"linodebs.csi.linode.com"}
        lke.linode.com/caplke-version: v1.28.3-001
    provisioner: linodebs.csi.linode.com
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    allowVolumeExpansion: true
    ```
    **NOTE:** The fields *spec.spec.image*, *metadata.name* (in StorageClass), and *provisioner* (in StorageClass) are modified according to the cloud environment provisioner utilized in our cluster. Specifically, we're utilizing a custom Cassandra image on Kubernetes in our scenario with the provisioner linodebs.csi.linode.com and a storage class of linode-block-storage. To determine the cloud provider, examine the configuration and metadata available to the cluster. More information regarding the ways to identify the cloud environment can be found in section C.1.

    Below is the Dockerfile of our custom Cassandra image. A custom image was made so we could install and have a python package in our Cassandra deployment. We'll need this for running Cassandra Query Language in its shell.
    ```
    #!/bin/bash
    # Use the official Cassandra base image for AMD64
    FROM cassandra:latest

    WORKDIR ./

    USER root
    # Expose Cassandra ports
    EXPOSE 7000 7001 7199 9042 9160

    # Custom configurations or additional setup can be added here if needed

    RUN apt-get update && \
    apt-get install -y python3 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

    USER cassandra
    COPY ready-probe.sh /ready-probe.sh

    # Start Cassandra when the container starts
    CMD ["cassandra", "-f"]
    ```

  - **Step 2.b** Verify the integrity of the Cassandra StatefulSet.
    ```
    Get the list of active Cassandra statefulset
    $ kubectl get statefulset

    Display the pods and observe the creation status
    $ kubectl get pods -l="app=cassandra"

    Execute the Cassandra node tool within the initial Pod to show case the status of the ring.
    $ kubectl exec -it cassandra-0 -- nodetool status
    ```
- **Step 3** (Optional) Scale the Cassandra StatefulSet
In Kubernetes, scaling a StatefulSet entails adding or decreasing the number of replicas (pods) for the StatefulSet. Scaling the StatefulSet in the context of Cassandra operating in Kubernetes allows you to add or remove Cassandra nodes to adjust to changes in workload, performance requirements, or other operational demands.
```
$ kubectl scale statefulsets <stateful-set-name> --replicas=<new-replicas>
```
- **Step 4** Creating a Unified Cassandra Cluster
The presence of Cassandra pods in the same cluster is critical for reaping the benefits of distributed databases. It offers scalability, fault tolerance, and efficient data dissemination, ensuring Cassandra's best performance in a distributed and dynamic context.

To connect Cassandra nodes in a cluster, update the cassandra.yaml configuration file with the same cluster name and seed nodes across all instances. Inside the cassandra pod, locate the yaml file at /etc/cassandra/cassandra.yaml

![](https://drive.google.com/uc?export=view&id=1h2RG4JZUR4BSdyvOZ-L2Ul0Y03z5-Fmi)

Scroll down until you locate the *seed_provider* section and edit the seeds with the actual IP address assigned to each cassandra pods.

![](https://drive.google.com/uc?export=view&id=1HEOaNA2C_E93m9EWCajzIgyPn63rIVPW)

Lookup the IP address of the cassandra prods:

```
$ kubectl get pods -o wide
```

- **Step 5** Configure the cassandra.yaml file inside the pod.
   - **Step 5.a** Copy cassandra.yaml from the pod to your local machine for it to be edit
     ```
     $ kubectl cp <your-cassandra-pod-name>:/etc/cassandra/cassandra.yaml ./cassandra.yaml
     ```
   - **Step 5.b** Edit the cassandra.yaml file on your local machine using your preferred text editor
   - **Step 5.c** Move the edited file back to the pod
     ```
     $ kubectl cp ./cassandra.yaml <your-cassandra-pod-name>:/etc/cassandra/cassandra.yaml
     ```
   - **Step 5.d** Verify Cluster Status. This command will display information about the nodes in the Cassandra cluster. Verify that all nodes are in the "UN" (Up and Normal) state.
     ```
     $ kubectl exec -it -- /bin/bash Nodetool status
     ```
   - **Step 5.e** Repeat this process for the other Cassandra pods to ensure       consistency. If you see all nodes in the "UN" state, it indicates that your Cassandra pods are part of the same cluster.

#### C.1 Determining the cloud provider by examining the configuration and metadata available to the cluster.

- **Step 1** Check Kubernetes ProviderID: Each node in a Kubernetes cluster has a ProviderID associated with it. You can check the provider ID of a node to identify the underlying cloud provider by using this command:
```
$ kubectl get nodes -o custom-columns=NAME:.metadata.name,PROVIDER:.spec.provider ID
```
![](https://drive.google.com/uc?export=view&id=16vz2r9fgl5whCvDxN3GlPOjGJVLDfWoa)

The output should look similar to this and look for the name of the provider, in our case it is Linode.

- **Step 2** CheckInstalled CSI Driver:
Verify that the Linode CSI driver is installed in your cluster. You can check the installed CSI drivers using the following command:
```
$ kubectl get csidrivers
```
- **Step 3** Inspect Storage Classes: Examine the Storage Classes available in your cluster to see if there is a Linode-specific storage class configured. You can use the following command:
```
$ kubectl get storageclass
```
Check if there is a storage class with the provisioner set to linodebs.csi.linode.com. This storage class is typically used for dynamic provisioning of Linode block storage volumes.
- **Step 4** Create the Cassandra StatefulSet from the cassandra-statefulset.yaml file
```
$ kubectl apply -f cassandra-statefulset.yaml
```
Since the statefulset yaml file is in our local directory, we can proceed to deploy this directly using the file.
<br>

### D. Creating the web app and Docker image

Since we had the freedom to choose any framework in creating the web application, we decided to use Flask, a Python-based web framework that can support Cassandra.

We created a simple To-do app that has authentication and can do CREATE, READ, DELETE processes.

- **Step 1** Create a flask environment by following this [guide from Visual Studio Code](https://code.visualstudio.com/docs/python/tutorial-flask).

However for this solution, **app.py** should look like this:
```
from flask import Flask, render_template, request, redirect, url_for, session, flash, make_response
from cassandra.cluster import Cluster
from passlib.hash import sha256_crypt

from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

import uuid, time

app = Flask(__name__)
app.config.from_pyfile('config.py')
app.secret_key = app.config['SECRET_KEY']

tracer_provider = TracerProvider()
trace.set_tracer_provider(tracer_provider)

exporter = OTLPSpanExporter(
   endpoint="http://139.162.36.121:30056", 
   headers={"Authorization": "Bearer ba539994-032a-47e3-bd86-38894400365d"},
)

tracer_provider.add_span_processor(BatchSpanProcessor(exporter))

tracer = trace.get_tracer_provider().get_tracer(__name__)

MAX_RETRIES = 5
RETRY_INTERVAL = 5 # seconds

cluster = None
session_db = None

# Retry mechanism
for _ in range(MAX_RETRIES):
    try:
        cluster = Cluster(['cassandra'], port=9042, connect_timeout=10)
        session_db = cluster.connect()
        break # if connection is successful, break the loop
    except Exception as e:
        print(f"Failed to connect to Cassandra: {e}")
        time.sleep(RETRY_INTERVAL) # wait before next attempt

if session_db is None:
    @app.errorhandler(500)
    def internal_server_error(e):
        with tracer.start_as_current_span("500") as span:
            resp = make_response(render_template('500.html'))
            span.set_attribute("http.status_code", 500)
        return resp

else:
    session_db.set_keyspace('todoapp')

    # Create user table if not exists
    session_db.execute(
    """
    CREATE TABLE IF NOT EXISTS users (
        id UUID PRIMARY KEY,
        username text,
        password text
    )
    """
    )

    # Create to-do list table if not exists
    session_db.execute(
        """
        CREATE TABLE IF NOT EXISTS todos (
            username text,
            task_id UUID PRIMARY KEY,
            task text
        )
        """
    )

    # Function to check if the user is loggedin
    def is_logged_in():
        return 'username' in session

    # Home route
    @app.route('/')
    def home():
        with tracer.start_as_current_span("home") as span:
            if is_logged_in():
                resp = make_response(redirect(url_for('todolist')))
            else:
                resp = make_response(redirect(url_for('login')))
            if span.is_recording():
                span.set_attribute("http.status_code", resp.status_code)
        return resp

    # Login route
    @app.route('/login', methods=['GET', 'POST'])
    def login():
        with tracer.start_as_current_span("login") as span:
            if request.method == 'POST':
                username = request.form['username']
                password_candidate = request.form['password']

                # Fetch user from database
                result = session_db.execute("SELECT id, password FROM users WHERE username = %s ALLOW FILTERING", (username,))
                user_data = result.one()

                if user_data and sha256_crypt.verify(password_candidate, user_data.password):
                    session['username'] = username
                    flash('You are now logged in', 'success')
                    resp = make_response(redirect(url_for('todolist')))
                    if span.is_recording():
                        span.set_attribute("http.status_code", 200)
                else:
                    flash('Invalid login', 'danger')
                    resp = make_response(render_template('login.html'), 401)
                    if span.is_recording():
                        span.set_attribute("http.status_code", 200)
            else:
                resp = make_response(render_template('login.html'))
                if span.is_recording():
                    span.set_attribute("http.status_code", resp.status_code)
        return resp

    # Logout route
    @app.route('/logout')
    def logout():
        with tracer.start_as_current_span("logout") as span:
            session.clear()
            flash('You are now logged out', 'success')
            resp = make_response(redirect(url_for('login')))
            if span.is_recording():
                span.set_attribute("http.status_code", 200)
        return resp

    # Signup route
    @app.route('/signup', methods=['GET', 'POST'])
    def signup():
        with tracer.start_as_current_span("signup") as span:
            if request.method == 'POST':
                username = request.form['username']
                password = sha256_crypt.encrypt(request.form['password'])

                # Check if username already exists
                result = session_db.execute("SELECT * FROM users WHERE username = %s ALLOW FILTERING", (username,))
                existing_user = result.one()

                if existing_user:
                    flash('Username already exists', 'danger')
                    resp = make_response(redirect(url_for('signup')))
                    if span.is_recording():
                        span.set_attribute("http.status_code", 409)
                    return resp

                # Generate UUID in Python
                user_id = uuid.uuid4()

                # Insert new user into the database
                session_db.execute("INSERT INTO users (id, username, password) VALUES (%s, %s, %s)", (user_id, username, password))

                flash('You are now registered and can log in', 'success')
                resp = make_response(redirect(url_for('login')))
                if span.is_recording():
                    span.set_attribute("http.status_code", 201)
                return resp

            resp = make_response(render_template('signup.html'))
            if span.is_recording():
                span.set_attribute("http.status_code", 200)
            return resp


@app.route('/todolist')
def todolist():
    with tracer.start_as_current_span("todolist") as span:
        if not is_logged_in():
            resp = make_response(redirect(url_for('login')))
            if span.is_recording():
                span.set_attribute("http.status_code", 302)
            return resp

        username = session['username']

        # Fetch tasks for the logged-in user
        result = session_db.execute("SELECT * FROM todos WHERE username = %s ALLOW FILTERING", (username,))
        todos = result.all()

        resp = make_response(render_template('todolist.html', todos=todos))
        if span.is_recording():
            span.set_attribute("http.status_code", 200)
        return resp

# Create/Add task route
@app.route('/add_task', methods=['POST'])
def add_task():
    with tracer.start_as_current_span("add_task") as span:
        if not is_logged_in():
            resp = make_response(redirect(url_for('login')))
            if span.is_recording():
                span.set_attribute("http.status_code", 302)
            return resp

        task = request.form['task']
        username = session['username']

        # Insert new task into the database
        task_id = uuid.uuid4()
        session_db.execute("INSERT INTO todos (username, task_id, task) VALUES (%s, %s, %s)", (username, task_id, task))

        flash('Task added', 'success')
        resp = make_response(redirect(url_for('todolist')))
        if span.is_recording():
            span.set_attribute("http.status_code", 201)
        return resp
    
# Delete task route
@app.route('/delete_task/<string:task_id>', methods=['POST'])
def delete_task(task_id):
    with tracer.start_as_current_span("delete_task") as span:
        if not is_logged_in():
            resp = make_response(redirect(url_for('login')))
            if span.is_recording():
                span.set_attribute("http.status_code", 302)
            return resp

        # Delete task from the database
        session_db.execute("DELETE FROM todos WHERE task_id = %s", (uuid.UUID(task_id),))

        flash('Task deleted', 'success')
        resp = make_response(redirect(url_for('todolist')))
        if span.is_recording():
            span.set_attribute("http.status_code", 202)
        return resp

   
if __name__ == '__main__':
    app.run(debug=True)
```
**IMPORTANT NOTE:** The file **config.py** holds the SECRET_KEY for this app. You may create your own **config.py** and the SECRET_KEY can be any string. This file is hidden from you for data privacy.

- **Step 2** Create a **templates** folder inside the project's folder and add the html files for the web pages.

**login.html**
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login</title>
</head>
<body>
    <h2>Login</h2>
    <form method="POST" action="{{ url_for('login') }}">
        <label for="username">Username:</label>
        <input type="text" id="username" name="username" required>
        <br>
        <label for="password">Password:</label>
        <input type="password" id="password" name="password" required>
        <br>
        <button type="submit">Login</button>
    </form>
    <p><a href="{{ url_for('signup') }}">New here? Signup instead!</a></p>
</body>
</html>
```

**signup.html**
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sign Up</title>
</head>
<body>
    <h2>Sign Up</h2>
    <form method="POST" action="{{ url_for('signup') }}">
        <label for="username">Username:</label>
        <input type="text" id="username" name="username" required>
        <br>
        <label for="password">Password:</label>
        <input type="password" id="password" name="password" required>
        <br>
        <button type="submit">Sign Up</button>
    </form>
    <p><a href="{{ url_for('login') }}">Already signed up? Login instead.</a></p>
</body>
</html>
```
**todolist.html**
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>To-Do List</title>
</head>
<body>
    <h2>Welcome, {{ session['username'] }}!</h2>
    <h3>To-Do List</h3>
    <form method="POST" action="{{ url_for('add_task') }}">
        <label for="task">New Task:</label>
        <input type="text" id="task" name="task" required>
        <button type="submit">Add Task</button>
    </form>
    <ul>
        {% for todo in todos %}
            <li>
                {{ todo.task }}
                <form method="POST" action="{{ url_for('delete_task', task_id=todo.task_id) }}" style="display: inline;">
                    <button type="submit">Delete</button>
                </form>
            </li>
        {% endfor %}
    </ul>
    <br>
    <a href="{{ url_for('logout') }}">Logout</a>
</body>
</html>
```

**500.html**
```
<!DOCTYPE html>
<html>
<head>
  <title>Error 500</title>
</head>
<body>
  <h1>Error 500: Internal Server Error</h1>
  <p></p>
</body>
</html>
```

- **Step 3** Create a Dockerfile. This file must only be named **Dockerfile** without any file extensions. We need this in order to create a Docker image of our app.

```
FROM python:3.11

WORKDIR /case_study

# Copy only the requirements.txt to the working directory
COPY requirements.txt .

# Install dependencies
RUN apt-get update && \
  apt-get install -y gcc python3-dev libev-dev

ENV CASS_DRIVER_NO_CYTHON=1

RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

# Copy the entire project to the working directory
COPY . .

# Set environment variables
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
ENV FLASK_RUN_PORT=5000

# Expose the port the app runs on
EXPOSE 5000

# Command to run the application
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
```
- **Step 4** Create the **requirements.txt** file. This file holds all the dependencies we need to create the Docker image.

```
flask==3.0.0
cassandra-driver==3.29.0
passlib==1.7.4
```

#### D.1 Build and deployment through Jenkins

<br>

### E. Deploying and Configuring Prometheus

Prometheus is an open-source alerting and monitoring toolkit that provides real-time application and system health metrics. It aids developers and administrators in understanding their app's behavior and performance by giving valuable insights to ensure the system is at its peak performance.

In this case study, Prometheus will be gathering various metrics from the web app, database, and the Kubernetes cluster itself.

The following steps are taken from the [Grafana Document for Installing the Prometheus Operator](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/configure-infrastructure-manually/prometheus/prometheus-operator/#create-a-prometheus-service)
<br>
- **Step 1** Install Splunk Operator using the **bundle.yaml** file from the Prometheus Operator GitHub repository.
```
$ kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
```
Verify if the Prometheus Operator installed successfully. Look for the *prometheus-operator* under the name field.
```
$ kubectl get deploy
```
- **Step 2** Configure Prometheus RBAC Permissions.
  - **2.a** Create a directory for your Kubernetes manifests and navigate to it.
```
$ mkdir operator_k8s
$ cd operator_k8s
```
  - **2.b** Create a manifest file called **prom_rbac.yaml** inside your created directory
  - **2.c** Create the object
- **Step 3** Deploy Prometheus
  - **3.a** Create a prometheus.yaml manifest file
  - **3.b** Deploy the manifest
  - **3.c** Verify the deployment
  - **3.d** Check underlying pods
- **Step 4** Create a Prometheus service
<br>

### F. Deploying and Configuring Splunk

**To deploy Splunk Enterprise by using Splunk Operator.** <br>
- **Step 1** is to install Splunk Operator to the Kubernetes cluster using this command. <br>
```
$ kubectl apply -f https://github.com/splunk/splunk-operator/releases/download/2.4.0/splunk-operator-cluster.yaml --server-side  --force-conflicts
```
 
- **Step 1.a** Use kubectl get pods to confirm if it’s already running. <br> <br>
   ![Image description](https://drive.google.com/uc?export=view&id=1-ll_9t-68Cbj_srKmrhY61XzPlVAZq0A)
    - **Step 1.b** To create an instance, you create this yaml file and run this command.
 
    ```
    apiVersion: enterprise.splunk.com/v4
    kind: Standalone
    metadata:
        name: s1
        finalizers:
        - enterprise.splunk.com/delete-pvc
    ```
    
    ```
    $ kubectl apply -n splunk-operator -f enterprise.yaml
    ```
    - **Step 1.c** Next is to determine the port used to access the web UI. <br> <br>
    ![Image description](https://drive.google.com/uc?export=view&id=1o5CYDa8UgDts5X2ouI18l7S2yZGWFu6n)
    - **Step 1.d** Get the credentials with default username admin use this command and decode the password.
    ![Image description](https://drive.google.com/uc?export=view&id=13JZhrBkFQSJ5n04h1I07Rpz4JUdyzZgR)
 
        ```
        $ echo cVZEamFVVDROZTFFRkg3YWVRYmcxRzk1 | base64 –d
        ```
 
    ![Image description](https://drive.google.com/uc?export=view&id=1dlfOFozW2L9mHghQ4NgVDUPlWsBmEFi8)
 
    ![Image description](https://drive.google.com/uc?export=view&id=1fQ75bqI84YPgjTKg8g7YooCQuuiwvQ4e)
 
This article provides a [Installation for Splunk Operator](https://splunk.github.io/splunk-operator/#installing-the-splunk-operator) for quick reference. You can also watch this [Video](https://www.youtube.com/watch?v=Qlm0VPackh8&t=625s).
 
- **Step2** Configure HTTP Event Collector on Splunk Enterprise
    
    - **Step 2.b** Enable HTTP Event Collector on Splunk Enterprise
 
        **NOTE** Before you can use Event Collector to receive events through HTTP, you must enable it. For Splunk Enterprise, enable HEC through the Global Settings dialog box.
 
        - Click Settings > Data Inputs.
 
        - Click HTTP Event Collector.
 
        - Click Global Settings.
 
        - In the All-Tokens toggle button, select Enabled.
 
        - **(Optional)** Choose a Default Source Type for all HEC tokens. You can also type in the name of the source type in the text field above the drop-down list box before choosing the source type.
 
        - **(Optional)** Choose a Default Index for all HEC tokens.
 
        - **(Optional)** Choose a Default Output Group for all HEC tokens.
 
        - **(Optional)** To use a deployment server to handle configurations for HEC tokens, click the Use Deployment Server check box.
 
        - **(Optional)** To have HEC listen and communicate over HTTPS rather than HTTP, click the Enable SSL checkbox.
 
        - **(Optional)** Enter a number in the HTTP Port Number field for HEC to listen on.
 
        - **Click Save.**
 
    - **Step 2.b**
 
        **NOTE:** To use HEC, you must configure at least one token.
 
        - Click Settings
 
        ![Image description](https://drive.google.com/uc?export=view&id=1tp42Cn9yoXyRcynd7CshTdsa5sX1acOF)
 
        - Click Monitor
 
        ![Image description](https://drive.google.com/uc?export=view&id=1LJ6I_atDJ7idaUoxS8rodyHONpT7tUh-)
 
        - Click HTTP Event Collector, In the Name field, enter a name for the token.
 
        ![Image description](https://drive.google.com/uc?export=view&id=1wa-yh3ifot_-jVd30xBQ0EB6sqa6GrTa)
 
        - Confirm that all settings for the endpoint are what you want. If all settings are what you want, click Submit. Otherwise, click < to make changes.
 
        ![Image description](https://drive.google.com/uc?export=view&id=1JyGbTL2K2Q-spRb4V5ClFMn2couVTpDR)
        
         **(Optional)** Copy the token value that Splunk Web displays and paste it into another document for reference later.
 
        ![Image description](https://drive.google.com/uc?export=view&id=1AM6r5g7a8pyGhRWAw0RycSLgIzOco6Q2)
 
For easy reference, this article offers a [HTTP Event Collector in Splunk Enterprise](https://docs.splunk.com/Documentation/Splunk/8.2.0/Data/UsetheHTTPEventCollector#Configure_HTTP_Event_Collector_on_Splunk_Enterprise).
 
- **Step 3** Next is to set up Splunk OTEL Collector.
    - **Step 3.a** Use this command
    
    ```
    helm repo add splunk-otel-collector-chart https://signalfx.github.io/splunk-otel-collector-chart
    ```
    ```
    wget https://github.com/signalfx/splunk-otel-collector-chart/releases/download/splunk-otel-collector-0.86.1/splunk-otel-collector-0.86.1.tgz
    ```
    ```
    tar -xzvf splunk-otel-collector-0.86.1.tgz
    ```
Next is to set up values.yaml file refer to this [Documentation](https://docs.rafay.co/recipes/logging/splunk-otel-collector/splunk/#assumptions)
 
```
################################################################################
# clusterName is a REQUIRED. It can be set to an arbitrary value that identifies
# your K8s cluster. The value will be associated with every trace, metric and
# log as "k8s.cluster.name" attribute.
################################################################################
 
clusterName: "lke146616"
 
################################################################################
# Splunk Cloud / Splunk Enterprise configuration.
################################################################################
 
# Specify `endpoint` and `token` in order to send data to Splunk Cloud or Splunk
# Enterprise.
splunkPlatform:
  # Required for Splunk Enterprise/Cloud. URL to a Splunk instance to send data
  # to. e.g. "http://X.X.X.X:8088/services/collector/event". Setting this parameter
  # enables Splunk Platform as a destination. Use the /services/collector/event
  # endpoint for proper extraction of fields.
  endpoint: "http://139.162.36.121:30056"
  # Required for Splunk Enterprise/Cloud (if `endpoint` is specified). Splunk
  # Alternatively the token can be provided as a secret.
  # Refer to https://github.com/signalfx/splunk-otel-collector-chart/blob/main/docs/advanced-configuration.md#provide-tokens-as-a-secret
  # HTTP Event Collector token.
  token: "ba539994-032a-47e3-bd86-38894400365d"
 
  # Name of the Splunk event type index targeted. Required when ingesting logs to Splunk Platform.
  index: "kube_cluster"
  # Name of the Splunk metric type index targeted. Required when ingesting metrics to Splunk Platform.
  metricsIndex: "kube_cluster_metrics"
  # Name of the Splunk event type index targeted. Required when ingesting traces to Splunk Platform.
  tracesIndex: ""
  # Optional. Default value for `source` field.
  source: "kubernetes"
  # Optional. Default value for `sourcetype` field. For container logs, it will
  # be container name.
  sourcetype: ""
  # Maximum HTTP connections to use simultaneously when sending data.
  maxConnections: 200
  # Whether to disable gzip compression over HTTP. Defaults to true.
  disableCompression: true
  # HTTP timeout when sending data. Defaults to 10s.
  timeout: 10s
  # Idle connection timeout. defaults to 10s
  idleConnTimeout: 10s
  # Whether to skip checking the certificate of the HEC endpoint when sending
  # data over HTTPS.
  insecureSkipVerify: true
  # The PEM-format CA certificate for this client.
  # Alternatively the clientCert, clientKey and caFile can be provided as a secret.
  # Refer to https://github.com/signalfx/splunk-otel-collector-chart/blob/main/docs/advanced-configuration.md#provide-tokens-as-a-secret
  # NOTE: The content of the certificate itself should be used here, not the
  #       file path. The certificate will be stored as a secret in kubernetes.
  clientCert: ""
  # The private key for this client.
  # NOTE: The content of the key itself should be used here, not the file path.
  #       The key will be stored as a secret in kubernetes.
  clientKey: ""
  # The PEM-format CA certificate file.
  # NOTE: The content of the file itself should be used here, not the file path.
  #       The file will be stored as a secret in kubernetes.
  caFile: ""
 
  # Options to disable or enable particular telemetry data types that will be sent to
  # Splunk Platform. Only logs collection is enabled by default.
  logsEnabled: true
  # If you enable metrics collection, make sure that `metricsIndex` is provided as well.
  metricsEnabled: true
 
################################################################################
# Logs collection engine:
# - `fluentd`: deploy a fluentd sidecar that will collect logs and send them to
#   otel-collector agent for further processing.
# - `otel`: utilize native OpenTelemetry log collection.
#
# `fluentd` will be deprecated soon, so it's recommended to use `otel` instead.
################################################################################
 
logsEngine: otel
 
################################################################################
# Cloud provider, if any, the collector is running on. Leave empty for none/other.
# - "aws" (Amazon Web Services)
# - "gcp" (Google Cloud Platform)
# - "azure" (Microsoft Azure)
################################################################################
 
cloudProvider: ""
 
################################################################################
# Kubernetes distribution being run. Leave empty for other.
# - "aks" (Azure Kubernetes Service)
# - "eks" (Amazon Elastic Kubernetes Service)
# - "eks/fargate" (Amazon Elastic Kubernetes Service with Fargate profiles )
# - "gke" (Google Kubernetes Engine / Standard mode)
# - "gke/autopilot" (Google Kubernetes Engine / Autopilot mode)
# - "openshift" (RedHat OpenShift)
################################################################################
 
distribution: ""
 
```
 
```
helm -n splunk-operator install my-splunk-otel-collector -f /Users/acadb517/otel/splunk-otel-collector/values.yaml splunk-otel-collector-chart/splunk-otel-collector
```
 
![Image description](https://drive.google.com/uc?export=view&id=1DK95DP9yJH_i2Jlq37Ycd2dcXx5hKogE)
 
![Image description](https://drive.google.com/uc?export=view&id=1IeJq3mvbOhCEsfAwKyB44_wkSk7d_JO7)
 
<br>

### G. Building Dashboards with Grafana

<br>

## Runbook

<br>

## Glossary

- **Automation**: A method that uses technology to do tasks so that there is less human intervention.
  
- **Cassandra ring**: Describes a resilient and distributed group of Cassandra nodes capable of handling faults.
  
- **CICD**: A software development practice that involves automating the process in the software development cycle.
  
- **Containers**: An isolated and running image of an application.
  
- **Dependencies**: A list of libraries or packages that are needed for an application to run properly.
  
- **Dockerfile**: A list of instructions and commands in order to build an image of an application.
  
- **Headless service** (Kubernetes): A headless service in Kubernetes is a service with a service IP but without load balancing. Each pod in the StatefulSet gets its DNS record, allowing direct communication between Cassandra nodes.
  
- **Image**: In Docker, an image contains all the things you need to run an application, including the application itself. This ensures that application's deployed using this image is standardized.
  
- **Keyspace** (Cassandra): Its a storage unit that holds Tables. It separates a collection of Tables, allowing for organization and managaement of various data.
  
- **Load Balancer**: Distributes network traffic across different servers or computing resources. 
  
- **Logs**: A collection of captured events, activities, and messages made by an application or a service.
  
- **Metrics**: Data that can quantitatively describe a system's health and performance.
  
- **Node**: In Kubernetes, a node is machine that is a part of a cluster.
  
- **NodePort**: A service in Kubernetes that exposes an application to the public.
  
- **Pipelines**: A set of automated process for CICD.
  
- **Pods**: Represents an instance of a running instance. Holds the container of an application.
  
- **Plugin**: A software component that can either extend or enhance a system or application's functionality.
  
- **Repository**: Contains a collection of files and folders for specific projects. A famous repository platform is GitHub.
  
- **Service** (Kubernetes): Service in kubernetes facilitates the communication between different parts of the application and others outside of it and assists us in establishing connections between apps and other apps or users.
  
- **StatefulSet**: A Kubernetes controller that ensures the order and uniqueness of pods, particularly in the Cassandra context. This guarantees stable network identities and persistent storage for individual nodes in the cluster.

