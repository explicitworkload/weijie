# Todo

A single-page todo app. One HTML file, no build step, no dependencies, no server. Tasks are saved in your browser via `localStorage`.

## Run it

Open [index.html](index.html) in a browser — double-click the file, or:

```sh
open index.html          # macOS
```

If you prefer serving it over HTTP:

```sh
python3 -m http.server 8000
# then visit http://localhost:8000
```

## What it does

- Add tasks, mark them done, delete them
- Edit a task by double-clicking its text (`Enter` saves, `Esc` cancels)
- Filter by **All** / **Active** / **Done**
- Clear all completed tasks at once
- Live count of remaining tasks
- Tasks persist across reloads
- Light and dark themes, following your OS setting
- Responsive down to small screens; keyboard and screen-reader friendly

## Project layout

```
index.html    the entire app — markup, styles, and logic
k8s/          OpenShift/Kubernetes manifests
README.md     this file
```

## Deploy to OpenShift

The manifests in [k8s/](k8s/) serve `index.html` from a ConfigMap using the UBI
nginx image. No image build and no registry are involved — the page is the only
artifact. They target the `weijie` namespace, which must already exist.

```sh
oc apply -f k8s/
oc rollout status deployment/todo -n weijie
oc get route todo -n weijie
```

| File | Purpose |
| --- | --- |
| [k8s/configmap.yaml](k8s/configmap.yaml) | `index.html`, generated from the file at the repo root |
| [k8s/deployment.yaml](k8s/deployment.yaml) | 2 nginx replicas serving `/opt/app-root/src` on port 8080 |
| [k8s/service.yaml](k8s/service.yaml) | ClusterIP service on port 8080 |
| [k8s/route.yaml](k8s/route.yaml) | Edge-TLS route, HTTP redirected to HTTPS |

### After editing `index.html`

The ConfigMap is generated, so regenerate and re-apply it:

```sh
oc create configmap todo-content \
  --from-file=index.html=index.html \
  --namespace=weijie \
  --dry-run=client -o yaml > k8s/configmap.yaml
oc apply -f k8s/configmap.yaml
```

The volume is mounted as a directory rather than with `subPath`, so the running
pods pick up the new content within a minute or so — no restart needed. To skip
the wait, run `oc rollout restart deployment/todo -n weijie`.

### Notes

- The Route sets no `host`, so OpenShift derives one from the cluster's apps
  domain. Change the namespace in all four files to deploy elsewhere.
- The pods run under the `restricted-v2` SCC: non-root with an
  OpenShift-assigned UID, all capabilities dropped, no privilege escalation.
- A ConfigMap caps out at 1 MiB. That is ample here, but a larger site would
  want a real image build instead.

## Data

Everything lives under the `localStorage` key `todo.items.v1`, as an array of `{ id, text, done }`. Nothing leaves the browser — no network calls, no analytics. Clearing site data for the page wipes your tasks.

To reset manually, run this in the browser console:

```js
localStorage.removeItem("todo.items.v1");
```
