This document outlines the object translation logic between AKO and the Avi controller. It's assumed that the reader is minimally versed with both
Kubernetes object semantics and the Avi Object semantics. 

### Service of type loadbalancer

AKO creates a Layer 4 virtualservice object in Avi corresponding to a service of type loadbalancer in Kubernetes. Let's take an example of such a service object in Kubernetes:

    apiVersion: v1
    kind: Service
    metadata:
      name: avisvc-lb
      namespace: red
    spec:
      type: LoadBalancer
      ports:
      - port: 80
        targetPort: 8080
        name: eighty
      selector:
        app: avi-server

AKO creates a dedicated virtual service for this object in kubernetes that refers to reserving a virtual IP for it. The layer 4 virtual service uses a pool section logic based on the ports configured on the service of type loadbalancer. In this case, the incoming port is port `80` and hence the virtual service listens on this ports for client requests. AKO selects the pods associated with this service as pool servers associated with the virtualservice.


### Insecure Ingress.

Let's take an example of an insecure hostname specification from a Kubernetes ingress object:

    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: my-ingress
    spec:
      rules:
        - host: myinsecurehost.avi.internal
          http:
            paths:
            - path: /foo
              backend:
                serviceName: service1
                servicePort: 80
              
For insecure host/path combinations, AKO uses a Sharded VS logic where based on either the `namespace` of this ingress or the `hostname`
value (`myhost.avi.internal`), a pool object is created on a Shared VS. A shared VS typically denotes a virtualservice in Avi that
is shared across multiple ingresses. A priority label is associated on the poolgroup against it's member pool (that is created as a part of
this ingress), with priority label of `myhost.avi.internal/foo`.

An associated datascript object with this shared virtual service is used to interpret the host fqdn/path combination of the incoming
request and the corresponding pool is chosen based on the priority label as mentioned above.

The paths specified are interpreted as `STARTSWITH` checks. This means for this particular host/path if pool X is created then, the matchrule can
be interpreted as - If the host header equals `myhost.avi.internal` and path `STARTSWITH` `foo` then route the request to pool X.

### Secure Ingress

Let's take an example of a secure ingress object:

    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: my-ingress
    spec:
      tls:
      - hosts:
        - myhost.avi.internal
        secretName: testsecret-tls
      rules:
        - host: myhost.avi.internal
          http:
            paths:
            - path: /foo
              backend:
                serviceName: service1
                servicePort: 80

##### SNI VS per secure hostname

AKO creates an SNI child VS to a parent shared VS for the secure hostname. The SNI VS is used to bind the hostname to an sslkeycert object.
The sslkeycert object is used to terminate the secure traffic on Avi's service engine. In the above example the `secretName` field denotes the
secret asssociated with the hostname `myhost.avi.internal`. AKO parses the attached secret object and appropriately creates the sslkeycert
object in Avi. The SNI virtualservice does not get created if the secret object does not exist in Kubernetes corresponding to the
reference specified in the ingress object.

##### Traffic routing post SSL termination

On the SNI VS, AKO creates httppolicyset rules to route the terminated (insecure) traffic to the appropriate pool object using the host/path
specified in the `rules` section of this ingress object.

##### Redirect secure hosts from http to https

Additionally - for these hostnames, AKO creates a redirect policy on the shared VS (parent to the SNI child) for this specific secure hostname.
This allows the client to automatically redirect the http requests to https if they are accessed on the insecure port (80).


### AKO created object naming conventions

In the current AKO model, all kubernetes cluster objects are created on the `admin` tenant in Avi. This is true even for multiple kubernetes clusters managed through a single IaaS cloud in Avi (for example - vcenter cloud). This poses a challenge where each VS/Pool/PoolGroup is expected to be unique to ensure no conflicts between similar object types.

AKO uses a combination of elements from each kubernetes objects to create a corresponding object in Avi that is unique for the cluster.

##### L4 VS names

The formula to derive a VirtualService (vsName) is as follows:

    vsName = vrfName + "--" + namespace + "--" + svcName`

Here the `vrfName` is the VRF context name used for the cluster.
`svcName` refers to the service object's name in kubernetes.
`namespace` refers to the namespace on which the service object is created.

##### L4 pool names

The following formula is used to derive the L4 pool names:

    poolname = vsName + "--" + listener_port`

Here the `listener_port` refers to the service port on which the virtualservice listens on. As it can be intepreted that the number of pools will be directly associated with the number of listener ports configured in the kubernetes service object.

##### L4 poolgroup names

The poolgroup name formula for L4 virtualservices is as follows:

    poolgroupname = vsName + "--" + listener_port`


##### Shared VS names

The shared VS names are derived based on a combination of fields to keep it unique per kubernetes cluster. This is the only object in Avi that does not derive it's name from any of the kubernetes objects.

###### If shardVSPrefix is used

If the user wants to custom name their shared VSes, then a shardVSPrefix can be specified in the values.yaml and AKO will override any internal naming convention with this prefix. For example if the shardVSPrefix is `Demo` and the ShardSize is `LARGE` then AKO will create the shared VSes as:
 
    Demo-0
    Demo-1
    ....
    Demo-7

###### If the ShardVSPrefix is not used

    ShardVSName = cloudName + "--" + vrfName + "-" <shardNum>
 
Here `cloudName` is the name of the IaaS VCenter cloud while shardNum is the number of the shared VS generated based on either hostname or namespace based shards.

##### Shared VS pool names

The formula to derive the Shared VS poolgroup is as follows:

    poolgroupname = vrfName + "--" + priorityLabel + "--" + namespace + "--" + ingName

Here the `priorityLabel` is a combination of the host/path combination specified in each rule of the kubernetes ingress object. `ingName` refers to the name of the ingress object while `namespace` refers to the namespace on which the ingress object is found in kubernetes.

##### Shared VS poolgroup names

The following is the formula to derive the Shared VS poolgroup name:

    poolgroupname = vrfName +"--" + vsName


##### SNI child VS names

The SNI child VSes namings vary between different sharding options.

###### Hostname shard

    vsName = vrfName + "--" + ingName + "--" + namespace + "--" + sniHostName

###### Namespace shard

    vsName = vrfName + "--" + ingName + "--" + namespace + "--" + secret

The difference in naming is done because with namespace based sharding only one SNI child is created per ingress/per secret object while in hostname based sharding each SNI VS is unique to the hostname specified in the ingress object.

##### SNI pool names

The formula to derive the SNI virtualservice's pools is as follows:

    poolname = vrfName + "--" + namespace + "--" + host + path + "--" + ingName

Here the `host` and `path` variables denote the secure hosts' hostname and path specified in the ingress object.

##### SNI poolgroup names

The formula to derive the SNI virtualservice's poolgroup is as follows:

    poolgroupname = vrfName + "--" + namespace + "--" + host + path + "--" + ingName

Some of these naming conventions can be used to debug/derive corresponding Avi object names that could prove as a tool for first level trouble shooting.