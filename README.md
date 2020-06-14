# TimescaleDB K8s - A simple TimescaleDB deployment to kubernetes

In this recipe, the following settings are applied:

- Name of the timescaledb database : `postgres`
- User of the database: `postgres`
- Password : `ghmuSuWyOb6ewjxfloRG7VYe3DPxRgWMmp2iPh`
- Available memory allocated to timescaledb: `12GB`
- Number of CPUs allocated to timescaledb: `4GB`
- Maximum size of the database: `10GB`

This document also explains how to do differently.

## 1. Install timescaleDB

### 1.1 Create a namespace (optional)

If you need to create a dedicated namespace (e.g. named `timescaledb`), use the following command:

	$ kubectl create namespace timescaledb

Then execute the command below to switch to:

	$ kubectl config set-context --current --namespace timescaledb
	

### 1.2 Set database name, login and password in a configMap

#### 1.2.1 Create file `1-configMap.yaml`:

	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: timescaledb-config
	  labels:
	    app: timescaledb
	data:
	  POSTGRES_DB: postgres
	  POSTGRES_USER: postgres
	  POSTGRES_PASSWORD: ghmuSuWyOb6ewjxfloRG7VYe3DPxRgWMmp2iPh

> Feel free to change `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` as needed.

#### 1.2.2 Apply

	$ kubectl apply -f "1-configMap.yaml"	

### 1.3 Create persistent storage

### 1.3.1 Create file `2-pvc.yaml`

	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: timescaledb-data
	  labels:
	    app: timescaledb
	spec:
	  accessModes:
	    - ReadWriteOnce
	  resources:
	    requests:
	      storage: 10Gi
	      
> Feel free to update `storage: 10Gi` to fit with your needs`.

### 1.3.2 Apply

	$ kubectl apply -f "2-pvc.yaml"	
	
Check creation of the volume by using the following command:

	$ kubectl get pvc/timescaledb-data
	NAME               STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS            AGE
	timescaledb-data   Bound    ...      10Gi       RWO            csi-cinder-high-speed   1m
	
### 1.4 Deploy TimescaleDB image

#### 1.4.1. Create file `3-deployment.yaml`

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: timescaledb
	  labels:
	    app: timescaledb
	spec: 
	  selector:
	    matchLabels:
	      app: timescaledb
	  template:
	    metadata:
	      labels:
	        app: timescaledb
	    spec:
	      containers:
	        - name: postgres
	          image: "timescale/pg_prometheus:latest-pg11"
	          imagePullPolicy: "IfNotPresent"
	          ports:
	            - containerPort: 5432
	              name: timescaledb
	          env:
	            - name: TS_TUNE_MEMORY
	              value: "12GB"
	            - name: TS_TUNE_NUM_CPUS
	              value: "4"
	          envFrom:
	            - configMapRef:
	                name: timescaledb-config
	          volumeMounts:
	            - mountPath: /var/lib/postgresql/data
	              name: data
	              subPath: timescaledb
	      volumes:
	        - name: data
	          persistentVolumeClaim:
	            claimName: timescaledb-data

> In this example, the available memory allocated to timescaledb is set to 12GB `TS_TUNE_MEMORY` and the number of CPUs is set to 4 `TS_TUNE_NUM_CPUS`. Feel free to adjust these settings.

#### 1.4.2 Add liveness and readiness probes (optional)

It is a good practice to define liveness and readiness probes,. To do so, insert those lines in the file `3-deployment.yaml
		
	apiVersion: apps/v1
	kind: Deployment
		...
	    spec:
	      containers:
	        - name: postgres
	          ...
	          volumeMounts:
	          ...
	          livenessProbe:
                exec:
                  command:
                  - pg_isready
                  - -h
                  - localhost
                  - -U
                  - postgres
                initialDelaySeconds: 30
                timeoutSeconds: 5
              readinessProbe:
                exec:
                  command:
                  - pg_isready
                  - -h
                  - localhost
                  - -U
                  - postgres
                initialDelaySeconds: 5
                timeoutSeconds: 1
		   volumes:
		   ...
		   
> Ensure that the user `postgres` fit with your settings (ConfigMap above).
	
#### 1.4.3 Apply

	$ kubectl apply -f "3-deployment.yaml"
	
