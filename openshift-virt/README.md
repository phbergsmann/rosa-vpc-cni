# Test OpenShift Virtualization

Validates OpenShift Virtualization compatibility with AWS VPC CNI: operator install, VM creation, VPC CNI verification, and live migration.

## Prerequisites

- ROSA HCP cluster with AWS VPC CNI installed (see root [README](../README.md))
- `oc` CLI connected to the cluster
- `rosa` CLI authenticated
- `virtctl` CLI installed

## 0. Create Bare Metal Machine Pool

```bash
rosa create machine-pool -c $CLUSTER_NAME --name bm --replicas=1 --instance-type c5n.metal
```

## 1. Fix Security Group for VPC CNI

The default ROSA security group only allows overlay tunnel traffic (Geneve/VXLAN) between nodes. With VPC CNI, pod-to-pod traffic flows directly between ENIs and requires explicit TCP/UDP rules. Without this fix, Konnectivity agents cannot reach webhook pods cross-node, causing all OpenShift Virtualization webhook calls to fail.

```bash
# Get the default ROSA security group
SG_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:api.openshift.com/name,Values=$CLUSTER_NAME" \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text --region $AWS_REGION)

# Allow all TCP traffic within the security group (required for VPC CNI)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 0-65535 \
  --source-group $SG_ID \
  --region $AWS_REGION
```

## 2. Apply IP Forward Tuning Config to Bare Metal Pool

The bare metal nodes need the same `ip-forward` tuning config as the workers pool for VPC CNI to function:

```bash
rosa edit machinepool -c $CLUSTER_NAME bm --tuning-configs ip-forward
```

## 3. Create StorageClass (before operator instance)

Set up the io2 StorageClass as default so all HyperConverged images (golden images, CDI) use it:

```bash
# Remove default from gp3-csi
oc patch storageclass gp3-csi -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'

# Apply ebs-sc as default
oc apply -k 03-storage/
```

## 4. Install OpenShift Virtualization Operator

```bash
oc apply -k 01-operators/
```

## 5. Wait for CRD

```bash
oc wait --for=condition=Established crd/hyperconvergeds.hco.kubevirt.io --timeout=300s
```

## 6. Deploy HyperConverged CR

```bash
oc apply -k 02-custom-resources/
```

Wait for OpenShift Virtualization to be fully available:

```bash
oc wait hyperconverged kubevirt-hyperconverged -n openshift-cnv \
  --for=condition=Available --timeout=600s
```

## 7. Patch StorageProfile

```bash
oc patch storageprofile ebs-sc --type json \
  -p '[{"op": "replace", "path": "/spec", "value": {"claimPropertySets": [{"accessModes": ["ReadWriteMany"], "volumeMode": "Block"}]}}]'
```

## 8. Create VM

```bash
oc apply -k 04-vm/

oc wait --for=jsonpath='{.status.ready}'=true vm/vm-test-vpcni -n vpcni-virt-test --timeout=600s
```

## 9. Verify VPC CNI is used by the VM

```bash
# Get virt-launcher pod IP -- must be in VPC CIDR (e.g. 10.0.x.x), NOT overlay (10.128.x.x)
oc get pod -n vpcni-virt-test -l kubevirt.io/domain=vm-test-vpcni -o wide

# Confirm aws-node runs on the same node as the VM
NODE=$(oc get pod -n vpcni-virt-test -l kubevirt.io/domain=vm-test-vpcni -o jsonpath='{.items[0].spec.nodeName}')
oc get pod -n kube-system -l app.kubernetes.io/name=aws-node --field-selector spec.nodeName=$NODE

# Connectivity test from a separate pod
POD_IP=$(oc get pod -n vpcni-virt-test -l kubevirt.io/domain=vm-test-vpcni -o jsonpath='{.items[0].status.podIP}')
oc run test-vpcni --image=nicolaka/netshoot --restart=Never -- ping -c 5 $POD_IP
sleep 10
oc logs test-vpcni
oc delete pod test-vpcni
```

## 10. Test Live Migration

```bash
virtctl migrate vm-test-vpcni -n vpcni-virt-test

# Watch migration status
oc get vmim -n vpcni-virt-test -w

# Verify VM moved to a different node
oc get pod -n vpcni-virt-test -l kubevirt.io/domain=vm-test-vpcni -o wide
```

## 11. Cleanup

Following the [Red Hat uninstall procedure](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/virtualization/installing#uninstalling-virt):

```bash
# Delete VM and namespace
oc delete -k 04-vm/

# Delete StorageClass
oc delete -k 03-storage/

# Delete HyperConverged CR
oc delete HyperConverged kubevirt-hyperconverged -n openshift-cnv

# Delete Subscription
oc delete subscription hco-operatorhub -n openshift-cnv

# Delete CSV
oc delete csv -n openshift-cnv -l operators.coreos.com/kubevirt-hyperconverged.openshift-cnv

# Delete namespace
oc delete namespace openshift-cnv

# Delete CRDs
oc delete crd -l operators.coreos.com/kubevirt-hyperconverged.openshift-cnv

# Restore gp3-csi as default
oc patch storageclass gp3-csi -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```
