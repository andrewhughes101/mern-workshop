MEAN workshop intended for Node Conf EU

# Requirements
- Docker - https://docs.docker.com/install/
- Kubernetes for Docker - Docker > Preferences > Enable Kubernetes
- Kubectl - `brew install kubernetes-cli` or https://kubernetes.io/docs/tasks/tools/install-kubectl/
- Node.js - https://nodejs.org/en/ (Node 8 LTS) or NVM

# Steps

1. Clone the MERN starter: `git clone https://github.com/IBM-Cloud/MERN-app.git`
2. Build the project with all dependencies, including dev dependencies, with the command:

    ```bash
    npm install
    ```

3. Start a mongodb docker container

  ```bash
   docker run -d -p 27017:27017 mongo
  ```

4. Run the project unit tests with the command:

    ```bash
    npm test
    ```

5. Run the app in dev mode with the command:

    ```bash
    npm run dev
    ```

    A development web server runs on port 3000 and the app itself runs on port 3100. The web server and app will automatically reload if changes are made to the source.

6. Run the app in interactive debug mode with the command:

    ```bash
    npm run debug
    ```

    The app listens on port 5858 for the debug client to attach to it, and on port 3000 for app requests.

### In release mode

1. Build the project:

    ```bash
    npm install --only=dev; npm run build; npm prune --production
    ```

    Upon completion, webpack has been run and dev dependencies removed.

2. Run the project:

    ```bash
    npm start
    ```

    The app will now run in release mode, listening on port 3000. Hot reload is not available in this mode.

## Setting up Minikube

1. Enure your Docker deamon is started

`docker images`

1. Verify that the kubectl is configured to use your cluster
`kubectl cluster-info`


## Deployment to Kubernetes

1. Build and tag the app with Docker: `docker build -t mern-app:v1.0.0 .`
1. Look at helm chart and ensure it refers to the image we just built and tagged above, key part being
```
repository: mern-app
tag: v1
```
1. `helm install --name mern-app chart/mernexample`
1. Check all pods come up with `kubectl get pods`

## Checking it works
To view the application go to `http://<cluster-external-IP>:<external-port>` in a browser. It should look something like this: `http://169.47.252.58:32080`

To find the external IP address of your Kubernetes cluster just use `$(minikube ip)`, if it's something external use your cloud provided Kubernetes dashboard. Find a _Nodes_ menu and look for the the _ExternalIP_ address

   ```
   Addresses:
   InternalIP: 10.177.184.198 ExternalIP: 169.47.252.58 Hostname: 10.177.184.198
   ```

Or run the command below [from the Kubernetes Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#viewing-finding-resources):

   ```bash
   $ kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
   169.47.252.58
   ```

Lastly, we'll need the external port. This was already given to us in the previous step after the `helm` command, but you can find it using:

```
$ kubectl get services
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
kubernetes            ClusterIP   172.21.0.1       <none>        443/TCP           17h
mernexample-service   NodePort    172.21.231.153   <none>        3000:32080/TCP    5h
mongo                 NodePort    172.21.67.175    <none>        27017:30308/TCP   5h
```

Where the column labeled `PORT(S)` has two values. The port number on the left is the internal / guest port from the container. The port number on the right is the external port. The external port is what you will use to access your application.

### Set your Helm Charts

### values.yml

* Open up `values.yml` under your charts directory (e.g. `chart/project/`)
* Set up the values that will be referenced in your mongo environments.

```yaml
services:
  mongo:
     url: {uri}
     dbName: {dbname}
     ca: {ca_certificate_base64}
     username: {username}
     password: {password}
     env: production
```
### bindings.yml

* Add the MONGO environment variables references at the end if they are not there already

```yaml
  - name: MONGO_URL
    value: {{ .Values.services.mongo.url }}
  - name: MONGO_DB_NAME
    value: {{ .Values.services.mongo.name }}
  - name: MONGO_USER
    value: {{ .Values.services.mongo.username }}
  - name: MONGO_PASS
    value: {{ .Values.services.mongo.password }}
  - name: MONGO_CA
    value: {{ .Values.services.mongo.ca }}
```

### Secrets (Optional)

If you prefer to not expose your credentials in your `deployment.yml` or `values.yml` you can use a base64 encoded string of your credentials. Using secrets is beyond the scope of this
README. You can find out how to use secrets in your application by reviewing the links below.

* [Creating a Secret Using kubectl create secret](https://kubernetes.io/docs/concepts/configuration/secret/#creating-your-own-secrets)
* [Encyrption Config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)


## Configure Mongoose (MongoDB Node) Client

* Open  `server/routers/mongo.js`
* Edit the MONGO environment variables

```js
  const mongoURL = process.env.MONGO_URL || 'localhost';
  const mongoUser = process.env.MONGO_USER || '';
  const mongoPass = process.env.MONGO_PASS || '';
  const mongoDBName = process.env.MONGO_DB_NAME || 'comments';
  const mongoCA = [new Buffer(process.env.MONGO_CA || '', 'base64')]
```

* Add SSL configurations

```js
  const options = {
      useMongoClient: true,
      ssl: true,
      sslValidate: true,
      sslCA: mongoCA,
      poolSize: 1,
      reconnectTries: 1
  };
```
