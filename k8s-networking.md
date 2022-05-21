1. K8s pod having issues hitting AWS metadata service `can't connect to remote host (169.254.169.254): Connection refused`

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

Error could be like below:

```
could not describe instances: NoCredentialProviders: no valid providers in chain. Deprecated.\n\tFor verbose messaging see aws.Config.CredentialsChainVerboseErrors
```

The problem would be that a kube2iam pod ran on the node and then it was removed. Kube2iam write an iptable rules to forward all the metadata request to it's pod running with `hostNetowork: true`. If the pod is removed it will not automatically clean up the iptables rule it created.

This will cause all of the calls to EC2 metadata to go nowhere. You will have to ssh into each node individually and remove the iptable rule yourself.

First list the iptable rules to find the one set up by kube2iam,

```
 iptables -t nat -S PREROUTING | grep  169.254.169.254 
```

The output should be similar to

```
-A PREROUTING -d 169.254.169.254/32 -i weave -p tcp -m tcp --dport 80 -j DNAT --to-destination 10.0.101.101:8181 
```

You may see multiple results if you deployed the agent with different --host-interface options on accident. You can delete them one at a time. To delete a rule, use the -D option of iptables and specify the entire line of output from above after -A. For example:

```
 iptables -t nat -D PREROUTING -d 169.254.169.254/32 -i weave -p tcp -m tcp --dport 80 -j DNAT --to-destination 10.0.101.101:8181 
```

As this is done on each node, EC2 metadata requests will no longer go to kube2iam.

Reference: 

1. https://www.bluematador.com/blog/iam-access-in-kubernetes-installing-kube2iam-in-production

