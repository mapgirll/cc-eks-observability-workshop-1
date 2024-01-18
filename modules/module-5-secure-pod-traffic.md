# Module 5 - Secure pod traffic using Calico Policy Recommendations

In this module, we will focus on securing the pod traffic using the Calico Policy Recommendations as a baseline and understanding how to visualize and troubleshoot any unintended behaviour. 

> **Note:** An important consideration here is that the policy recommender cannot work properly if a global deny policy is already applied as it needs to be able to look at the accepted flows to determine the behaviour in a certain time range.

As the Stars application is comprised of microservices in three namespaces, we will use policy recommendations to create namespace-scoped policies for each namespace one by one and then tweak policies if needed.

## Policy Recommendation Steps

1. Select the correct cluster context on the top right, then in the left hamburger menu click on ```Policies > Recommendations``` on the top right. If it's not already enabled, enable it.
</br>

2. If no policy recommendations are showing edit the Global Settings. As this is an instructional environment we can reduce the stabilization period and processing interval. Policies should start to appear, and may be ```Processing```.
</br>

3. When the Policy Recommendations are ```Ready``` click ```Actions``` for the client policy and ```Add to Policy Board```. This puts the policy in a preview-only (staged) mode and doesn't enforce the policy.
   
4.  Go to the Policy Board (left hand menu > Policies > Policies) and you should see a namespace-isolation tier, with the staged client policy.
    
5. Go back to Policy Recommendations and repeat the steps for the ```management-ui```, ```yaobank``` and the ```stars``` namespaces.

6. The policy board should show all of the staged policies and already we see there is some traffic being allowed. 


[:arrow_right: Module 6 - Zero-trust security for pod traffic](module-6-zero-trust-security.md)   <br>

[:arrow_left: Module 4 - Observe traffic flows in Calico Cloud](module-4-observe-traffic.md)

[:leftwards_arrow_with_hook: Back to Main](../README.md) 
