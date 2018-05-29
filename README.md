# Installing Quay on OpenShift

The instructions here have been tested as of 05/29/2018.

**Note** that you will need help from the OpenShift Cluster Administrator to perform some steps.

#### Step 1: Download the configuration files

Download the following configuration files. 

```
wget https://coreos.com/quay-enterprise/docs/latest/tectonic/files/quay-enterprise-config-secret.yml
wget https://coreos.com/quay-enterprise/docs/latest/tectonic/files/quay-enterprise-redis.yml
wget https://coreos.com/quay-enterprise/docs/latest/tectonic/files/quay-enterprise-app-rc.yml
wget https://coreos.com/quay-enterprise/docs/latest/tectonic/files/quay-enterprise-service-loadbalancer.yml
``` 

#### Step 2: Create a project to install Quay 

Run the following command to create the project

```
$ oc new-project quay-enterprise
```


#### Step 3: Cluster administrator to elevate privileges

**Cluster Administrator** should download the following two files to create a role and a role bining

```
wget https://coreos.com/quay-enterprise/docs/latest/tectonic/files/quay-servicetoken-role-k8s1-6.yaml
wget https://coreos.com/quay-enterprise/docs/latest/tectonic/files/quay-servicetoken-role-binding-k8s1-6.yaml
```

The first one creates a role and the second one creates a rolebinding.

```
$ cat quay-servicetoken-role-k8s1-6.yaml 
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:  
  name: quay-enterprise-serviceaccount
  namespace: quay-enterprise
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - put
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get


$ cat quay-servicetoken-role-binding-k8s1-6.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: quay-enterprise-secret-writer
  namespace: quay-enterprise
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: quay-enterprise-serviceaccount
subjects:
- kind: ServiceAccount
  name: default
  
```

Now as a **Cluster Administrator** create the role and rolebinding by running the following commands

```
$ oc create -f quay-servicetoken-role-k8s1-6.yaml 
```

```
$ oc create -f quay-servicetoken-role-binding-k8s1-6.yaml
```

Also provide `anyuid` access to the `default` service account in the `quay-enterprise` project as *currently* both `redis` and `quay` runs as `root`. Run the following command to elevate priveleges to the `default` SA.

```
# oc adm policy add-scc-to-user anyuid -z default -n quay-enterprise

scc "anyuid" added to: ["system:serviceaccount:quay-enterprise:default"]
```

#### Step 4: Deploy Redis


Look at the deployment for `redis`. This includes two components one is a `deployment` and the other is a service that frontends the pod.

```
$ cat quay-enterprise-redis.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: quay-enterprise
  name: quay-enterprise-redis
  labels:
    quay-enterprise-component: redis
spec:
  replicas: 1
  selector:
    matchLabels:  
      quay-enterprise-component: redis
  template:
    metadata:
      namespace: quay-enterprise
      labels:
        quay-enterprise-component: redis
    spec:
      containers:
      - name: redis-master
        image: quay.io/quay/redis
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  namespace: quay-enterprise
  name: quay-enterprise-redis
  labels:
    quay-enterprise-component: redis
spec:
  ports:
    - port: 6379
  selector:
    quay-enterprise-component: redis
```

Let's create this deployment and service by running

```
$ oc create -f quay-enterprise-redis.yml

deployment "quay-enterprise-redis" created
service "quay-enterprise-redis" created
```

You should now see the pod getting created and a service already created

```
$ oc get po
NAME                                     READY     STATUS              RESTARTS   AGE
quay-enterprise-redis-6b59dc84b8-xblvc   0/1       ContainerCreating   0          19s
$ oc get svc
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
quay-enterprise-redis   ClusterIP   172.30.123.25   <none>        6379/TCP   25s  
```

Give it a few mins for the redis pod to be up and running.

#### Step 5: Sign up with CoreOS 

