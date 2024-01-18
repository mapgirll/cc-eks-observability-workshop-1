# Module 6 - Zero-trust security for pod traffic

Policy Tiers are a hierarchical construct used to group policies and enforce higher precedence policies that other teams cannot circumvent, providing the basis for **Identity-aware microsegmentation**.

All Calico and Kubernetes security policies reside in tiers. You can start “thinking in tiers” by grouping your teams and the types of policies within each group.

In our scenario, we have a lot of namescoped application-specific policies for traffic which are fine-grained controls. Some of the policies can be segmented into higher tiers for wider cluster-wide policy implementation by a security team while the application team still has access to their own tier.

In our scenario we will consider the Star application to be used by one tenant while the Yaobank application is being used by another. We want to restrict the traffic within tenants as a cluster-wide policy in a higher ```security``` tier.

## Security tier policies overview

The ```security``` tier will be used to implement high-level guardrails for the cluster.

- A ```block-alienvault-ipthreatfeed``` security policy will be enforced for all cluster workloads. The policy will deny egress connectivity to malicious IPs in the ```threatfeed```.
- The ```cluster-dns-allow-all``` security policy will have rules to permit ingress DNS traffic to the ```kube-dns``` endpoints on TCP and UDP port 53 from all endpoints in the cluster. The security policy will also have egress rules to permit all endpoints in the cluster to send DNS traffic to the ```kube-dns``` endpoints on the same ports.
- Finally, ```security-default-pass``` policy will be used to pass any traffic that is not explicitly allowed or denied in this tier to the subsequent tier for policy processing.

## Default tier policies overview

The default tier is used to implement global default deny policy.

- ```default-deny``` denies any traffic that is not explicitly allowed in security, platform, and app tiers.

## Building the policies

1. Apply the tiers YAML
  
    ```bash
    kubectl apply -f policy/00-tiers.yaml
    ```

2. Apply the ```default-deny``` policy as a ```Staged``` policy first in the ```default``` tier for all the necessary namespaces:

    ```bash
    kubectl apply -f policy/01-default-deny.yaml
    ```

3. Apply the ```security``` tier policies

    ```bash
    kubectl apply -f policy/02-security.yaml
    ```

## Visualize denied staged traffic via Elasticsearch Log Explorer

1. To explore the flows that would be denied by the staged policies, we can explore the logs better in the ElasticSearch instance that collects all the flow logs. This can be accessed by clicking on ```Logs``` in the left menu, this takes us to the Kibana dashboard in a new browser tab.
    
    ![Logs_menu](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/8c20ddf6-f0bc-4325-a81d-71af8370d69e)

2. From the hamburger menu on the top left corner, click ```Discover```.

   ![kibana_discover](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/85a5702b-e210-4c4f-a784-ec5a66d7f63c) 

4. From the left menu bar just below Add filter, ensure tigera_secure_ee_flows* index is selected and then click on the plus sign next to the following flow logs metadata to filter through the metadata. Make sure to filter as per the order listed below to have an organized and clear view of the filtered information. Change the filter time range to ```last 15 minutes```.

    ```bash
    source_namespace
    source_name_aggr
    dest_namespace
    dest_name_aggr
    dest_port
    reporter
    policies
    index
    ```
    ![add_field](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/7c5e974e-e10b-42f9-8809-fbe43540adf2)

5. Once the above filter is implemented, you should see a page similar to the following:
    ![filtered_logs](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/c49e7ff9-1b31-4326-b161-2620ad4e7d41)

6. Type the following in the search bar of the ```Discover``` page, this will look for any traffic that would be getting denied by the staged policies we implemented in the default tier:

    ```policies:{ all_policies: *default.staged**deny*  }```

    We see some matches on traffic that our staged default-deny policy would be denying. Let's try to understand what flows are being blocked, whether they are legitimate traffic that instead needs to be allowed, and make the required changes to the policy.

> **Note:** If there are multiple clusters connected to your Calico Cloud you might see traffic flows for other clusters. You can check this in the *index* column and then add an additional filter.

7. Expand one of the flows and look at the policies:
```json
{
  "all_policies": [
    "1|namespace-isolation|stars/namespace-isolation.staged:stars-kcxyk|allow|2",
    "2|default|default.staged:default-deny|deny|-1",
    "3|__PROFILE__|__PROFILE__.kns.stars|allow|0",
    "0|security|security.security-default-pass|pass|0"
  ]
}
```
Here we can see that the security-default-pass policy in the security tier passed the traffic (it didn't match any policies in the security tier). Then it would've been allowed by the stars policy in the namespace-isolation tier if that policy were enforced. 
As staged policies don't impact traffic flow, it goes on to the default-deny policy where it would've been denied. As the default deny policy is staged the traffic flow is ultimately allowed by the default Kubernetes behaviour.

## Enforcing a Staged Policy

1. In the Policy Board select the default-deny policy and click ```Edit Policy```.

2. Check that the policy has the correct scope, applies to the correct endpoints and has the correct rules. For this default deny, it applies to all of the namespaces that were created earlier in the workshop.

```Policy label selector: [[projectcalico.org/namespace in yaobank,stars,client,management-ui,hipstershop]]``` 

3. Click ```Enforce```.

## Testing traffic flow to ```yaobank``` namespace

4. Let's re-run the traffic test from the ```management-ui``` pod in ```management-ui``` namespace to the ```customer``` pod in the ```yaobank``` namespace:

```bash
kubectl exec -it -n management-ui deploy/management-ui -- sh -c 'curl -m3 -sI http://customer.yaobank 2>/dev/null | grep -i http'
```

We should get ```command terminated with exit code 1``` as a result.

## App tier policies overview

The ```app``` tier is used by the Star and Yaobank applications to deploy namespace-scoped policies.

- Yaobank application policies consist of coarse-grained security policies for Yaobank app per namespace. An allow policy for the namespace will ensure that all workloads inside the namespace can communicate with one another. However, security ```rules``` must permit traffic flows in and out of the namespace.
- Stars application policies are fine-grained policies that specify ingress and egress rules for defined endpoints.
- Finally, the ```app-default-pass``` security policy has the lowest precedence in the app tier. It is deployed to ensure that a pass action is applied to all endpoints traffic matched in this tier, but the traffic was not explicitly allowed or denied. This rule causes the traffic matching it to be further processed in the default tier. We will implement global ```default-deny``` policy in the default tier blocking any traffic that was not explicitly allowed in the previous tiers.

Apply the ```app``` tier policies:

   ```bash
   kubectl apply -f policy/03-app.yaml
   ```

[:arrow_right: Module 7 - Use Observability to Troubleshoot Connectivity Issues](module-7-troubleshooting.md)   <br>

[:arrow_left: Module 5 - Observe traffic flows in Calico Cloud](module-5-secure-pod-traffic.md)

[:leftwards_arrow_with_hook: Back to Main](../README.md) 
