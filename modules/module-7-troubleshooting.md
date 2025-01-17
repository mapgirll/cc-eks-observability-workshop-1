# Module 7 - Use Observability to Troubleshoot Connectivity Issues

## Deploy Hipstershop application and the required security policy

Hipstershop application diagram is shown below:

<img width="1778" alt="hipstershop-arch" src="https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/4b709f14-1dcd-4e62-96e9-c5efbe93be50">


Services and port list:

| Deployment Name         | Label                     | Service Port/Proto  |
| :-----------:           | :-------------:           | :-----------------: |
| adservice               | app=adservice             | 9555/TCP            |
| cartservice             | app=cartservice           | 7070/TCP            |
| checkoutservice         | app=checkoutservice       | 5050/TCP            |
| currencyservice         | app=currencyservice       | 7000/TCP            |
| emailservice            | app=emailservice          | 5000/TCP            |
| frontend                | app=frontend              | 80/TCP              |
| loadgenerator           | app=loadgenerator         |                     |
| paymentservice          | app=paymentservice        | 50051/TCP           |
| productcatalogservice   | app=productcatalogservice | 3550/TCP            |
| recommendationservice   | app=recommendationservice | 8080/TCP            |
| redis-cart              |app=redis-cart             | 6379/TCP            |
| shippingservice         | app=shippingservice       | 50051/TCP           |

1. Create the namespace

   ```bash
   kubectl create namespace hipstershop
   ```

2. Deploy the application Online Boutique (Hipstershop) to the namespace. This will install the application from the Google repository.

    ```bash
    kubectl apply -n hipstershop -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/v0.3.9/release/kubernetes-manifests.yaml
    ```

3. Wait for all PODs to get into a running status.

    ```bash
    kubectl get pods -n hipstershop
    ```

    You should see an output similar to the following.

    ```bash
    NAME                                     READY   STATUS    RESTARTS   AGE
    adservice-6f498fc6c6-c5rhh               1/1     Running   0          2m40s
    cartservice-bc9b949b-rgqpc               1/1     Running   0          2m40s
    checkoutservice-598d5b586d-nxjck         1/1     Running   0          2m41s
    currencyservice-6ddbdd4956-vjzs8         1/1     Running   0          2m40s
    emailservice-68fc78478-qg8qp             1/1     Running   0          2m41s
    frontend-5bd77dd84b-l8qqx                1/1     Running   0          2m41s
    loadgenerator-8f7d5d8d8-d7gwj            1/1     Running   0          2m40s
    paymentservice-584567958d-hr5vl          1/1     Running   0          2m41s
    productcatalogservice-75f4877bf4-jvnlq   1/1     Running   0          2m40s
    recommendationservice-646c88579b-2t55b   1/1     Running   0          2m41s
    redis-cart-5b569cd47-l29tx               1/1     Running   0          2m40s
    shippingservice-79849ddf8-nlfjv          1/1     Running   0          2m40s
    ```

4. Apply the required microsegmentation policies to the ```app``` policy tier for allowing traffic in our zero-trust default-deny model.

   ```bash
   kubectl apply -f policy/04-hipstershop.yaml
   ```

5. Check the service graph for the traffic flows that should look like the following:


   ![no_policy_flow_ob](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/c9f9d038-ce6d-42b3-847c-2e8f2eb0eabd)


## Investigate denied flows using the Calico Enterprise Observability tools (ServiceGraph)

1. Grant executable permissions to the script we will be using

    ```bash
    chmod +x scripts/lab-script.sh
    ```

2. Run the script

    ```bash
    ./lab-script.sh
    ```

3. Type “1” (Demo Break Online Boutique - Dynamic Service and Threat Graph) and press Enter. Then type "99" and hit Enter to exit.

4. Use the ```LoadBalancer``` ```frontend-external``` service to access the service on a browser

    ```bash
    kubectl get svc -n hipstershop
    ```