Go to [https://account.coreos.com/login](https://account.coreos.com/login) to sign up for a CoreOS account.

A confirmation email will be sent to the listed account. Check your inbox for an email from CoreOS Support, and click Verify Email.

Click Free for use up to 10 nodes below Tectonic, enter your contact information, and click Get License for 10 nodes to complete the registration process. Once complete, the Overview page will list active subscriptions, and provide access to your CoreOS License and Pull Secret.

Check the `Pull Secret`right below your account information. Either download this Pull Secret or Copy&Paste into a file with name `config.json`

You will need this file to pull `quay` image from [quay.io](quay.io)

#### Step 6: Deploy quay registry

Understand the deployment file that creates quay

```
$ cat quay-enterprise-app-rc.yml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: quay-enterprise
  name: quay-enterprise-app
  labels:
    quay-enterprise-component: app
spec:
  replicas: 1
  selector:
    matchLabels:
      quay-enterprise-component: app
  template:
    metadata:
      namespace: quay-enterprise
      labels:
        quay-enterprise-component: app
    spec:
      volumes:
        - name: configvolume
          secret:
            secretName: quay-enterprise-config-secret
      containers:
      - name: quay-enterprise-app
        image: quay.io/coreos/quay:v2.9.2
        ports:
        - containerPort: 80
        volumeMounts:
        - name: configvolume
          readOnly: false
          mountPath: /conf/stack
      imagePullSecrets:
        - name: coreos-pull-secret
```
Note that this deployment uses `quay-enterprise-config-secret` and a dockerconfig as the imagePullSecret with name `coreos-pull-secret`. Let's first create both of them

Look at the following file

```
$ cat quay-enterprise-config-secret.yml 
apiVersion: v1
kind: Secret
metadata:
  namespace: quay-enterprise
  name: quay-enterprise-config-secret
```

Now create the secret by running the following command

```
$ oc create -f quay-enterprise-config-secret.yml 

secret "quay-enterprise-config-secret" created
```


Quay uses the dockerconfig that you downloaded in the previous step as the imagePullSecret. Let us add this as a secret with name `coreos-pull-secret` to our project by running this command

```
$ oc create secret generic coreos-pull-secret --from-file=".dockerconfigjson=config.json" --type='kubernetes.io/dockerconfigjson' --namespace=quay-enterprise

secret "coreos-pull-secret" created
```

Now let's create the deployment for quay.

```
$ oc create -f quay-enterprise-app-rc.yml
 
deployment "quay-enterprise-app" created
```

In a few mins you should see the quay pod running along with redis pod.

```
$ oc get po
NAME                                     READY     STATUS    RESTARTS   
quay-enterprise-app-5dc94f57dd-5tqbb     1/1       Running   0          
quay-enterprise-redis-6b59dc84b8-fczw7   1/1       Running   0          
```

Now let's create a service that front-ends quay pod. We downloaded a file with the following name `quay-enterprise-service-loadbalancer.yml` in the past. Look at this file it creates a service of type loadbalancer. Let us change this to create a regular service.

```
$ cat quay-enterprise-service-loadbalancer.yml
apiVersion: v1
kind: Service
metadata:
  namespace: quay-enterprise
  name: quay-enterprise
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      name: http
    - protocol: TCP
      port: 443
      targetPort: 443
      name: https
  selector:
    quay-enterprise-component: app
```

Let us convert convert this to a regular service by substituting `LoadBalancer` type with `ClusterIP` type as shown below.

```
$ sed -e 's/LoadBalancer/ClusterIP/g' quay-enterprise-service-loadbalancer.yml > quay-enterprise-service.yml

$ cat quay-enterprise-service.yml
apiVersion: v1
kind: Service
metadata:
  namespace: quay-enterprise
  name: quay-enterprise
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      name: http
    - protocol: TCP
      port: 443
      targetPort: 443
      name: https
  selector:
    quay-enterprise-component: app
```

Now let us create the service by running

```
$ oc create -f quay-enterprise-service.yml

service "quay-enterprise" created
```

Expose the service to create a route

```
$ oc expose svc quay-enterprise
```

```
$ oc get route
NAME              HOST/PORT                                                  PATH      SERVICES          PORT      TERMINATION   WILDCARD
quay-enterprise   quay-enterprise-quay-enterprise.apps.opsday.ocpcloud.com             quay-enterprise   http                    None
```


#### Step 7: Install MySQL DB

Quay requires a database a MySQL DB or PostgreSQL. Let's install a MySQL DB. For this exercise, I am just using MySQL ephemeral, but you can do a persistent database if you wanted to.

Choose your `MYSQL_PASSWORD` and `MYSQL_ROOT_PASSWORD`

```
$ export MYSQL_PASSWD=<yourvalue>
$ export MYSQL_ROOT_PASSWD=<yourvalue>
```

Now create the `mysql` database service

```
$ oc new-app mysql-ephemeral \
-p DATABASE_SERVICE_NAME=mysql \
-p MYSQL_USER=coreosuser \
-p MYSQL_PASSWORD=${MYSQL_PASSWD} \
-p MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWD} \
-p MYSQL_DATABASE=enterpriseregistrydb

.....
.....
--> Creating resources ...
    secret "mysql" created
    service "mysql" created
    deploymentconfig "mysql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/mysql' 
    Run 'oc status' to view your app.
```

Run `oc get po` to see MySQL pod running in a few mins.

#### Step 8: Set up Quay 

Access the Quay URL `http://{route>/setup`, (`http://quay-enterprise-quay-enterprise.apps.opsday.ocpcloud.com/setup` in my case) to run Quay setup.

Follow the instructions to configure your MySQL database details and Redis service name `quay-enterprise-redis` for Quay to connect to. You will also be setting up `admin` user name for Quay.  

The Quay pod restarts a couple of times and will be ready to go.



**Congratulations** You have installed Quay on your cluster.

 







 






