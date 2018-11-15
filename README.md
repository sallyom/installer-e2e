# installer-e2e

This repo is my setup/notes for launching openshift clusters through ci-operator running [here](https://api.ci.openshift.org).

First, get a token and `oc login` to [the OpenShift CI cluster](https://api.ci.openshift.org).

## Modified templates to choose from:

Templates in this repository are kept in sync with templates in `release/ci-operator/templates` with minor modifications. 

1.  `cluster-launch-installer-e2e-modified.yaml template`:

This results in a 3 master 3 worker cluster running in the AWS|OpenStack account for which you supply credentials.
There are a few minor additions to the `cluster-launch-installer-e2e.yaml` template in this repo from the template running in CI; 
Look in `templates/cluster-launch-installer-e2e-modified.yaml` for `#MODIFIED` to see the diffs.  The main diff is there is a secret added
to transfer aws credentials and quay pull secret.  Also, I've commented out the teardown container, because usually I want to work with 
the cluster once it's launched.  You can uncomment the teardown container to have your cluster auto-destroyed upon a launch (like in CI).
Or, use the `hiveutil aws-tag-deprovision` cleanup command given below to destroy your cluster otherwise.  
```
Clean up your AWS resources from a cluster!!!

*  git clone git@github.com:openshift/hive.git
*  cd hive; make hiveutil
*  bin/hiveutil aws-tag-deprovision --cluster-name your-cluster-name --loglevel debug tectonicClusterID=see-aws-console-for-tag
```

2.  `nested-libvirt-installer-e2e-modified.yaml template`: 

This results in a 1 master 1 worker libvirt cluster running nested in a GCE instance in project `openshift-gce-devel` for which you supply credentials.
There are a few minor additions to the `nested-libvirt-installer-e2e.yaml` template in this repo from the template running in CI; 
Look in `templates/nested-libvirt-installer-e2e-modified.yaml` for `#MODIFIED` to see the diffs.  The main diff is there is a secret added
to transfer gce credentials and quay pull secret.  The GCE instance is torn down upon completion.  Comment out the teardown container if you need this 
instance/cluster to stay running.


3.  `cluster-launch-src-modified.yaml template`: 

This results in a cluster launched via ansible playbook (pre 4.0 cluster) with the AWS account for which you supply credentials.
The cluster will be deprovisioned upon completion.  Comment out the teardown container if you need this cluster to stay running.

## ci-operator config file
Whatever repository you are testing with the installer, you will pass this config file to ci-operator:

`openshift/release/ci-operator/config/openshift/the-repo/the-repo-master.yaml`

See [any origin release image repository that currently runs the CI job aws-e2e](https://github.com/openshift/release/tree/master/ci-operator/config/openshift) to find the right config file to pass.



## Secret to pass cloud credentials

Note the files required for the `cluster-profile-aws`, `cluster-profile-libvirt`,
or `cluster-profile-openstack` secret and populate their content.
The ssh key-pair is meant to be generated/only used for ci-testing. The private
key is used only when running the full conformance test suite, that is optional.

Fill in contents of a directory named `cluster-profile-CLUSTER_TYPE` <-this dir name is the secret name, with contents of files:

when CLUSTER_TYPE=aws:
 ```
   credentials     -see/follow noted format in this repo
   pull-secret     -quay pull secret json config, as a one-liner
   ssh-privatekey  -only required for full conformance tests, not required for aws-e2e tests
   ssh-publickey   -required by installer
```

when CLUSTER_TYPE=openstack:
 ```
   clouds.yaml     -contents of clouds.yaml file that holds openstack credentials
```

when CLUSTER_TYPE=libvirt:
 ```
   pull-secret     -quay pull secret json, as one-liner
   gce.json        -gce credentials file - since libvirt cluster will be launched within a gce instance
```

## Finally, you're ready to run ci-operator: 

Edit from this repository `templates/cluster-launch-installer-e2e-modified.yaml` or `templates/nested-libvirt-installer-e2e-modified.yaml` or 
`templates/cluster-launch-src.yaml` the `"your-ns"` to a value of your choice.
The namespace translates to the base of the cluster-name parameter or instance_prefix.  Look in AWS console or ci-operator output for full cluster-name.
Also, note the `TEST_COMMAND` value and change that accordingly.  You can find an appropriate value for that in the release repo, for 
example, with the `openshift-cluster-dns-operator` e2e-aws job,
[here](https://github.com/openshift/release/blob/master/ci-operator/jobs/openshift/cluster-dns-operator/openshift-cluster-dns-operator-master-presubmits.yaml#L32-#L34)

Now run the ci-operator command.  To get `ci-operator` binary run `make build` from your checkout of [ci-operator](https://github.com/openshift/ci-operator) 

```console
$ ci-operator -template templates/cluster-launch-installer-e2e-modified.yaml|nested-libvirt-installer-e2e-modified.yaml|cluster-launch-src-modified.yaml \
>  -config /path/to/openshift/release/ci-operator/config/openshift/whatever-release-repo/whatever-release-repo-master.yaml \
>  -git-ref=your-gh-username/whatever-repo@your-branch \
>  -secret-dir=/path/to/cluster-profile-${CLUSTER_TYPE} \
>  -namespace=the-ns-you-filled-in-the-template-above
```


You can then access your project in the CI web console, check the pods.

Useful output will be found via pod logs for `pod/dev container setup` for the cluster launch, 
and pod logs for `pod/dev container/test` for all test output.

You can `grab the kubeconfig`, copy to your local system, and access the cluster running in AWS.
From the CI web console in your project, `go to pod/dev container/test terminal`, 
kubeconfig is at `/tmp/artifacts/installer/auth/kubeconfig`.  Copy that to your local system, then
`export KUBECONFIG=/path/to/copied/kubeconfig`.

