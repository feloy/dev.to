# odo, a tool to simplify development on Kubernetes

In this article, we will see how to use the `odo` tool to develop cloud native applications, without the need for developers to manipulate containers and Kubernetes resources.

The `odo` tool is based on the deployment of pre-defined containers embedding all the environment to build and execute the developed programs. This way, the developers do not need to write any `Dockerfile` or Kubernetes manifests. 

> In the article, we will use quoted text, as for this paragraph, and a different narrative, to describe what happens in the Kubernetes cluster, generally using the `kubectl` command. This helps to clearly separate what are the commands executed by developers on a daily basis, and what are the commands executed only to illustrate in the article what happens when developers execute the commands, and what an experienced Kubernetes user can expect to see in the cluster.

## Installing `odo`

The `odo` tool is composed of a single binary. Developers can install the binary either from the Golang source available at github.com/openshift/odo, or can download a binary from https://mirror.openshift.com/pub/openshift-v4/clients/odo/. The detailed instructions to install `odo` are available at https://odo.dev/docs/installing-odo/.

### Auto-completion

Developers can activate auto-completion for the `odo` command using:

```
$ odo --complete
```

> On Linux, this command will add the following line at your `~/.bashrc` file:
>
>```
>complete -C /path/to/your/odo odo
>```
>

To effectively activate the completion without restarting a new terminal, developers need to reload the `.bashrc` file, with:

```
$ source ~/.bashrc
```

## Creating a new project

An application developed with `odo` is contained into a **project**. Developers can create a new project with the following command:

```
$ odo project create prj1
```

> On the cluster, this command creates a Kubernetes namespace of the same name into the current cluster:
>
> ```
> $ kubectl get namespaces
> NAME              STATUS   AGE
> [...]
> prj1              Active   5s
> ```
>
> It also sets this newly created namespace as the current namespace of the current Kubernetes context:
>
> ```
> $ kubectl ns -c
> prj1
> ```
>

## Creating a new micro-service

As an example of micro-service, we will create an API, using the Nest framework:

```
$ nest new prj1-api
$ cd prj1-api
```

> You can test the API locally, by running the following commands:
>
> ```
> $ npm install
> $ npm start
> ```
>
> Executing these commands, and if you are lucky enough (if the port 3000 is not already in use for example), the API will start listening for HTTP requests on the port 3000 of your host.
>
> You can test the result of a request with the command from a new terminal:
>
> ```
> $ curl http://localhost:3000
> Hello World!
> ```
>

## Deploying the micro-service

Instead of building and running the API locally, developers can deploy the API into the Kubernetes cluster using specific `odo` commands.

First, developers need to add the API as a component of the project, with the command:

```
$ odo create nodejs prj1-api
Devfile Object Validation
 ✓  Checking devfile existence [51789ns]
 ✓  Creating a devfile component from registry: DefaultDevfileRegistry [82276ns]
Validation
 ✓  Validating if devfile name is correct [93222ns]

Please use `odo push` command to create the component with source deployed
```

Within this command, developers indicate that the component uses the `nodejs` environment. This way, `odo` knows it has to deploy a pre-defined container containing all the tools necessary to build and execute such applications (for example `npm` and `node`). `odo` also knows how to build and start the application (here with `npm install`, `npm start`).

The effect of this command is to create a `devfile.yaml` file, describing the project and its components. At this point, nothing has been deployed into the cluster, and the developer has to run the following command to deploy the component just defined:

```
$ odo push
Validation
 ✓  Validating the devfile [20769ns]

Creating Kubernetes resources for component prj1-api
 ✓  Waiting for component to start [2m]
 ✓  Waiting for component to start [4ms]
 ⚠  Unable to create ingress, missing host information for Endpoint http-3000, please check instructions on URL creation (refer `odo url create --help`)


Applying URL changes
 ✓  URLs are synced with the cluster, no changes are required.

Syncing to component prj1-api
 ✓  Checking files for pushing [453118ns]
 ✓  Syncing files to the component [113ms]

Executing devfile commands for component prj1-api
 ✓  Waiting for component to start [1ms]
 ✓  Executing install command "npm install" [32s]
 ✓  Executing run command "npm start" [1s]

Pushing devfile component "prj1-api"
 ✓  Changes successfully pushed to component
```

> On the cluster, a pod will be deployed, containing a container in which the API will be built and started.
> 
> ```
> $ kubectl get pods
> NAME                        READY   STATUS    RESTARTS   AGE
> prj1-api-7d58fcf4f4-4m727   1/1     Running   0          2m41s
> ```
>
> A Kubernetes *Service* has been created, and you can try to access the API through the service address:
> 
> ```
> $ kubectl port-forward svc/prj1-api 3000 > /dev/null &
> $  curl http://localhost:3000/
> Hello World!
> ```
>

Developers can examine the logs of the component with the command:

```
$ odo log
[...]
[Nest] 108   - 05/07/2021, 11:16:06 AM   [NestApplication] Nest application successfully started
```

## Working on the micro-service

After editing the code of the API, developers can restart the API into the cluster with:

```
$ odo push
```

This command will synchronize the sources into the container, and will rebuild and restart the application in the container.

If developers want to automate this process, they can run the command `odo watch`, and this process will be done every time they save modifications into the API sources:

```
$ odo watch
```

If developers want to undeploy the API from the cluster, keeping the information into the `devfile.yaml` file, they can run:

```
$ odo delete prj1-api
```

## Configuration

Micro-services generally need some configuration, and this configuration is generally done through environment variables.

To pass environment variables to a component, developers can use the `odo config set --env` command:

```
$ odo config set --env PORT=3000 --env LOG_LEVEL=debug
 ✓  Environment variables were successfully updated

Run `odo push` command to apply changes to the cluster

$ odo push
```

With this example, two environment variables `PORT` and `LOG_LEVEL` will be defined into the container running the API.

> These environment variables are defined in the *Pod* spec template, as we can see by looking at the definition of the *Deployment* resource created by `odo`:
>
> ```
> $ kubectl get deployments.apps prj1-api -o yaml
>    [...]
>    containers:
>    - name: runtime
>      env:
>      - name: PORT
>        value: "3000"
>      - name: LOG_LEVEL
>        value: debug
>    [...]        

## Conclusion

In this article, we have introduced the `odo` tool and how developers can use it to develop and test their micro-services in a Kubernetes cluster without the need to write `Dockerfile` or Kubernetes manifests.

The complete documentation of `odo` is available at https://odo.dev/.

The `odo` tool permits many more things, and also provides IDEs integrations, for *Visual Studio Code* and *IntelliJ*. More articles are coming with these features.

