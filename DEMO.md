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

<!-- @wait_clear -->

## Installing Backstage

Backstage is a framework for building our developer platforms and portals, and
luckily we've used that framework to build our very own just for this demo!
This repository was created using `npx @backstage/create-app` and then customized
a little bit for this demo by adding a few additional plugins, it then produces a
docker container of the Developer Platform ready to deploy into your cluster.

Let's go ahead and install it into the cluster.

```bash
ls -l ./k8s
kubectl apply -f ./k8s
```

We can check that this has been deployed and ready by checking the pods:

```bash
kubectl rollout status -n backstage deploy --timeout 90s
```

In the real world, you would deploy Backstage behind some kind of ingress
controller with proper authentication, but for this demo, we're just going to
use `kubectl port-forward` to access it. Ugly, we know, but it's just for demo
purposes.

```bash
kubectl port-forward -n backstage deployment/backstage 7007:7007
```

Now we can access Backstage at http://localhost:7007.

```bash
open http://localhost:7007/catalog
```

## Exploring the Backstage Software Catalog

So first off what you're going to see in Backstage is the Software Catalog.
This is the list of all of your software in your organization. It's pretty empty,
just sample item inside there called `example-website`, which is just a demo component
that we've added.

This list of components can be ingested from a multitude of sources,
and processed in different ways. It's recommended to keep a `catalog-info.yaml` file next to
your software source code, in order to provide metadata about your component. We'll come onto
that shortly when we want to add some information about our `faces` stack.

Let's dive into the `example-website` component, we can do that by
clicking on the `example-website` item in the table.

```bash
open http://localhost:70007/catalog/default/component/example-website
```

This is the `Entity Page` for the `example-website` component.

Each of these different cards and tabs come from plugins which you can take advantage
from either Open Source, or by building your own.

Clicking through the Tabs at the top you can see things like `CI/CD` to show deployments
and workflow runs, `Techdocs` for documentation on this particular software component and
`Dependencies` to track what this component depends on and who depends on it.
