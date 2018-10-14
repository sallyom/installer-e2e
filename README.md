# installer-e2e
This repo is my setup/notes for launching openshift/installer through ci-operator running [here](https://api.ci.openshift.org).
This results in a 3 master 3 worker cluster running the the AWS account for which you supply credentials.

Whatever repository you are testing with the installer, you will pass this config file to ci-operator:

`openshift/release/ci-operator/config/openshift/the-repo/the-repo-master.yaml`

See [any origin release image repository that currently runs the CI job aws-e2e](https://github.com/openshift/release/tree/master/ci-operator/config/openshift) to find the right config file to pass.


There are a few minor additions to the `cluster-launch-installer-e2e.yaml` template in this repo from the template running in CI; 
Look in `templates/cluster-launch-installer-e2e-modified.yaml` for `#MODIFIED` to see the diffs.  The main diff is that I've added a secret 
to transfer aws credentials and quay pull secret.  Also, I've commented out the teardown container, because usually I want to work with 
the cluster once it's launched.  You can uncomment the teardown container to have your cluster auto-destroyed upon a launch (like in CI).
Or, use the `hiveutil aws-tag-deprovision` cleanup command given below to destroy your cluster otherwise.  

First, get a token and `oc login` to [the OpenShift CI cluster](https://api.ci.openshift.org).

Next: Note the files required for the `cluster-profile-aws secret` and populate their content.
The ssh key-pair is meant to be generated/only used for ci-testing. The private
key is used only when running the full conformance test suite, that is optional.

Fill in contents of a directory named `cluster-profile-aws` <-important as this dir name is the secret name, with contents of files:

 ```
   credentials     -see/follow noted format in this repo
   pull-secret     -quay pull secret json config, as a one-liner
   ssh-privatekey  -only required for full conformance tests, not required for aws-e2e tests
   ssh-publickey   -required by installer
```

Then: 

Edit from this repository `templates/cluster-launch-installer-e2e-modified.yaml` the `"your-ns"` to a value of your choice.
The namespace translates to the base of the cluster-name parameter.  Look in AWS console or ci-operator output for full cluster-name.
Also, note the `TEST_COMMAND` value and change that accordingly.  You can find an appropriate value for that in the release repo, for 
example, with the `openshift-cluster-dns-operator` job, [here](https://github.com/openshift/release/blob/master/ci-operator/jobs/openshift/cluster-dns-operator/openshift-cluster-dns-operator-master-presubmits.yaml#L32-L#L34)

Now run the ci-operator command.  To get `ci-operator` binary run `make build` from your checkout of [ci-operator](https://github.com/openshift/ci-operator) 
```bash
ci-operator -template templates/cluster-launch-installer-e2e-modified.yaml \
               -config /path/to/openshift/release/ci-operator/config/openshift/whatever-release-repo/whatever-release-repo-master.yaml \
               -git-ref=your-gh-username/whatever-release-repo@your-branch \
               -secret-dir=/path/to/cluster-profile-aws \
               -namespace=the-ns-you-filled-in-the-template-above
```


You can then access your project in the CI web console, check the pods.

Useful output will be found via pod logs for `pod/dev container setup` for the cluster launch, 
and pod logs for `pod/dev container/test` for all test output.

You can `grab the kubeconfig`, copy to your local system, and access the cluster running in AWS.
From the CI web console in your project, `go to pod/dev container/test terminal`, 
kubeconfig is at `/tmp/artifacts/installer/auth/kubeconfig`.  Copy that to your local system, then
`export KUBECONFIG=/path/to/copied/kubeconfig`.

Clean up your AWS resources from a cluster!!!
1. `git clone git@github.com:openshift/hive.git`
2. `cd hive; make hiveutil`
3. `bin/hiveutil aws-tag-deprovision --cluster-name your-cluster-name --loglevel debug tectonicClusterID=see-aws-console-for-tag`
