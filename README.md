# Cilium Zero Trust Deployment
By default kubernetes allows all pods to comunicate with each other to the outside world. This is convenient but exposes a potential fault as a compromisied workload can reach databases, internal APIs and external services with nothing in is way. Cilium can be used to prevent this by adding a layer of security between workloads and the outside world.

## Initial setup
This project deploys [Google's Online Boutique microservice](https://github.com/GoogleCloudPlatform/microservices-demo) and uses Cilium to secure the communication between the services. Cilium is deployed on a 3-node Kind cluster with L2 announcement and observability enabled through hubble. You can find the deployemnt here [cilium-values](./cilium-values.yaml).

The microservice is deployed into the `boutique` namespace. It's architecture is as follows:
[![architecture](./assets/architecture-diagram.png)](./assets/architecture-diagram.png)

It's hubble dashboard at this stage closely mirrors the architecture diagram.

[![hubble](./assets/Initial%20screen.png)](./assets/Initial%20screen.png)

## Securing Communication
From the architecture we can develop a call graph of what pods should be able to communicate with each other. 

[![mental-model](./assets/boutique-egress-modules.png)](./assets/boutique-egress-modules.png)

But currently each pod can communicate freely with any other pod, even pods they have no business with. For instance by sshing into the payment pod we can access the ad service:

[![ssh](./assets/free-communication.png)](./assets/free-communication.png)

```log
Oct 28 14:40:15.276: boutique/paymentservice-78f6cfdf7f-jx25l:40287 (ID:63978) -> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) to-endpoint FORWARDED (TCP Flags: SYN)
Oct 28 14:40:15.276: boutique/paymentservice-78f6cfdf7f-jx25l:40287 (ID:63978) -> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) to-endpoint FORWARDED (TCP Flags: ACK)
Oct 28 14:40:15.276: boutique/paymentservice-78f6cfdf7f-jx25l:40287 (ID:63978) -> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) to-endpoint FORWARDED (TCP Flags: ACK, FIN)
Oct 28 14:40:15.757: boutique/paymentservice-78f6cfdf7f-jx25l:40287 (ID:63978) -> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) to-endpoint FORWARDED (TCP Flags: RST)
```

As we can see from the logs request from the payment service is successfully forwarded to the ad service.

### Denying communication
We set up a cluster wide policy to deny egress from all pods not within the `kube-system` namespace. It is important to note that the pods still have DNS access. The policy is defined in the [deny-egress.yaml](./default-deny.yaml) file.
The architecture now looks like this: 

[![architecture](./assets/deny-egress.png)](./assets/deny-egress.png)

```log
Oct 28 14:56:45.927: boutique/paymentservice-78f6cfdf7f-jx25l:40909 (ID:63978) <> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) policy-verdict:none TRAFFIC_DIRECTION_UNKNOWN DENIED (TCP Flags: SYN)
Oct 28 14:56:45.927: boutique/paymentservice-78f6cfdf7f-jx25l:40909 (ID:63978) <> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) Policy denied DROPPED (TCP Flags: SYN)
Oct 28 14:56:46.942: boutique/paymentservice-78f6cfdf7f-jx25l:40909 (ID:63978) <> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) policy-verdict:none TRAFFIC_DIRECTION_UNKNOWN DENIED (TCP Flags: SYN)
Oct 28 14:56:46.942: boutique/paymentservice-78f6cfdf7f-jx25l:40909 (ID:63978) <> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) Policy denied DROPPED (TCP Flags: SYN)
Oct 28 14:56:47.966: boutique/paymentservice-78f6cfdf7f-jx25l:40909 (ID:63978) <> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) policy-verdict:none TRAFFIC_DIRECTION_UNKNOWN DENIED (TCP Flags: SYN)
Oct 28 14:56:47.966: boutique/paymentservice-78f6cfdf7f-jx25l:40909 (ID:63978) <> boutique/adservice-6bfc5fc46c-vqw8z:9555 (ID:60129) Policy denied DROPPED (TCP Flags: SYN)
```

We can see a fuller vision of the communication state through hubble.

[![hubble](./assets/deny-egress-hubble.png)](./assets/deny-egress-hubble.png)

We can now deploy the egress rules in according to the aforementioned call graph. This can be done using the [egress-network-policies.yaml](./egress-network-policies.yaml) file.

We can test the communication by calling the payment and email service from checkout. We can see the requests are approved by the policy.
```logs
Oct 28 15:55:38.369: boutique/checkoutservice-5d5f4589c5-pvh48:36306 (ID:40344) -> boutique/emailservice-558899d44c-lw6j7:8080 (ID:14088) to-endpoint FORWARDED (TCP Flags: ACK)
Oct 28 15:55:51.475: boutique/checkoutservice-5d5f4589c5-pvh48:36306 (ID:40344) -> boutique/emailservice-558899d44c-lw6j7:8080 (ID:14088) to-endpoint FORWARDED (TCP Flags: ACK)
Oct 28 15:56:00.182: boutique/checkoutservice-5d5f4589c5-pvh48:36306 (ID:40344) -> boutique/emailservice-558899d44c-lw6j7:8080 (ID:14088) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:23.897: boutique/checkoutservice-5d5f4589c5-pvh48:36306 (ID:40344) -> boutique/emailservice-558899d44c-lw6j7:8080 (ID:14088) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:30.131: boutique/checkoutservice-5d5f4589c5-pvh48:36306 (ID:40344) -> boutique/emailservice-558899d44c-lw6j7:8080 (ID:14088) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:48.966: boutique/checkoutservice-5d5f4589c5-pvh48:36306 (ID:40344) -> boutique/emailservice-558899d44c-lw6j7:8080 (ID:14088) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
----
Oct 28 15:54:54.474: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:55:28.244: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:55:39.387: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-overlay FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:55:39.387: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:00.180: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-overlay FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:00.180: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:23.895: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-overlay FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:23.895: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:30.129: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-overlay FORWARDED (TCP Flags: ACK, PSH)
Oct 28 15:56:30.129: boutique/checkoutservice-5d5f4589c5-pvh48:58206 (ID:40344) -> boutique/paymentservice-78f6cfdf7f-jx25l:50051 (ID:63978) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
```