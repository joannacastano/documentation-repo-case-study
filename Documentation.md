# Databases: Web Application Integration with a Cassandra NoSQL Database 
This case study provides an extensive overview of the procedure involved in successfully implementing and operating a web application seamlessly linked to a NoSQL database. This document provides a comprehensive description, including objectives, an architecture diagram, workflows, and a discussion of the solution's features. The methodology section offers a systematic procedure, leading the reader through the process of setting up a ready-made Kubernetes cluster, configuring a Git repository, creating Docker images, deploying and configuring Cassandra, as well as orchestrating Jenkins, Prometheus, and Splunk. The Runbook expressly covers probable issues, including 4xx and 5xx faults, and provides a Disaster Recovery Plan. 

Although having some knowledge of distributed systems and Kubernetes is advantageous, our goal is to enable readers with different degrees of expertise. A glossary is included in the final section of this document for convenient reference.
<br>
### Learning Objectives

[x] **Deployment and Integration:** Gain the ability to integrate and deploy a web application that utilizes a Cassandra NoSQL database on a dynamic Kubernetes framework.

[x] **Automation with Jenkins CI/CD:** Investigate the automation capabilities through the implementation of Jenkins CI/CD solutions, which optimize the deployment workflow.

[x] **Logging and Monitoring:** Operationalize logging and monitoring solutions utilizing Splunk and Prometheus, to ensure that the database and web application operate constantly.

[x] **Ops Simulation:** Deliberately introduce errors to the system to replicate operational accidents and routine maintenance scenarios.

<br>

## Methodology
### A. Accessing the Kubernetes Cluster

Kubernetes is a free and open-source tool for managing containerized applications. In the technical world, Kubernetes is called an "orchestration engine" as it orchestrates the deployment, scaling, and handling of the application/system, much like a maestro with its orchestra.
In this case study, we used Kubernetes as our orchestration platform for deploying our web app and database, along with services such as Jenkins, Prometheus, and Splunk.
As you go through this document, you will learn more about the services I mentioned above. For now, look at the steps below to access the Kubernetes cluster.

