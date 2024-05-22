<!--
# SPDX-FileCopyrightText: 2024 Buoyant Inc.
# SPDX-License-Identifier: Apache-2.0
#
# Copyright 2024 Buoyant Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may
# not use this file except in compliance with the License.  You may obtain
# a copy of the License at
#
#     http:#www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
 -->

# Backstage and Linkerd

This is the documentation - and executable code! - for the demo of Backstage
and Linkerd presented at Cloud Native Live on 22 May 2024. The easiest way to
use this file is to execute it with [demosh].

Things in Markdown comments are safe to ignore when reading this later. When
executing this with [demosh], things after the horizontal rule below (which
is just before a commented `@SHOW` directive) will get displayed.

[demosh]: https://github.com/BuoyantIO/demosh

When you use `demosh` to run this file, your cluster will be checked for you.

<!-- set -e >
<!-- @start_livecast -->

---

<!-- @SHOW -->

# Backstage and Linkerd

We're going to show how Backstage and Linkerd can work together to help
understand an application. For this demo, we'll be using the Faces demo from
https://github.com/BuoyantIO/faces-demo: this is a deliberately-broken
single-page web app built using a microservices architecture.

**We assume that you already have an empty cluster ready to go.** If you
don't, stop now and create one! Pretty much any cluster will work as long as
you're running a recent version of Kubernetes.

<!-- @wait_clear -->

## Installing Faces

Let's start by installing Faces. We'll use its Helm chart to do this, and
we'll also tell it to use a LoadBalancer for the workload that serves the GUI,
which lets us run the demo without any additional ingress controller. (This is
probably a terrible idea for the real world! But it simplifies the demo.)

```bash
helm install faces -n faces --create-namespace \
     oci://ghcr.io/benjdlambert/faces-chart --version 0.0.3 \
     --set gui.serviceType=LoadBalancer

kubectl rollout status -n faces deploy
```

OK, Faces should be running! Let's take a look at the external IP of the
`faces-gui` Service; that's what we need to point a web browser at Faces.

```bash
kubectl get svc -n faces faces-gui
```

Given that, we can go take a look at Faces in our web browser.

<!-- @browser_then_terminal -->

## The State of Faces

It should be pretty obvious that Faces is in pretty sorry shape: we should see
grinning faces on blue backgrounds, but we often don't. This, of course,
raises the question of _why_ it's in such sorry shape.

<!-- @wait -->

We don't actually have a lot of options for how to answer that right now.
Faces uses four microservices, and we have no information at all about what's
failing. Faces doesn't even log anything useful.

```bash
kubectl logs -n faces deploy/face
```

We have the Git commit, at least, but we have _nothing_ about what's going
wrong.

<!-- @wait_clear -->

## Installing Linkerd

We need something to tell us what's happening -- which is to say, we need
_observability_. This is a core feature of a service mesh, so let's get
Linkerd in there and see what we can see.

We'll use the Linkerd CLI for this -- in production you might want Helm or the
like, but the CLI is great for demos like this one. One quick note here: the
Linkerd project only produces edge release artifacts, with the vendor
community picking up stable distributions. We're going to use the latest edge
release for this demo, so we'll start with installing the latest edge CLI.

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install-edge | sh
```

Let's make sure which version we have...

```bash
linkerd version --proxy --short
```

...and then get to installing! First, we install Linkerd's (relatively few)
CRDs.

```bash
linkerd install --crds | kubectl apply -f -
```

After that, we can install Linkerd itself.

```bash
linkerd install | kubectl apply -f -
```

We also want the Linkerd Viz extension: this does all of Linkerd's
visualizations, and it's installed separately from the core of Linkerd because
you don't actually need it to get real benefit from Linkerd. It's highly
recommended, though, for exactly the kind of situation we're facing here.

```bash
linkerd viz install | kubectl apply -f -
```

Finally, we can check that everything is working as expected.

```bash
linkerd check
```

## Meshing Faces

Once we have Linkerd installed, we can mesh Faces. We'll do this the easy way,
by annotating the whole namespace to be included in the mesh:

```bash
kubectl annotate ns faces linkerd.io/inject=enabled
```

We then need to restart Faces to bring the Linkerd proxies into play.

```bash
kubectl rollout restart -n faces deploy
kubectl rollout status -n faces deploy
```

If we flip over to our web browser, everything should still be working - or
not working! - as it was before.

<!-- @browser_then_terminal -->

## Observing Faces

Now that Faces is meshed, we can start to see what's happening. We'll start by
looking at the Linkerd dashboard.

```bash
linkerd viz dashboard
```

This is an improvement! We can start to see what's happening in Faces in real
time... but of course, with just Linkerd we don't know much more about these
microservices.

This is where an internal developer platform comes in.
