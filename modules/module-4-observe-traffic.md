# Module 4 - Observe traffic flows in Calico Cloud

## Install sample application stacks

1. From the cloned directory, execute:

    ```bash
    kubectl apply -f manifests
    ```
  
    (Optional) Also install the metrics-server on EKS to get an idea as to the resource consumption on the cluster

    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```

    Verify that the sample applications are running:

    ```bash
    kubectl get pods -A
    ```
  
## Check traffic flows

1. Try sending some traffic between the pods

   a. From the ```management-ui``` pod in ```management-ui``` namespace to the ```customer``` pod in the ```yaobank``` namespace  

    ```bash
    kubectl exec -it -n management-ui deploy/management-ui -- sh -c 'curl -m3 -sI http://customer.yaobank 2>/dev/null | grep -i http'
    ```

   The expected result should be ```HTTP/1.0 200 OK``` indicating that the request succeeded between the namespaces.

2. Each application also has a webserver with a ```LoadBalancer``` service to access from the internet to check that the application is functioning normally.

   a. Validate and access the ```management-ui``` svc of the Stars application via the ```Loadbalancer``` service in a browser

    ```bash
    kubectl get svc -n management-ui
    ```

    The output gives the external-IP of the AWS LB that can be used to access the svc:

    ```bash
    NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)        AGE
    management-ui   LoadBalancer   10.100.154.186   a2b94c3b1192d490f8c4b1b9caf30589-1684915063.ca-central-1.elb.amazonaws.com   80:31996/TCP   4h48m
    ```

    In a browser, the following should be seen:
    ![stars_gui](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/7774d604-361c-4fe9-928f-18b45a4bb948)

    

   b. Validate and access the  ```customer``` svc of the Yaobank application via its ```Loadbalancer``` service in a browser

    ```bash
    kubectl get svc customer -n yaobank
    ```

      The output gives the external-IP of the AWS LB that can be used to access the svc

    ```bash
    NAME       TYPE           CLUSTER-IP      EXTERNAL-IP                                                                  PORT(S)        AGE
    customer   LoadBalancer   10.100.75.183   a373657b6b99a44e58503a2377ec7de9-1936085547.ca-central-1.elb.amazonaws.com   80:30180/TCP   3h14m
    ```

    In a browser, the following should be seen:

    ![yaobank_ui](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/1b4db718-ebf9-43d6-a0a7-92311a539472)

## Observe traffic flows

### Dynamic Service and Threat Graph

In Calico Cloud, on the left hand side click Service Graph > Views > Default.

You should see a topological view of all of the namespaces within your cluster, including ```stars```, ```yaobank```, ```management-ui``` and ```client```.
The screen is busy with Tigera and Calico resources, so let's hide them from this view.

Expand the left-hand panel, and in the Layers tab click on the three dots next to Tigera Infrastructure and choose ```hide layer```.

This makes it easier to see the namespaces we created.

Double-click on ```stars``` to see what resources are inside this namespace.
</br>

#### What are the key features within Service Graph?

##### Flow Lines
In the service graph view, notice the lines between resources. The colour of the flow indicates whether the flow is allowed or denied, and the direction of the arrow shows the direction of communication. Hovering over the line shows the number of allowed or denied packets and bytes.

##### Resources
Each component in the cluster is represented visually in the service graph. The icon indicates whether it's a network, service, replicaset, etc. Hovering over an icon shows the amount of ingress or egress traffic. Right-clicking on a resource lets you manage the view (hiding or de-emphasizing it) and also initiate a packet capture.

##### Flows
As you click around within Service Graph, you may notice the values in the flows table in the bottom panel change. As you click on objects in Service Graph, the Flows table automatically creates filters specific to that object. Each row in the table is a traffic flow within the cluster and includes information about the source, destination, port, protocol, network policies, Kubernetes metadata and more. You can write your own filters.

### Flow Visualization

In the left-hand menu select the second option which should be ```Service Graph```. Select ```Flow Visualizations``` under ```Other```.

FlowViz gives you a 360' view of the volumetric flow of traffic within your cluster.

#### Understanding FlowViz
Moving from the outside of the circle inwards you can see:

#### Namespace
The outside segments of the circle represent namespaces. The larger the arc segment, the more traffic within that cluster is going to/from that namespace.

#### Workload
The segments on the second inner ring represent workloads or services. These are the resources within the namespaces that have traffic going in or out of them.

#### Flows
The inner ring of segments represents flows. One pod or service within a namespace may have multiple flows, where it's communicating with multiple other services or a different network. These are colour coded and will be green if traffic is allowed and red if the traffic is denied. There is a legend on the bottom right of the screen.

#### Flow lines
In the middle of the circle a volumetric flow lines representing all communication within the cluster.

#### Filtering Flow Visualization

To change the colour scheme and highlight different information use the toggles on the top right.

Below this you can filter your flows and it will emphasize them within the Flow Visualization view.
Depending on the filters that are configured you will be able to see Kubernetes metadata, policies, traffic listed.

If you've filtered or selected a specific namespace, clicking the magnifying glass to zoom in will change the view to just that namespace, and any other namespaces or networks that it communicates with.


[:arrow_right: Module 5 - Secure pod traffic using Calico Policy Recommender](module-5-secure-pod-traffic.md)   <br>

[:arrow_left: Module 3 - Deploy an EKS cluster](module-3-connect-calicocloud.md)

[:leftwards_arrow_with_hook: Back to Main](../README.md)  
