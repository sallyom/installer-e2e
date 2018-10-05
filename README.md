# installer-e2e
This repo is my setup/notes for running openshift/installer through ci-operator.
I use this to launch a cluster in AWS via openshift/installer, and to run CI against the cluster.
There are a few minor additions to the templates in this repo from the template running in CI; 
Look in the template for `#MODIFIED` to see the diffs.  The main diff is that I've added a secret 
to transfer aws credentials and quay pull secret.  

First, get a token and login to the OpenShift CI cluster.
Next: Note the files required for the `cluster-profile-aws secret` and populate their content.
The ssh key-pair is meant to be generated/only used for ci-testing. The private
key is used only when running the full conformance test suite, so that's optional.

Then: 

Create a new project in https://api.ci.openshift.org:
 ```bash
oc new-project your-namespace
```

Fill in contents of  cluster-profile-aws directory with contents of files:

(see files in `cluster-profile-aws` directory for more info)
 ```
   credentials     -see/follow noted format in this repo
   pull-secret     -quay pull secret json config, as a one-liner
   ssh-privatekey  -only required for full conformance tests, not required for aws-e2e tests
   ssh-publickey   -required by installer
```

Now run the ci-operator command.  To get `ci-operator` binary run `make build` from your checkout of [ci-operator](https://github.com/openshift/ci-operator) 
```bash
ci-operator -template templates/cluster-launch-installer-e2e-new.yaml \
               -config /path/to/openshift/release/ci-operator/config/openshift/installer/openshift-installer-master.yaml \
               -git-ref=your-gh-username/installer@your-branch \
               -secret-dir=/path/to/cluster-profile-aws \
               -namespace=your-namespace
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