Check creation of the deployment by using the following command:

	$ kubectl get all
	NAME                                   READY   STATUS    RESTARTS   AGE
	...
	pod/timescaledb-bbf69bc8c-p7mb8        1/1     Running   0          1m
	...
	NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/timescaledb        1/1     1            1           1m


## Create Service

### 1. Create file `4-service.yaml`

	apiVersion: v1
	kind: Service
	metadata:
	  name: timescaledb
	  labels:
	    app: timescaledb
	spec:
	  type: LoadBalancer
	  ports:
	   - port: 5432
	     protocol: TCP
	  selector:
	   app: timescaledb

> By using the service type `LoadBalancer`, the database will be accessible by the outside (north-south traffic). If you want to restrict the access only inside the cluster (east-west traffic), use instead the service type `ClusterIP` as shown below: 

	apiVersion: v1
	kind: Service
	...
	spec:
	  type: ClusterIP
	  ...

After provision of the IP address or DNS name (depending on the k8s cluster, you may wait several minutes), you can check the service by using this command:

	$ kubectl get svc
	NAME               TYPE           CLUSTER-IP   EXTERNAL-IP                         PORT(S)          AGE
	timescaledb        LoadBalancer   10.3.3.155   6ddrv7foi4.lb.c1.gra7.k8s.ovh.net   5432:30620/TCP   1m
	
> In this example, the cluster has returned the DNS name to externally access to the cluster (north-south traffic): `6ddrv7foi4.lb.c1.gra7.k8s.ovh.net`
	
Well done! TimescaleDB is installed to K8S, now let's see how to connect to the database instance.

## 2 Check connection to TimescaleDB with pgamin4

### 2.1 Install pgadmin4 with helm

1. **First**, check if `helm` is installed locally as well as in your Kubernetes cluster:

	    $ helm list
	
	Shall return something like:
	
	    NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION
	
	> If the `helm` command returns nothing, follow the instructions given [here](https://helm.sh/docs/intro/install/).

2. **Second**, add the repo `runix`:

    	$ helm repo add runix https://helm.runix.net/
    
2. **Finally**, don't forget to replace \<your email\> and \<your password\> in the following command and execute it:

	    $ helm install pgadmin runix/pgadmin4 --set service.type=ClusterIP --set env.email=<your email> --set env.password=<your password>
	    
	    NAME: pgadmin
	    LAST DEPLOYED: Sun Jun 14 19:13:08 2020
	    NAMESPACE: timescaledb
	    STATUS: deployed
	    REVISION: 1
	    NOTES:
	    1. Get the application URL by running these commands:
		  export POD_NAME=$(kubectl get pods --namespace timescaledb -l "app.kubernetes.io/name=pgadmin4,app.kubernetes.io/instance=pgadmin" -o jsonpath="{.items[0].metadata.name}")
		  echo "Visit http://127.0.0.1:8080 to use your application"
		  kubectl port-forward $POD_NAME 8080:80
	
	> You can also use a LoadBalancer service to externally access to pgadmin4 (north-south traffic). To do so, just replace `service.type=ClusterIP` 
	by `service.type=LoadBalancer` in the helm command above.
	
	> If nginx-ingress is available in your cluster, you can alternatively create an ingress to access to pgadmin4. An example is given in the file k8s/5-pgadmin4-ingress.yaml.
    
### 2.2 Connect to pgadmin4 and configure TimescaleDB connection

1. Connect to pgadmin4 using the DNS Name (or IP address) given by the LoadBalancer service or by using the ingress subdomain. Fill email and password at login page with the values given to \<your email\> and \<your password\> just above.
2. In the left left side of pgadmin4, right-click on `Servers`, then select `Create/Server` option.
3. In `General` tab, give the connection a name, e.g. `timescaledb`
4. In `Connection` tab, type the internal kubernetes dns name in the `Host name/address` field: i.e`timescaledb.timescaledb.svc.cluster.local` (the first token corresponds to the k8s service name, the second to the namespace name).
5. Fill the username field (e.g. postgres) and the password depending on your ConfigMap settings 
6. Click on the `Save` button to test the connection between pgadmin4 and TimescaleDB.

> If you have configured a LoadBalancer service for TimescaleDB, you can alternatively use it in `Host name/address` field.

## 3. What's next ?

### 3.1 Tune TimescaleDB

<mark>TODO</mark>

### 3.2 Increase TimescaleDB volume size

<mark>TODO</mark>


