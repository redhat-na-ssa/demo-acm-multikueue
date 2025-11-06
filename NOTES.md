## Links

1. ACM with multikueue [instructions](https://github.com/stolostron/foundation-docs/blob/main/guide/ManagedCluster/KueueAddonInstallation.md). Replace the Managed Service Account Addon with the Cluster Proxy Addon. Do NOT use the Managed Service Account Addon.

2. Open Cluster Management with multikueue [instructions](https://github.com/open-cluster-management-io/ocm/tree/main/solutions/kueue-admission-check). Follow the ACM instructions instead.

3. Multikueue Standalone [instructions](https://kueue.sigs.k8s.io/docs/tasks/manage/setup_multikueue/#multikueue-specific-kubeconfig). If all fails, configure multikueue manually. The upstream Kueue and Red Hat Build of Kueue both activate Multikueue by default in the current release (as of 11/2025).

## Useful Tips

1. ACM/OCM requires clusters to have valid API certs. The `AWS with Open Environment` clusters do NOT issue valid API certs. You can use the `RHOAI on OCP on AWS with NVIDIA GPUs` [catalog](https://catalog.demo.redhat.com/catalog/babylon-catalog-prod/order/sandboxes-gpte.ocp4-demo-rhods-nvidia-gpu-aws.prod) that has the option for a Cert Manager API. Otherwise, you will need to issue certs manually.

For example, using [acme.sh](acme.sh)

> Note: It is not recommended to use your AWS root account keys. You can create a separate IAM user or role with the following policy

```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:GetHostedZone",
                "route53:ListHostedZones",
                "route53:ListHostedZonesByName",
                "route53:GetHostedZoneCount",
                "route53:ChangeResourceRecordSets",
                "route53:ListResourceRecordSets"
            ],
            "Resource": "*"
        }
    ]
}
```

```bash
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_KEY_ACCESS_KEY=
export LE_API=$(oc whoami --show-server | cut -f 2 -d ':' | cut -f 3 -d '/' | sed 's/-api././')
export CERTDIR=$HOME/certs
mkdir -p $CERTDIR

./acme.sh/acme.sh --issue -d ${LE_API} --dns dns_aws

./acme.sh/acme.sh --install-cert -d ${LE_API} --cert-file ${CERTDIR}/cert.pem --key-file ${CERTDIR}/key.pem --fullchain-file ${CERTDIR}/fullchain.pem --ca-file ${CERTDIR}/ca.cer

oc create secret tls api-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/key.pem -n openshift-config

oc patch apiserver cluster --type merge --patch="{\"spec\": {\"servingCerts\": {\"namedCertificates\": [ { \"names\": [  \"$LE_API\"  ], \"servingCertificate\": {\"name\": \"api-certs\" }}]}}}"
```

2. Managed Service Account Addon

The `kueue-addon` will generate a `multikueue` secret in the spoke clusters namespace. This secret will contain a `kubeconfig` to authenticate to the spoke clusters. Currently, the self signed in-cluster CA is  added to this `kubeconfig` and will result in a x509 invalid cert error.

> TODO: File a bug

3. OCM

This is specific to OCM (not ACM). When you run `clusteradm join` on the spoke cluster, a managed cluster resource is generated on the hub cluster. The managed cluste resource will NOT have the Managed Cluster URL. This URL is required for the Managed Service Account Addon to generate the proper kubeconfig. To fix this, 

```bash
oc edit mcl <cluster-name>

...
    hubAcceptsClient:
    leaseDurationSeconds:
    managedClusterClientConfigs:
      url:
...
```


