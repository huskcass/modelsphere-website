---
title: Getting started
description: Install ModelSphere on a Kubernetes cluster you already have, open the Console, learn the interface and menus, and follow one flow from picking a model to calling it from code.
---

ModelSphere Console is the single entry point of the inference platform: deploy models, try them out and issue API keys here; programs then call models through the OpenAI-compatible API with those keys.

## Prerequisites

Run the commands below from a checkout of the [modelsphere](https://github.com/modelsphere/modelsphere) repository. This section is Part C of the install guide: installing the stack on a Kubernetes cluster you already have. Preparing bare machines and creating a cluster are covered in the [install guide](https://github.com/modelsphere/modelsphere/blob/main/docs/install.md).

On the machine you type from:

| Requirement | How to get it |
|---|---|
| A Kubernetes cluster, and `kubectl` pointing at it | `kubectl config current-context` |
| helm ≥ 3.8 and helmfile ≥ 1.0 | Install them yourself; tested on helm v4.3.0 + helmfile 1.8.0 |
| The helm-diff plugin | `make helm-deps` — once per machine, and not optional: `helmfile apply` shells out to it |

## Install the stack

:::warning
This step writes to the cluster, and the defaults are for a cluster being built from scratch. `environments/default.yaml` switches on eighteen components, among them a CNI, a GPU operator, a scheduler and a monitoring stack. On a cluster that has its own, turn them off in your environment file first — [On a cluster you already have](https://github.com/modelsphere/modelsphere/blob/main/docs/install.md#on-a-cluster-you-already-have) is that block, ready to copy. `make helm-diff ENV=<env>` lists everything an apply would create, and changes nothing.
:::

Optional: run `make helm-bootstrap` first, which installs the Gateway API CRDs and PriorityClasses. Skip it when all three hold: you expose the routing layer some other way, no values of yours name a PriorityClass, and you have not turned `cilium.gatewayAPI` on (off by default; on, Cilium needs these CRDs too). Both objects are cluster-wide — installed once per cluster, not once per person.

```bash
make helm-bootstrap   # Gateway API CRDs + PriorityClasses
```

Then copy the environment file, edit it for your cluster, register it under `environments:` in `helmfile.yaml.gotmpl`, and install the whole stack in one pass. Putting the file in `environments/` is not enough: helmfile never reads a values file that nothing registers.

```bash
# 1. the stack itself, one pass
cp environments/private.yaml.example environments/mycluster.yaml   # edit it,
#    then add it under `environments:` in helmfile.yaml.gotmpl -- a values file
#    nothing registers is a file helmfile never reads:
#      mycluster:
#        values:
#          - environments/default.yaml
#          - environments/mycluster.yaml
make helm-apply ENV=mycluster
```

The registration in `helmfile.yaml.gotmpl` looks like this:

```yaml
environments:
  mycluster:
    values:
      - environments/default.yaml     # the base layer, then your overrides
      - environments/mycluster.yaml
```

When it finishes, `helm list -A` should show every enabled release `deployed`, and `kubectl get pods -A` nothing outside `Running`/`Completed`.

## Open the Console

The Service is a `NodePort`, so the UI is reachable as soon as the release is up — no Ingress needed:

```bash
NODE_PORT=$(kubectl -n console get svc console-console -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "http://$NODE_IP:$NODE_PORT/"
```

That is the chart's own address recipe; `helm status console -n console` prints it too. Behind a firewall, port-forward and open `http://localhost:8080/`:

```bash
kubectl -n console port-forward svc/console-console 8080:8080
```

Sign in as `admin` with the password `P@88w0rd`. The first login asks you to set a new password — do it; the default is public knowledge. If your cluster renamed the namespace, use `namespaces.console` from your environment file in place of `console` above.

## The whole flow

```
pick a model -> deploy an inference service -> wait until ready -> try it in the Playground -> create an API key -> call /v1
```

| Step | Page | See |
|---|---|---|
| Pick and deploy a model | Model Serving → Model Library | [Deploying models](deploy.md) |
| Watch the rollout | Model Serving → Inference Services | [Deploying models](deploy.md) |
| Chat with a model | Playground → Chat | [Playground](playground.md) |
| Issue keys | Router → API Keys | [Calling the API](api.md) |
| Manage users and roles | Access Control → Users | [Access control](access-control.md) |

## Signing in

- Sign in with the username and password your administrator gave you. The administrator account created at install time is `admin`.
- **The first sign-in asks you to set a new password.** Keeping the initial one is allowed, with a warning, but choose one only you know.
- To change it later: avatar at the top right → **Change password**. At least 8 characters; the strength meter updates as you type.

## The interface

- The **sidebar** groups pages by feature and shows only the pages your account may open. If a menu is missing, your role does not grant it; ask an administrator.
- **Top right**:
  - User Guide: opens this guide from any page.
  - Language icon: switch between Chinese and English.
  - Palette icon: preferences, with three layouts (mixed nav, classic, minimal). Stored in this browser only.
  - Avatar: your identity, change password, sign out.
- The **Overview** page shows the current session (user, groups, role) and each module's summary cards.

## Menus

| Group | Menus | Purpose |
|---|---|---|
| Model Serving | Inference Services, Model Library, Nodes, Activity, Site Profile | Deploy and manage inference services |
| Playground | Chat | Talk to models in the browser |
| Router | API Keys | Keys that programs use to call `/v1` |
| Access Control | Users, Login History, Roles | Accounts and permissions |
| Help | User Guide | This guide |
