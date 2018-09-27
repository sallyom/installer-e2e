# installer-e2e
This repo is my setup/notes for running openshift/installer through ci-operator.
I use this to launch a cluster in AWS via openshift/installer, and to run CI against the cluster.
There are a few minor additions to the templates in this repo from those running in CI; I've 
added a secret to transfer aws credentials and quay pull secret.  

First, get a token and login to the OpenShift CI cluster.
Next: Note the files required for the cluster-secrets-aws secret and populate their content.
The ssh key-pair is meant to be generated/only used for ci-testing. The private
key is used only when running the full conformance test suite, so that's optional.

Pick a short namespace name to avoid:

> the S3 bucket name
> "wking-next-gen-installer-ef260.origin-ci-int-aws.dev.rhcloud.com",
> generated from the cluster name and base domain, is too long; S3
> bucket names must be less than 63 characters; please choose a
> shorter cluster name or base domain

It looks like 22 characters is the max with the
`${NAMESPACE}-${JOB_NAME_HASH}` approach to cluster naming.

Then:

Create a new project in https://api.ci.openshift.org:
 ```bash
oc new-project your-namespace
```

Fill in contents of  cluster-secrets-aws directory with contents of files:

(see files in `cluster-secrets-aws` directory for more info)
 ```bash
   credentials (see noted format in this repo)
   pull-secret (quay pull secret json config)
   ssh-privatekey (only required for full conformance tests, not required for aws-e2e tests)
   ssh-publickey (required by installer)
```

Now run the ci-operator command: 
```bash
ci-operator -template templates/cluster-launch-installer-e2e-new.yaml \
               -config /path/to/openshift/release/ci-operator/config/openshift/installer/master.yaml \
               -git-ref=your-gh-username/installer@your-branch \
               -secret-dir=/path/to/cluster-secrets-aws \
               -namespace=your-namespace
```


You can then access your project in the CI web console, check the pods.

When running the full test suite, conformance test output will be found via
pod logs for pod/dev container/test.  

You can grab the kubeconfig, copy to your local system, and access the cluster running in AWS.
From the CI web console in your project, go to pod/dev container/setup and in the terminal, 
kubeconfig is at /tmp/shared/cluster/generated/auth/kubeconfig.  Copy that to your local system, then
`export KUBECONFIG=/path/to/copied/kubeconfig`.

Clean up your AWS resources from a cluster!!!
1. `git clone git@github.com:openshift/hive.git`
2. `cd hive; make hiveutil`
3. `bin/hiveutil aws-tag-deprovision --cluster-name your-cluster-name --loglevel debug tectonicClusterID=see-aws-console-for-tag`