5. Add any product to cart

6. Click 'Place Order'

7. The request should not complete and return HTTP 500 Internal error.

    ```nano
    HTTP Status: 500 Internal Server Error

    rpc error: code = Internal desc = failed to charge card: could not charge the card: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial tcp 10.100.34.227:50051: i/o timeout"
    ```
   
   ![500_error](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/3a70c230-5de0-41f6-b3ab-d489c3896d75)

8. As indiciated by the screenshot above, the order placement failed due to connectivity issue. Let us use Calico Service Graph to find the root cause of the connectivity issue. Go into the hipstershop namespace in ServiceGraph, where the Online Boutique application is running.

9. From the Online Boutique Diagram, we can see that the CheckoutService will send the request to PaymentService.

10. You should see a red arrow (indicating denied traffic) between the checkoutservice and paymentservice. Hover your mouse over the red/orange line between checkoutservice to see more information about the connection.

   
   ![red_arrow_flow](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/7cebebd8-3774-4733-b9a1-bfb1e60b70a7)

11. By selecting the red arrow, it will show the flows related to that arrow on the bottom of the screen:

   ![select_red_arrow](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/de6c01b9-3549-4fc2-96a0-da7dc2a5b942)

12. Expand one of denied flows from checkoutservice to paymentservice. It will show a comprehensive list of metadata about the connection parties. Scroll down to see the `Action =  Deny` and the `Policies = *“0|security|security.tenant-hipstershop|pass|0, 1|default|default.default-deny|deny|-1”*`. This policy entry means that the flow did not match with any security policy rule configured and it ended up on an implicit default deny.

   ![detailed_flow](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/eb9a7c25-5959-4468-871d-f2ae65dbf0c0)
   
13. As the flow did not match with security policy, let’s review the Security Policy created for the paymentservice. Open `paymentservice` security policy. You see "0" Endpoints is associated with that policy.

   ![no_endpoints](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/3567b2b6-c18a-48b6-a698-afc458b70e46)

14. Labels are the primary means of deploying security policies. Let’s check the labels from the paymentservice pod using the command below (the label can also be checked through the Tigera UI in Endpoint tab).

   ```bash
    kubectl get po -n hipstershop --show-labels | grep payment
   ```

    ```bash
    paymentservice-584567958d-5z8jx          1/1     Running   1 (38h ago)   7d    app=paymentservice,pod-template-hash=584567958d
    ```

15. We can see the label of the payment pod is `app=paymentservice` and the label configured in the security policy is `app=paymentserviceee` which doesn't allow the pod endpoint to get matched by the policy. Therefore there is a typo on the security policy label.

   ![wrong_label](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/e23e8206-97d3-4e09-a91c-538c434fe860)

16. Use the Policy Board UI to fix the wrong label. Make sure the configured label is `app=paymentservice`. You should see the Endpoints in the Security Policy shows “1”.

   ![fix_label](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/aa52ceea-cbe4-440a-80ee-0bd3915e851d)


18. If we click on the “1”, it will show the endpoint secured by the security policy (in this case the paymentservice pod).

   ![click_endpoint](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/e48abba3-4ef6-4c92-8f78-097def8dd53f)

   ![endpoint_details](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/4c420681-2926-45cb-ab4f-11acfa1dd4e5)


19. After fixing the paymentservice label, we should be able to place the order by clicking on “Place Order”.

   ![order_success](https://github.com/tigera-solutions/cc-eks-observability-workshop/assets/117195889/9f04035e-1cde-4d6e-99ae-3ad700893592)
   
20. Also note that the default-deny policy is no longer denying traffic.


[:arrow_right: Module 8 - Cleanup](module-8-cleanup.md)   <br>

[:arrow_left: Module 6 - Zero-trust security for pod traffic](module-6-zero-trust-security.md)

[:leftwards_arrow_with_hook: Back to Main](../README.md) 
