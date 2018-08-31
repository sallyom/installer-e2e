# installer-e2e
This repo is my setup/notes for running openshift/installer through ci-operator.
I use this to launch a cluster in AWS via openshift/installer, and to run CI against the cluster.
There are a few minor additions to the templates in this repo from those running in CI; I've 
added a secret to transfer aws credentials, quay pull secret and the tectonic config.  

First, get a token and login to the OpenShift CI cluster.
Next: Note the files required for the cluster-secrets-aws secret and create them/change paths
accordingly.  The ssh key-pair is optional, and is required for OpenShift conformance tests to 
pass.

Then: 
1. oc new-project my-namespace
2. oc create secret generic dev-cluster-profile --from-file=aws/openshift.yaml -n your-namespace
3. oc create secret -n your-namespace generic cluster-secrets-aws \
                    --from-file=secret/credentials \
                    --from-file=secret/pull-secret \
                    --from-file=secret/license \
                    --from-file=secret/ssh-privatekey \
                    --from-file=secret/ssh-publickey \
                    -o yaml --dry-run | oc -n your-namespace apply -f -
4. ci-operator -template templates/cluster-launch-installer-e2e.yaml \
               -config /path/to/openshift/release/ci-operator/config/openshift/installer/master.json \
               -secret-dir=aws \
               -secret-dir=secret \
               -git-ref=your-gh-username/installer@your-branch \
               -namespace=your-namespace


You can then access your project in the CI web console, check the pods.
When running the smoke tests, you'll see the output of the smoke tests in 
logs of pod/dev container/setup.

When running the full test suite, conformance test output will be found via
pod logs for pod/dev container/test.  

You can grab the kubeconfig, copy to your local system, and access the cluster running in AWS.
From the CI web console in your project, go to pod/dev container/setup and in the terminal, 
kubeconfig is at /tmp/shared/cluster/generated/auth/kubeconfig.  Copy that to your local system, then
export KUBECONFIG=/path/to/copied/kubeconfig.

It's a real PITA to clean up your AWS resources from a cluster started in this way, so be prepared
to manually clean things up (or create a script to do so, I have not yet).
Resources you'll need to manage/delete:
- ec2 instances
- route53 record sets and hosted zones
- load balancers
- s3 bucket
- VPCs and then after deletion release all EIP addresses

IAM roles/IAM resources will also need to be deleted (generally you won't have permission to do that).