**DISCLAIMER**: We used MacOS when creating this solution. For other operating systems, ignore steps 1 and 2. Refer to the [Official Kubernetes Documentation](https://kubernetes.io/docs/tasks/tools/) for installing kubectl in [Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) or [Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).

**Step 1** Install HomeBrew
```
 $ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
**Step 2** Install the kubectl CLI in your local
```
$ brew install kubernetes-cli 
```
**Step 3** Check if the Installation is successful
```
$ kubectl version –client 
```
**Step 4** Export kubeconfig as root user
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

**Step 1** Create Jenkins Deployment using a yaml file.You may name it anything you prefer, but for readability we'll name this **jenkins-deployment.yaml**

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
**Step 2** Create Jenkins Service using yaml file. For readability, we'll name it **jenkins-service.yaml**.

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
**Step 3** Access the UI by looking for the host node's *external IP* and through the configured *nodePort* in the jenkins-service.yaml

```
$ kubectl get pods -o wide

$ kubectl get nodes –o wide
```
In our case, we can access the Jenkins web UI at *139.162.36.121:30979*.

**Step 4** You will be asked for the Administrator Password. To access this, inspect the jenkin pod's logs. You can see the full pod's name from running the *get pods* command in the previous step.
```
$ kubectl logs jenkins-<some string>
```
Look for this line in the logs:
```
Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
```
**Step 5** Install plugins. You will be redirected to the plugins page upon successful authentication. 
<br>

### C. Deploying Cassandra

Cassandra is a NoSQL database. Its query language is very similar to SQL, however its structure is entirely different. It is designed to handle large amounts of data with high reliability and scalability. This makes Cassandra an optimal database choice for scalable solutions.

<br>

### D. Creating the web app and Docker image

Since we had the freedom to choose any framework in creating the web application, we decided to use Flask, a Python-based web framework that can support Cassandra.

We created a simple To-do app that has authentication and can do CRUD processes.

**Step 1** Create a flask environment by following this [guide from Visual Studio Code](https://code.visualstudio.com/docs/python/tutorial-flask).

However for this solution, **app.py** should look like this:

```
from flask import Flask, render_template, request, redirect, url_for, session, flash, abort
from cassandra.cluster import Cluster
from passlib.hash import sha256_crypt
import uuid, time


app = Flask(__name__)
app.config.from_pyfile('config.py')
app.secret_key = app.config['SECRET_KEY']


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
        return render_template('500.html'), 500

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
        if is_logged_in():
            return redirect(url_for('todolist'))
        return redirect(url_for('login'))

    # Login route
    @app.route('/login', methods=['GET', 'POST'])
    def login():
        if request.method == 'POST':
            username = request.form['username']
            password_candidate = request.form['password']

            # Fetch user from database
            result = session_db.execute("SELECT id, password FROM users WHERE username = %s ALLOW FILTERING", (username,))
            user_data = result.one()

            if user_data and sha256_crypt.verify(password_candidate, user_data.password):
                session['username'] = username
                flash('You are now logged in', 'success')
                return redirect(url_for('todolist'))
            else:
                flash('Invalid login', 'danger')

        return render_template('login.html')

    # Logout route
    @app.route('/logout')
    def logout():
        session.clear()
        flash('You are now logged out', 'success')
        return redirect(url_for('login'))

    # Signup route
    @app.route('/signup', methods=['GET', 'POST'])
    def signup():
        if request.method == 'POST':
            username = request.form['username']
            password = sha256_crypt.encrypt(request.form['password'])

            # Check if username already exists
            result = session_db.execute("SELECT * FROM users WHERE username = %s ALLOW FILTERING", (username,))
            existing_user = result.one()

            if existing_user:
                flash('Username already exists', 'danger')
                return redirect(url_for('signup'))

            # Generate UUID in Python
            user_id = uuid.uuid4()

            # Insert new user into the database
            session_db.execute("INSERT INTO users (id, username, password) VALUES (%s, %s, %s)", (user_id, username, password))

            flash('You are now registered and can log in', 'success')
            return redirect(url_for('login'))

        return render_template('signup.html')


    # To-do list route
    @app.route('/todolist')
    def todolist():
        if not is_logged_in():
            return redirect(url_for('login'))

        username = session['username']

        # Fetch tasks for the logged-in user
        result = session_db.execute("SELECT * FROM todos WHERE username = %s ALLOW FILTERING", (username,))
        todos = result.all()

        return render_template('todolist.html', todos=todos)

    # Add task route
    @app.route('/add_task', methods=['POST'])
    def add_task():
        if not is_logged_in():
            return redirect(url_for('login'))

        task = request.form['task']
        username = session['username']

        # Insert new task into the database
        task_id = uuid.uuid4()
        session_db.execute("INSERT INTO todos (username, task_id, task) VALUES (%s, %s, %s)", (username, task_id, task))

        flash('Task added', 'success')
        return redirect(url_for('todolist'))

    # Delete task route
    @app.route('/delete_task/<string:task_id>', methods=['POST'])
    def delete_task(task_id):
        if not is_logged_in():
            return redirect(url_for('login'))

        # Delete task from the database
        session_db.execute("DELETE FROM todos WHERE task_id = %s", (uuid.UUID(task_id),))

        flash('Task deleted', 'success')
        return redirect(url_for('todolist'))
   
if __name__ == '__main__':
    app.run(debug=True)

```
**IMPORTANT NOTE:** The file **config.py** holds the SECRET_KEY for this app. You may create your own **config.py** and the SECRET_KEY can be any string. This file is hidden from you for data privacy.

**Step 2** Create a **templates** folder inside the project's folder and add the html files for the web pages.

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

**Step 3** Create a Dockerfile. This file must only be named **Dockerfile** without any file extensions. We need this in order to create a Docker image of our app.

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
**Step 4** Create the **requirements.txt** file. This file holds all the dependencies we need to create the Docker image.

```
flask==3.0.0
cassandra-driver==3.29.0
passlib==1.7.4
```

#### D.1 Building and deployment through Jenkins

<br>

### E. Deploying and Configuring Prometheus

Prometheus is an open-source alerting and monitoring toolkit that provides real-time application and system health metrics. It aids developers and administrators in understanding their app's behavior and performance by giving valuable insights to ensure the system is at its peak performance.

In this case study, Prometheus will be gathering various metrics from the web app, database, and the Kubernetes cluster itself.

<br>

### F. Deploying and Configuring Splunk

<br>

## Runbook

<br>

## Glossary

- Automation
- CICD
- Cluster
- Containers
- CRUD
- Dashboard
- Dependencies
- Framework
- Image
- Keyspace
- Nodes
- Pipelines
- Pods
- Plugins
- Repository
- Yaml
