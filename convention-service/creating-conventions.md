# Creating conventions

This document describes how to create and deploy custom conventions to the Tanzu Application Platform.

## <a id='intro'></a>Introduction

Tanzu Application Platform helps developers transform their code
into containerized workloads with a URL.
The Supply Chain Choreographer for Tanzu manages this transformation.
For more information, see [Supply Chain Choreographer](../scc/about.html).

[Convention Service](about.md) is a key component of the supply chain
compositions the choreographer calls into action.
Convention Service enables people in operational roles to efficiently apply
their expertise. They can specify the runtime best practices, policies, and
conventions of their organization to workloads as they are created on the platform.
The power of this component becomes evident when the conventions of an organization
are applied consistently, at scale, and without hindering the velocity of application developers.

Opinions and policies vary from organization to organization. Convention
Service supports the creation of custom conventions to meet the unique operational needs
and requirements of an organization.

Before jumping into the details of creating a custom convention, let's look at two
distinct components of Convention Service: the
[convention controller](#convention-controller) and [convention server](#convention-server).

### <a id='convention-server'></a>Convention server

The convention server is the component that applies a convention already defined on the server.
Each convention server can host one or more conventions.
The application of each convention by a convention server can be controlled conditionally.
The conditional criteria governing the application of a convention is customizable and can be based
on the evaluation of a custom Kubernetes resource called [PodIntent](reference/pod-intent.md).
PodIntent is the vehicle by which Convention Service as a whole delivers its value.

A PodIntent is created, or updated if already existing, when a workload is run through a Tanzu Application Platform supply chain.
The custom resource includes both the PodTemplateSpec (see the [Kubernetes documentation](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-template-v1/#PodTemplateSpec)) and the OCI image metadata associated with a workload.
The conditional criteria for a convention can be based on any property or value found in the PodTemplateSpec or the Open Containers Initiative (OCI) image metadata available in the PodIntent.

If a convention's criteria are met, the convention server enriches the PodTemplateSpec
in the PodIntent. The convention server also updates the `status` section of the PodIntent
with the name of the convention that's been applied.
So if needed, you can figure out after the fact which conventions were applied to the workload.

To provide flexibility in how conventions are organized, you can deploy multiple convention servers. Each server can contain a convention or set of conventions focused on a
specific class of runtime modifications, on a specific language framework, and so on. How
the conventions are organized, grouped, and deployed is up to you and the needs of
your organization.

Convention servers deployed to the cluster will not take action unless triggered to
do so by the second component of Convention Service, the [convention controller](#convention-controller).

### <a id='convention-controller'></a>Convention controller

The convention controller is the orchestrator of one or many convention servers deployed to the cluster.
When the Supply Chain Choreographer creates or updates a PodIntent for a workload, the convention
controller retrieves the OCI image metadata from the repository
containing the workload's images and sets it in the PodIntent.

The convention controller then uses a webhook architecture to pass the PodIntent to each convention
server deployed to the cluster. The controller orchestrates the processing of the PodIntent by
the convention servers sequentially, based on the `priority` value that's set on the convention server.
For more information, see [ClusterPodConvention](reference/cluster-pod-convention.html).

After all convention servers are finished processing a PodIntent for a workload,
the convention controller updates the PodIntent with the latest version of the PodTemplateSpec and sets
`PodIntent.status.conditions[].status=True` where `PodIntent.status.conditions[].type=Ready`.
This status change signals the Supply Chain Choreographer that Convention Service is finished with its work.
The status change also executes whatever steps are waiting in the supply chain.

## <a id='get-started'></a>Getting started

With this high-level understanding of Convention Service components, let's look at how to create and deploy a custom convention.

> **Note:** This document covers developing conventions using [GOLANG](https://golang.org/), but this can be done using other languages by following the specs.

### <a id='prereqs'></a>Prerequisites

The following prerequisites must be met before a convention can be developed and deployed:

+ The Kubernetes command line tool (Kubectl) CLI is installed. For more information, see the [Kubernetes documentation](https://kubernetes.io/docs/tasks/tools/).
+ Tanzu Application Platform prerequisites are installed. For more information, see [Prerequisites](../prerequisites.md)
+ Tanzu Application Platform components are installed. For more information, see the [Installing the Tanzu CLI](../install-tanzu-cli.md).
+ The default supply chain is installed. Download Supply Chain Security Tools for VMware Tanzu from [Tanzu Network](https://network.tanzu.vmware.com/products/supply-chain-security-tools/).
+ Your kubeconfig context is set to the Tanzu Application Platform-enabled cluster:

    ```
    kubectl config use-context CONTEXT_NAME
    ```

+ The ko CLI is installed [from GitHub](https://github.com/google/ko).
  (These instructions use `ko` to build an image, but if there is an existing image or build process, `ko` is optional.)

## <a id='define-conv-criteria'></a> Define convention criteria

The `server.go` file contains the configuration for the server and the logic the server applies when a workload matches the defined criteria.
For example, adding a Prometheus sidecar to web applications, or adding a `workload-type=spring-boot` label to any workload that has metadata, indicating it is a Spring Boot app.  

> **Note:** For this example, the package `model` is used to define [resources](./reference/convention-resources.md) types.

1. <a id='convention-1'></a>The example `server.go` sets up the `ConventionHandler` to ingest the webhook requests([PodConventionContext](./reference/pod-convention-context.md)) from the convention controller. Here the handler must only deal with the existing [`PodTemplateSpec`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-template-v1/#PodTemplateSpec) and [`ImageConfig`](./reference/image-config.md).

   ```go
   ...
   import (
      corev1 "k8s.io/api/core/v1"
   )
   ...
   func ConventionHandler(template *corev1.PodTemplateSpec, images []model.ImageConfig) ([]string, error) {
       // Create custom conventions
   }
   ...
   ```

     Where:

     + `template` is the predefined `PodTemplateSpec` that the convention is going to modify. For more information about `PodTemplateSpec`, see the [Kubernetes documentation](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-template-v1/#PodTemplateSpec).
     + `images` are the [`ImageConfig`](./reference/image-config.md) used as reference to make decisions in the conventions. In this example, the type was created within the `model` package.

2. <a id='server-2'></a>The example `server.go` also configures the convention server to listen for requests:

    ```go
    ...
    import (
        "context"
        "fmt"
        "log"
        "net/http"
        "os"
        ...
    )
    ...
    func main() {
        ctx := context.Background()
        port := os.Getenv("PORT")
        if port == "" {
            port = "9000"
        }
        http.HandleFunc("/", webhook.ServerHandler(convention.ConventionHandler))
        log.Fatal(webhook.NewConventionServer(ctx, fmt.Sprintf(":%s", port)))
    }
    ...
    ```

    Where:

    + `PORT` is a possible environment variable, for this example, defined in the [`Deployment`](#install-deployment).
    + `ServerHandler` is the *handler* function called when any request comes to the server.
    + `NewConventionServer` is the function in charge of configure and create the *http webhook* server.
    + `port` is the calculated port of the server to listen for requests. It needs to match the [`Deployment`](#install-deployment) if the `PORT` variable is not defined in it.
    + The `path` or pattern (default to `/`) is the convention server's default path. If it is changed, it must be changed in the [`ClusterPodConvention`](#install-convention).

> **Note:** The *Server Handler* (`func ConventionHandler(...)`) and the configure/start web server (`func NewConventionServer(...)`) are defined in the convention controller within the `webhook` package, but a custom one can be used.

3. Creating the *Server Handler*, which handles the request from the convention controller with the [PodConventionContext](./reference/pod-convention-context.md) serialized to JSON.

    ```go
    package webhook
    ...
    func ServerHandler(conventionHandler func(template *corev1.PodTemplateSpec, images []model.ImageConfig) ([]string, error)) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            ...
            // Check request method
            ...
            // Decode the PodConventionContext
            podConventionContext := &model.PodConventionContext{}
            err = json.Unmarshal(body, &podConventionContext)
            if err != nil {
                w.WriteHeader(http.StatusBadRequest)
                return
            }
            // Validate the PodTemplateSpec and ImageConfig
            ...
            // Apply the conventions
            pts := podConventionContext.Spec.Template.DeepCopy()
            appliedConventions, err := conventionHandler(pts, podConventionContext.Spec.Images)
            if err != nil {
                w.WriteHeader(http.StatusInternalServerError)
                return
            }
            // Update the applied conventions and status with the new PodTemplateSpec
            podConventionContext.Status.AppliedConventions = appliedConventions
            podConventionContext.Status.Template = *pts
            // Return the updated PodConventionContext
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusOK)
            json.NewEncoder(w).Encode(podConventionContext)
        }
    }
    ...
    ```

4. Configure and start the web server by defining the `NewConventionServer` function, which starts the server with the defined port and current context. The server uses the `.crt` and `.key` files to handle *TLS* traffic.

    ```go
    package webhook
    ...
    // Watch handles the security by certificates.
    type certWatcher struct {
        CrtFile string
        KeyFile string

        m       sync.Mutex
        keyPair *tls.Certificate
    }
    func (w *certWatcher) Load() error {
        // Creates a X509KeyPair from PEM encoded client certificate and private key.
        ...
    }
    func (w *certWatcher) GetCertificate() *tls.Certificate {
        w.m.Lock()
        defer w.m.Unlock()

        return w.keyPair
    }
    ...
    func NewConventionServer(ctx context.Context, addr string) error {
        // Define a health check endpoint to readiness and liveness probes.
        http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
            w.WriteHeader(http.StatusOK)
        })

        if err := watcher.Load(); err != nil {
            return err
        }
        // Defines the server with the TSL configuration.
        server := &http.Server{
            Addr: addr,
            TLSConfig: &tls.Config{
                GetCertificate: func(_ *tls.ClientHelloInfo) (*tls.Certificate, error) {
                    cert := watcher.GetCertificate()
                    return cert, nil
                },
                PreferServerCipherSuites: true,
                MinVersion:               tls.VersionTLS13,
            },
            BaseContext: func(_ net.Listener) context.Context {
                return ctx
            },
        }
        go func() {
            <-ctx.Done()
            server.Close()
        }()

        return server.ListenAndServeTLS("", "")
    }
    ```
## <a id='define-conv-behavior'></a> Define the convention behavior

Any property or value within the [PodTemplateSpec](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-template-v1/#PodTemplateSpec) or OCI image metadata associated with a workload can be used to define the criteria for applying conventions. The following are a few examples.  

### <a id='match-crit-labels-annot'></a> Matching criteria by labels or annotations

When using labels or annotations to define whether a convention should be applied, the server checks the [PodTemplateSpec](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-template-v1/#PodTemplateSpec) of workloads.

   + PodTemplateSpec

        ```yaml
        ...
        template:
          metadata:
            labels:
              awesome-label: awesome-value
            annotations:
              awesome-annotation: awesome-value
        ...
        ```

   + Handler

        ```go
        package convention
        ...
        func conventionHandler(template *corev1.PodTemplateSpec, images []model.ImageConfig) ([]string, error) {
            c:= []string{}
            // This convention is applied if a specific label is present.
            if lv, le := template.Labels["awesome-label"]; le && lv == "awesome-value" {
                // DO COOl STUFF
                c = append(c, "awesome-label-convention")
            }
            // This convention is applied if a specific annotation is present.
            if av, ae := template.Annotations["awesome-annotation"]; ae && av == "awesome-value" {
                // DO COOl STUFF
                c = append(c, "awesome-annotation-convention")
            }

            return c, nil
        }
        ...
        ```

 Where:
 + `conventionHandler` is the *handler*.
 + `awesome-label` is the **label** that we want to validate.
 + `awesome-annotation` is the **annotation** that we want to validate.
 + `awesome-value` is the value that must have the **label**/**annotation**.

### <a id='match-criteria-env-var'></a> Matching criteria by environment variables

When using environment variables to define whether the convention is applicable, it should be present in the [PodTemplateSpec](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-template-v1/#PodTemplateSpec).[spec](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec).[containers](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)[*].[env](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#environment-variables). and we can validate the value.

   + PodTemplateSpec

        ```yaml
        ...
        template:
          spec:
            containers:
              - name: awesome-container
                env:
        ...
        ```

   + Handler

        ```go
        package convention
        ...
        func conventionHandler(template *corev1.PodTemplateSpec, images []model.ImageConfig) ([]string, error) {
            if len(template.Spec.Containers[0].Env) == 0 {
                template.Spec.Containers[0].Env = append(template.Spec.Containers[0].Env, corev1.EnvVar{
                    Name: "MY_AWESOME_VAR",
                    Value: "MY_AWESOME_VALUE",
                })
                return []string{"awesome-envs-convention"}, nil
            }
            return []string{}, nil
            ...
        }
        ```

### <a id='match-crit-img-metadata'></a>Matching criteria by image metadata

For each image contained within the PodTemplateSpec, the convention controller fetches the OCI image metadata and known [`bill of materials (BOMs)`](reference/bom.md) providing it to the convention server as [`ImageConfig`](./reference/image-config.md). This metadata can be introspected to make decisions about how to configure the PodTemplateSpec.

## <a id='install'></a> Configure and install the convention server

The `server.yaml` defines the Kubernetes components that enable the convention server in the cluster. The next definitions are within the file.

1. <a id='install-namespace'></a>A `namespace` is created for the convention server components and has the required objects to run the server. It's used in the [`ClusterPodConvention`](#install-convention) section to indicate to the controller where the server is.

    ```yaml
    ...
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: awesome-convention
    ---
    ...
    ```

2. <a id='install-cm'></a>(Optional) A certificate manager `Issuer` is created to issue the certificate needed for TLS communication.

    ```yaml
    ...
    ---
    # The following manifests contain a self-signed issuer CR and a certificate CR.
    # More document can be found at https://docs.cert-manager.io
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: awesome-selfsigned-issuer
      namespace: awesome-convention
    spec:
      selfSigned: {}
    ---
    ...
    ```

3. <a id='install-cert'></a>(Optional) A self-signed `Certificate` is created.

    ```yaml
    ...
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: awesome-webhook-cert
      namespace: awesome-convention
    spec:
      subject:
        organizations:
        - vmware
        organizationalUnits:
        - tanzu
      commonName: awesome-webhook.awesome-convention.svc
      dnsNames:
      - awesome-webhook.awesome-convention.svc
      - awesome-webhook.awesome-convention.svc.cluster.local
      issuerRef:
        kind: Issuer
        name: awesome-selfsigned-issuer
      secretName: awesome-webhook-cert
      revisionHistoryLimit: 10
    ---
    ...
    ```

4. <a id='install-deployment'></a>A Kubernetes `Deployment` is created for the webhook to run from. The [`Service`](#install-service) uses the container port defined by the `Deployment` to expose the server.

    ```yaml
    ...
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: awesome-webhook
      namespace: awesome-convention
    spec:
      replicas: 1
      selector:
        matchLabels:
        app: awesome-webhook
      template:
        metadata:
          labels:
            app: awesome-webhook
        spec:
          containers:
          - name: webhook
            # Set the prebuilt image of the convention or use ko to build an image from code.
            # see https://github.com/google/ko
            image: ko://awesome-repo/awesome-user/awesome-convention
          env:
          - name: PORT
            value: "8443"
          ports:
          - containerPort: 8443
            name: webhook
          livenessProbe:
            httpGet:
              scheme: HTTPS
              port: webhook
              path: /healthz
          readinessProbe:
            httpGet:
              scheme: HTTPS
              port: webhook
              path: /healthz
          volumeMounts:
          - name: certs
            mountPath: /config/certs
            readOnly: true
        volumes:
        - name: certs
          secret:
            defaultMode: 420
            secretName: awesome-webhook-cert
    ---
    ...
    ```

5.  <a id='install-service'></a>A Kubernetes `Service` to expose the convention deployment is also created. For this example, the exposed port is the default `443`, but if it is changed, the [`ClusterPodConvention`](#install-convention) needs to be updated with the proper one.

    ```yaml
    ...
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: awesome-webhook
      namespace: awesome-convention
      labels:
        app: awesome-webhook
    spec:
      selector:
        app: awesome-webhook
      ports:
        - protocol: TCP
          port: 443
          targetPort: webhook
    ---
    ...
    ```
6. <a id='install-convention'></a>Finally, the [`ClusterPodConvention`](./reference/cluster-pod-convention.md) adds the convention to the cluster to make it available for the Convention Controller:
    >**Note:** The `annotations` block is only needed if you use a self-signed certificate. Otherwise, check the [cert-manager documentation](https://cert-manager.io/docs/).

    ```yaml
    ...
    ---
    apiVersion: conventions.apps.tanzu.vmware.com/v1alpha1
    kind: ClusterPodConvention
    metadata:
      name: awesome-convention
      annotations:
        conventions.apps.tanzu.vmware.com/inject-ca-from: "awesome-convention/awesome-webhook-cert"
    spec:
      webhook:
        clientConfig:
          service:
            name: awesome-webhook
            namespace: awesome-convention
            # path: "/" # default
            # port: 443 # default
    ```

## <a id="deploy-convention-server"></a> Deploy a convention server

To deploy a convention server:

1. Build and install the convention.

    + If the convention needs to be built and deployed, use the [ko] tool on GitHub (https://github.com/google/ko). It compiles yout _go_ code into a docker image and pushes it to the registry(`KO_DOCKER_REGISTRY`).

        ```bash
        ko apply -f dist/server.yaml
        ```

    + If a different tool is used to build the image, the configuration can be also be applied using either kubectl or `kapp`, setting the correct image in the [`Deployment`](#install-convention) descriptor.

       kubectl

        ```bash
        kubectl apply -f server.yaml
        ```

       kapp

        ```bash
        kapp deploy -y -a awesome-convention -f server.yaml
        ```

2. Verify the convention server.
To check the status of the convention server, check for the running convention Pods:

    + If the server is running, `kubectl get all -n awesome-convention` returns something like:

        ```text
        NAME                                       READY   STATUS    RESTARTS   AGE
        pod/awesome-webhook-1234567890-12345       1/1     Running   0          8h

        NAME                          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
        service/awesome-webhook       ClusterIP   10.56.12.49   <none>        443/TCP   28h

        NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/awesome-webhook       1/1     1            1           28h

        NAME                                             DESIRED   CURRENT   READY   AGE
        replicaset.apps/awesome-webhook-1234563213       0         0         0       23h
        replicaset.apps/awesome-webhook-5b79d5cb59       0         0         0       28h
        replicaset.apps/awesome-webhook-5bf557c9f8       1         1         1       20h
        replicaset.apps/awesome-webhook-77c647c987       0         0         0       23h
        replicaset.apps/awesome-webhook-79d9c6f74c       0         0         0       23h
        replicaset.apps/awesome-webhook-7d9d667b8d       0         0         0       9h
        replicaset.apps/awesome-webhook-8668664d75       0         0         0       23h
        replicaset.apps/awesome-webhook-9b6957476        0         0         0       24h
        ```

    + To verify the conventions are being applied, check the `PodIntent` of a workload that matches the convention criteria:  

        ```bash
        kubectl -o yaml get podintents.conventions.apps.tanzu.vmware.co awesome-app
        ```

        ```yaml
        apiVersion: conventions.apps.tanzu.vmware.com/v1alpha1
        kind: PodIntent
        metadata:
          creationTimestamp: "2021-10-07T13:30:00Z"
          generation: 1
          labels:
            app.kubernetes.io/component: intent
            carto.run/cluster-supply-chain-name: awesome-supply-chain
            carto.run/cluster-template-name: convention-template
            carto.run/component-name: config-provider
            carto.run/template-kind: ClusterConfigTemplate
            carto.run/workload-name: awesome-app
            carto.run/workload-namespace: default
          name: awesome-app
          namespace: default
        ownerReferences:
        - apiVersion: carto.run/v1alpha1
          blockOwnerDeletion: true
          controller: true
          kind: Workload
          name: awesome-app
          uid: "********"
        resourceVersion: "********"
        uid: "********"
        spec:
        imagePullSecrets:
          - name: registry-credentials
            serviceAccountName: default
            template:
              metadata:
                annotations:
                  developer.conventions/target-containers: workload
                labels:
                  app.kubernetes.io/component: run
                  app.kubernetes.io/part-of: awesome-app
                  carto.run/workload-name: awesome-app
              spec:
                containers:
                - image: awesome-repo.com/awesome-project/awesome-app@sha256:********
                  name: workload
                  resources: {}
                  securityContext:
                  runAsUser: 1000
        status:
          conditions:
          - lastTransitionTime: "2021-10-07T13:30:00Z"
            status: "True"
            type: ConventionsApplied
          - lastTransitionTime: "2021-10-07T13:30:00Z"
            status: "True"
            type: Ready
        observedGeneration: 1
        template:
          metadata:
            annotations:
              awesome-annotation: awesome-value
              conventions.apps.tanzu.vmware.com/applied-conventions: |-
                awesome-label-convention
                awesome-annotation-convention
                awesome-envs-convention
                awesome-image-convention
                developer.conventions/target-containers: workload
            labels:
              awesome-label: awesome-value
              app.kubernetes.io/component: run
              app.kubernetes.io/part-of: awesome-app
              carto.run/workload-name: awesome-app
              conventions.apps.tanzu.vmware.com/framework: go
          spec:
            containers:
            - env:
              - name: MY_AWESOME_VAR
                value: "MY_AWESOME_VALUE"
              image: awesome-repo.com/awesome-project/awesome-app@sha256:********
              name: workload
              ports:
                - containerPort: 8080
                  protocol: TCP
              resources: {}
              securityContext:
                runAsUser: 1000
        ```

## <a id='next-steps'></a> Next Steps

Keep Exploring:

+ Try to use different matching criteria for the conventions or enhance the supply chain with multiple conventions.
