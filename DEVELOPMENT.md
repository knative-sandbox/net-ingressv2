# Development

This doc explains how to setup a development environment so you can get started
[contributing](https://www.knative.dev/contributing/) to Knative
`net-ingressv2`. Also take a look at:

- [The pull request workflow](https://knative.dev/community/contributing/reviewing/)

## Getting started

1. Create [a GitHub account](https://github.com/join)
1. Setup
   [GitHub access via SSH](https://help.github.com/articles/connecting-to-github-with-ssh/)
1. Install [requirements](#requirements)
1. Set up your [shell environment](#environment-setup)
1. [Create and checkout a repo fork](#checkout-your-fork)

Before submitting a PR, see also [CONTRIBUTING.md](./CONTRIBUTING.md).

### Requirements

You must install these tools:

1. [`go`](https://golang.org/doc/install): The language Knative `net-ingressv2`
   is built in
1. [`git`](https://help.github.com/articles/set-up-git/): For source control
1. [`dep`](https://github.com/golang/dep): For managing external dependencies.

### Environment setup

To get started you'll need to set these environment variables (we recommend
adding them to your `.bashrc`):

1. `GOPATH`: If you don't have one, simply pick a directory and add
   `export GOPATH=...`

1. `$GOPATH/bin` on `PATH`: This is so that tooling installed via `go get` will
   work properly.

`.bashrc` example:

```shell
export GOPATH="$HOME/go"
export PATH="${PATH}:${GOPATH}/bin"
```

### Checkout your fork

The Go tools require that you clone the repository to the
`src/knative.dev/net-ingressv2` directory in your
[`GOPATH`](https://github.com/golang/go/wiki/SettingGOPATH).

To check out this repository:

1. Create your own
   [fork of this repo](https://help.github.com/articles/fork-a-repo/)

1. Clone it to your machine:

```shell
mkdir -p ${GOPATH}/src/knative.dev
cd ${GOPATH}/src/knative.dev
git clone git@github.com:${YOUR_GITHUB_USERNAME}/net-ingressv2.git
cd net-ingressv2
git remote add upstream https://github.com/knative-sandbox/net-ingressv2.git
git remote set-url --push upstream no_push
```

_Adding the `upstream` remote sets you up nicely for regularly
[syncing your fork](https://help.github.com/articles/syncing-a-fork/)._

Once you reach this point you are ready to do a full build and deploy as
described below.

### Execute conformance tests

Currently this repo tests with Istio and Contour. Please follow
[Test with Istio](#test-with-istio) or [Test with Contour](#test-with-contour).

### Test with Istio

#### Prepare test resources such as namespaces

```
kubectl apply -f test/config/
```

#### Install Gateway API CRDs

```
kubectl apply -k 'github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.2.0'
```

#### Deploy Istio

Run the following command to install Istio (from HEAD) for development purpose:

```shell
./third_party/istio-head/install-istio.sh istio-kind-no-mesh.yaml
```

#### Deploy GatewayClass and Gateway

```
kubectl apply -f ./third_party/istio-head/gateway/
```

#### Execute test

```shell
GATEWAY_OVERRIDE=istio-ingressgateway
GATEWAY_NAMESPACE_OVERRIDE=istio-system
IPS=( $(kubectl get nodes -lkubernetes.io/hostname!=kind-control-plane -ojsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}') )

go test -v -tags=e2e -count=1  ./test/conformance/ingressv2/  -run "TestIngressConformance/basics" \
  --ingressClass=istio \
  --ingressendpoint="${IPS[0]}"
```

Some tests are still not available. Please see
https://github.com/knative-sandbox/net-ingressv2/issues/23.

### Test with Contoour

#### Prepare test resources such as namespaces

```
kubectl apply -f test/config/
```

#### Install Gateway API CRDs

This step is not necessary for Contour as contour operator installs Gateway API
CRDs.

#### Deploy Contour

Run the following command to install Contour and its operator (from HEAD).

```shell
./third_party/contour-head/install-operator.sh
```

#### Deploy GatewayClass and Gateway

```
ko resolve -f ./third_party/contour-head/gateway/gateway-external.yaml | \
  sed 's/LoadBalancerService/NodePortService/g' | \
  kubectl apply -f -

ko resolve -f ./third_party/contour-head/gateway/gateway-internal.yaml | \
  kubectl apply -f -
```

#### Execute test

```shell
GATEWAY_OVERRIDE=envoy
GATEWAY_NAMESPACE_OVERRIDE=contour-external
LOCAL_GATEWAY_OVERRIDE=envoy
LOCAL_GATEWAY_NAMESPACE_OVERRIDE=contour-internal
IPS=( $(kubectl get nodes -lkubernetes.io/hostname!=kind-control-plane -ojsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}') )

go test -v -tags=e2e -count=1  ./test/conformance/ingressv2/  -run "TestIngressConformance/hosts/basics" \
  --ingressClass=contour \
  --ingressendpoint="${IPS[0]}"
```

Some tests are still not available. Please see
https://github.com/knative-sandbox/net-ingressv2/issues/36.
