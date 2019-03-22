---
layout: post
title: Building & Running K8s from source
tags: [k8s]
comments: true
---


[http://buildingk8s.com](http://buildingk8s.com/)

Authors:  [ianchak@google.com](mailto:ianchak@google.com),  [mtaufen@google.com](mailto:mtaufen@google.com),  [silviabear@google.com](mailto:silviabear@google.com),  [fbongiovanni@google.com](mailto:fbongiovanni@google.com)

# Prerequisites

-   A computer with a connection to the Internet and a web browser.
    -   A temporary Google Cloud account will be distributed for use at the session.
-   A GitHub account, created prior to the session.
-   Agree to CNCF code of conduct and sign CLA, prior to the session.

# Outline

## Building Kubernetes and Creating a Local Cluster

-   Create a VM in Google Cloud Console
-   Install Dependencies and Clone Kubernetes
-   Build Kubernetes
-   Run a Local Kubernetes Cluster

## Contributing to Open-Source Kubernetes

-   Kubernetes Community Prerequisites
-   Fork Kubernetes on GitHub
-   Change the Kubernetes Source Code
-   Run Kubernetes Unit Tests Locally
-   Contribute a Pull-Request

# Building Kubernetes and Creating a Local Cluster

## Create a VM in Google Cloud Console,  [console.cloud.google.com](http://console.cloud.google.com/)

Navigate to  `Compute Engine`  >  `VM instances`  and click the  `CREATE INSTANCE`  button. On smaller displays this button appears as a  `+`  sign in the menu at the top of the page.

Setup option below when you create the VM instance:

-   Machine type: n1-standard-16 (~15m build time)
    -   Minimum: n1-standard-8 (~20m build time)
    -   To ensure sufficient resources for building and running a cluster.
-   Boot disk size: 50 GB or greater
-   Boot disk image: Google Drawfork CentOS 7

We recommend the setup option below:

-   Region: us-west1 (Oregon), Zones: us-west1-b, us-west1-c, us-west1-d
    -   May have short network latency for this session.

### Cloud console quick VM start command:

```
gcloud compute instances create k8s-build \
--zone us-west1-b \
--image-family centos-7 \
--image-project centos-cloud \
--machine-type n1-standard-16 \
--boot-disk-size=50GB

```

The steps in the next session should be run in the VM over an ssh session.
To start the session, navigate to  `Compute Engine`  >  `VM instances`, and click the  `SSH`  button next to the instance you just created.

See  [https://cloud.google.com/compute/docs/instances/connecting-to-instance](https://cloud.google.com/compute/docs/instances/connecting-to-instance)  for additional details.

## Install Dependencies and Clone Kubernetes

```
sudo yum -y update

# install dependencies
sudo yum -y install yum-utils \
net-tools \
device-mapper-persistent-data \
lvm2 \
gcc \
git

# add docker package repo
sudo yum-config-manager \
--add-repo \
https://download.docker.com/linux/centos/docker-ce.repo

# install docker-ce
sudo yum -y install docker-ce

# cleanup
sudo yum -y clean all

# install golang-11
curl -O https://dl.google.com/go/go1.11.2.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.11.2.linux-amd64.tar.gz
rm -f go1.11.2.linux-amd64.tar.gz

# set up environment via bash profile

echo 'export PATH=${PATH}:/usr/local/go/bin' >> ~/.bash_profile
echo 'export GOPATH_K8S=${HOME}/go/src/k8s.io/kubernetes' >> ~/.bash_profile
echo 'export PATH=${GOPATH_K8S}/third_party/etcd:${PATH}' >> ~/.bash_profile
source ~/.bash_profile

# clone kubernetes
mkdir -p ${GOPATH_K8S}
git clone https://github.com/kubernetes/kubernetes ${GOPATH_K8S}
cd ${GOPATH_K8S}
git remote rename origin upstream

# install etcd
hack/install-etcd.sh

# add your user to the docker group, so sudo isn't required for docker commands
sudo usermod -a -G docker ${USER}

# start the docker daemon
sudo systemctl enable docker
sudo systemctl start docker


```

**Logout and login again before proceeding, so that docker usermod takes effect.**

## Build Kubernetes

```
# build kubernetes (this step takes roughly 15 minutes on n1-standard-16)
cd ${GOPATH_K8S}
git checkout v1.12.3
time make quick-release

```

## Run a Local Kubernetes Cluster

Start the cluster:

```
${GOPATH_K8S}/hack/local-up-cluster.sh

```

Verify the cluster is running in another ssh window:

```
export KUBERNETES_PROVIDER=local
export KUBECONFIG=/var/run/kubernetes/admin.kubeconfig
${GOPATH_K8S}/cluster/kubectl.sh get nodes

```

# Contributing to Open-Source Kubernetes

## Kubernetes Community Prerequisites

Please note that the following items are required before your PR can be accepted by the community. Please complete these on your own time, before/after the session, if you wish to send a PR.

-   Read and agree to the Code of Conduct:  [https://github.com/cncf/foundation/blob/master/code-of-conduct.md](https://github.com/cncf/foundation/blob/master/code-of-conduct.md)
-   Sign the CLA. Instructions:  [https://github.com/kubernetes/community/blob/master/CLA.md](https://github.com/kubernetes/community/blob/master/CLA.md)

## Fork Kubernetes on GitHub

1.  Navigate to  [https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)
2.  Click the “Fork” button in the upper-right corner and follow the on-screen instructions.

## Setup GitHub username

You will need to provide [YOUR_GITHUB_USER_NAME]. You can export it as a bash variable, and add it to your bash profile.

```
export GITHUB_USER=[INSERT_YOUR_GITHUB_USER_NAME]
echo 'export GITHUB_USER=${GITHUB_USER}' >> ~/.bash_profile

```

## Change the Kubernetes Source Code

Navigate to the directory containing your clone of Kubernetes and add your fork:

```
cd ${GOPATH_K8S}
git remote add origin https://github.com/${GITHUB_USER}/kubernetes
git fetch origin

```

These instructions make the following assumptions:

-   Your  `upstream`  remote points to  `https://github.com/kubernetes/kubernetes`.
-   Your  `origin`  remote points to  `https://github.com/${GITHUB_USER}/kubernetes`.

You can double-check by running:

```
git remote -v

```

Create a branch before making a change:

```
git checkout upstream/master
git checkout -b my-branch-of-k8s

```

If you would like to make a change, we recommend finding some untested code and adding a few unit tests. If you don’t know how to write unit tests in Go, read this:  [https://golang.org/doc/code.html#Testing](https://golang.org/doc/code.html#Testing).

## Example new test:

File: pkg/util/normalizer/normalizer_test.go

```
package normalizer

import (
	"testing"
)

func TestLongDesc(t *testing.T) {
  s := ""
  if r := LongDesc(s); r != s {
    t.Errorf("Unexpected result for zero length string, got: %v, want: %v", r, s)
  }
}

```

Note: We are using this unit test in the tutorial, so please don’t submit this change.

## Run Kubernetes Unit Tests Locally

```
make test WHAT=k8s.io/kubernetes/pkg/util/normalizer

```

## Contribute a Pull-Request

Commit your changes and push them to your fork:

```
# gofmt -w [INSERT PATH TO CHANGED FILES]
gofmt -w pkg/util/normalizer/normalizer_test.go
git config --global user.name "[Your Name]"
git config --global user.email "[youremail@example.com]â€
git add -A
git commit -m "[Commit message here]"
git push -u origin/my-branch-of-k8s

```

Create a pull request via the GitHub UI as follows:

1.  Navigate to  [https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)
2.  Click the “New Pull Request” button
3.  Click the “compare across forks” link
4.  Select your fork and branch from the two dropdowns on the right
5.  Click the “Create pull request” button
6.  Follow the instructions in the PR template (example below)
7.  Please prefix your PR title with  `[KubeCon PR Tutorial]`

Note that an existing maintainer must comment with the  `/ok-to-test`  command to initiate the automated tests for new contributors.

Example PR template:

```
<!--  Thanks for sending a pull request!  Here are some tips for you:
1. If this is your first time, read our contributor guidelines https://git.k8s.io/community/contributors/guide#your-first-contribution and developer guide https://git.k8s.io/community/contributors/devel/development.md#development-guide
2. Please label this pull request according to what type of issue you are addressing, especially if this is a release targeted pull request. For reference on required PR/issue labels, read here:
https://git.k8s.io/community/contributors/devel/release.md#issue-kind-label
3. Ensure you have added or ran the appropriate tests for your PR: https://git.k8s.io/community/contributors/devel/testing.md
4. If you want *faster* PR reviews, read how: https://git.k8s.io/community/contributors/guide/pull-requests.md#best-practices-for-faster-reviews
5. Follow the instructions for writing a release note: https://git.k8s.io/community/contributors/guide/release-notes.md
6. If the PR is unfinished, see how to mark it: https://git.k8s.io/community/contributors/guide/pull-requests.md#marking-unfinished-pull-requests
-->

**What type of PR is this?**
> Uncomment only one, leave it on its own line:
>
> /kind api-change
> /kind bug
> /kind cleanup
> /kind design
> /kind documentation
> /kind failing-test
> /kind feature
> /kind flake

**What this PR does / why we need it**:

**Which issue(s) this PR fixes** *(optional, in `fixes #<issue number>(, fixes #<issue_number>, ...)` format, will close the issue(s) when PR gets merged)*:
Fixes #

**Special notes for your reviewer**:

**Does this PR introduce a user-facing change?**:
<!--
If no, just write "NONE".
If yes, a release note is required:
Enter your extended release note in the block below. If the PR requires additional action from users switching to the new release, include the string "action required".
2.
-->
```release-note

```

```

# Clean up the VM

Navigate to  `Compute Engine`  >  `VM instances`  and click on the name of the VM you wish to delete, then click the  `DELETE`button. On smaller displays this button appears as a trashcan sign in the menu at the top of the page.
