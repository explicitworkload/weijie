# OpenShift Tasks — weijie

A self-paced set of hands-on tasks. Work through them in order; each one builds
on the last. There is no time limit — the point is understanding, not speed.

## Progress

- [ ] Task 1 — Deploy an app and reach it from a browser
- [ ] Task 2 — Fix three broken deployments
- [ ] Task 3 — Scale it, then try to kill it
- [ ] Task 4 — Change the app without rebuilding it
- [ ] Task 5 — Ship an update, then roll it back
- [ ] Task 6 — Build a cluster from scratch (Assisted Installer)

## How to work through these

**Predict before you run.** Every task has a "predict first" step. Say out loud
what you think will happen, then run the command. Being wrong is the useful
part — that is where the learning actually happens.

**Do not copy-paste blindly.** If you paste a command you do not understand,
you have completed nothing. `oc explain <resource>` is your friend
(`oc explain deployment.spec.strategy`).

**Hints and answers are collapsed** behind toggles. Sit with a problem for a
genuine while before opening them.

### Before you start

You need the `oc` CLI and access to a cluster. Give yourself your own project
so you cannot break anyone else's work:

```sh
oc version                     # confirm the client is installed
oc whoami                      # confirm you are logged in
oc new-project weijie-tasks    # your own sandbox
```

Everything below assumes you are in that project. Check with `oc project -q`.

---

## Task 1 — Deploy an app and reach it from a browser

**Goal:** get a web page running in the cluster and open it in a browser.

Make a trivial page and put it in the cluster as a ConfigMap:

```sh
echo '<h1>Hello from OpenShift</h1>' > index.html
oc create configmap web-content --from-file=index.html
```

Now write three objects yourself — a **Deployment**, a **Service**, and a
**Route**. Use the image `registry.access.redhat.com/ubi9/nginx-124:latest`,
which serves files from `/opt/app-root/src` on port 8080. Mount the ConfigMap
there.

> This image needs `command: ["nginx", "-g", "daemon off;"]`. Find out why in
> Task 2 — for now, just include it.

**You will probably get stuck on** the difference between `containerPort`,
`port`, and `targetPort`. That confusion is the entire point of this task. Work
it out before reading the hint.

**Done when:** `oc get route` gives you a URL that loads in a browser.

<details>
<summary>Hint — what the three objects are for</summary>

- **Deployment** creates and maintains Pods. Pods get random IPs and die often.
- **Service** gives that changing set of Pods one stable internal address.
  `port` is what the Service listens on; `targetPort` is the container port it
  forwards to.
- **Route** is OpenShift-specific. It exposes a Service to the outside world on
  a real hostname. (Vanilla Kubernetes calls this an Ingress.)

Nothing is reachable from a browser until all three exist.
</details>

<details>
<summary>Check your work</summary>

```sh
oc get deployment,svc,route
oc get pods                            # should be Running and 1/1 READY
oc get endpoints web                   # must list a pod IP, not <none>
curl -I "https://$(oc get route web -o jsonpath='{.spec.host}')"
```

If `oc get endpoints` shows `<none>`, your Service selector does not match your
Pod labels.
</details>

**Variant worth doing:** delete it all and redo it in one command with
`oc new-app`. Then run `oc get deployment -o yaml` and read what that single
command generated for you. Magic first, explanation second.

---

## Task 2 — Fix three broken deployments

**Goal:** diagnose and repair three deployments that are each broken in a
different way. This is the most valuable task here. Most people can deploy
something that works; far fewer can fix something that does not.

Save each block below to a file and apply it. They all expect the
`web-content` ConfigMap from Task 1.

### broken-1.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-1
  template:
    metadata:
      labels:
        app: broken-1
    spec:
      containers:
        - name: web
          image: registry.access.redhat.com/ubi9/nginx-124:lastest
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: content
              mountPath: /opt/app-root/src
      volumes:
        - name: content
          configMap:
            name: web-content
```

### broken-2.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-2
  template:
    metadata:
      labels:
        app: broken-2
    spec:
      containers:
        - name: web
          image: registry.access.redhat.com/ubi9/nginx-124:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: content
              mountPath: /opt/app-root/src
      volumes:
        - name: content
          configMap:
            name: web-content
```

### broken-3.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-3
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-3
  template:
    metadata:
      labels:
        app: broken-3
    spec:
      containers:
        - name: web
          image: registry.access.redhat.com/ubi9/nginx-124:latest
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: content
              mountPath: /opt/app-root/src
      volumes:
        - name: content
          configMap:
            name: web-content
---
apiVersion: v1
kind: Service
metadata:
  name: broken-3
spec:
  selector:
    app: broken-3
  ports:
    - port: 8080
      targetPort: 8081
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: broken-3
spec:
  to:
    kind: Service
    name: broken-3
```

### Your job

For **each** one, write down three things before you fix it:

1. What is the symptom? (What does `oc get pods` say? What does the route do?)
2. What is the actual cause?
3. Which command told you the answer?

The third question matters most. Build the reflex of running this loop:

```sh
oc get pods                  # what state is it in?
oc describe pod <name>       # read the Events at the bottom — always
oc logs <name>               # what did the app itself say?
oc get events --sort-by=.lastTimestamp
```

**Note that broken-3 is different.** Its pods will look perfectly healthy. That
is deliberate — "Running" does not mean "working", and learning to distrust a
green pod list is the single most useful thing in this task.

**Done when:** all three serve the page, and you can explain each failure in
one sentence.

<details>
<summary>Hint — broken-1</summary>

Look at `oc get pods` — the status itself names the problem. Then run
`oc describe pod` and read the Events. The message tells you the cluster tried
to fetch something and could not find it. Read the image line character by
character.
</details>

<details>
<summary>Hint — broken-2</summary>

The pod starts, then stops, then starts again. Run `oc get pods` a few times —
the `STATUS` column flips between `CrashLoopBackOff` and `Completed`, and
`RESTARTS` keeps climbing. That climbing number is the real signal.

`Completed` looks like success. Ask yourself why a web server finishing its
work would be a problem.

`oc logs` will show you exactly what the container printed before it exited.
This image is a *builder* image — what does it do when you run it without
telling it what to run?
</details>

<details>
<summary>Hint — broken-3</summary>

The pods are fine. Stop looking at them. Traffic goes
Route → Service → Pod, so if the pods are healthy, the break is somewhere in
that chain.

```sh
oc get endpoints broken-3
```

Endpoints are listed as `IP:PORT`. Is that port one that anything is actually
listening on?
</details>

<details>
<summary>Answers</summary>

**broken-1 — `ImagePullBackOff`.** The tag is `lastest`, not `latest`. The
registry has no such tag. Fix the typo.

*Lesson:* `ImagePullBackOff` is always about getting the image — a typo, a
private registry with no pull secret, or a nonexistent tag.

**broken-2 — `CrashLoopBackOff` / `Completed`.** The `command:` override is
missing. This is a Source-to-Image builder image whose default command runs a
`usage` script that prints help text and exits. Add back:

```yaml
command: ["nginx", "-g", "daemon off;"]
```

*Lesson 1:* the status alternating between `Completed` and `CrashLoopBackOff`
is not two different problems — it is one pod cycling. `Completed` is the
moment after it exits; `CrashLoopBackOff` is Kubernetes waiting before trying
again, with the delay doubling each time. The `RESTARTS` counter is the
reliable signal, not `STATUS`.

*Lesson 2:* check the exit code — it is **0**, meaning "success":

```sh
oc get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

A container is not a script that finishes. Its main process is meant to run
forever, so exiting *at all* is the failure, even exiting cleanly. This is why
`daemon off` matters: without it nginx would background itself, the foreground
process would exit 0, and you would get this exact behaviour from a web server
that genuinely started fine.

*Lesson 3:* `oc logs` almost always tells you why — here it prints the image's
usage text.

**broken-3 — pods healthy, route returns 503.** The Service forwards to
`targetPort: 8081`, but nginx listens on **8080**. One line fixes it:

```yaml
ports:
  - port: 8080
    targetPort: 8080     # was 8081
```

Re-apply the Service and the route works within seconds. You do not need to
touch the Deployment or the Route.

*Lesson:* this is the nastiest class of bug because every individual object
reports healthy. The Service even has endpoints — `10.128.0.159:8081` — so it
looks wired up. It is pointing at a port where nothing is listening, and every
connection is refused.

When pods are healthy and traffic still fails, walk the chain in order:

```
Route  →  Service  →  Endpoints  →  Pod
```

`oc get endpoints` is the fastest and most under-used check in Kubernetes. It
answers two questions at once: *did the selector match anything* (is the list
empty?) and *is the port right* (does it match the container port?).
</details>

**Stretch:** break it yourself. Change the Service `selector` to `app: broken-33`
and apply. What does `oc get endpoints` show now? This is the most common
real-world version of this bug.

<details>
<summary>A trap worth knowing about before it costs you an afternoon</summary>

Editing a **Route's** `spec.port.targetPort` in place often does *not* take
effect. The router keeps serving the old configuration, so you fix the problem
correctly and still get a 503 — which makes you doubt a correct answer and go
chasing the wrong thing.

```sh
oc delete route <name> && oc expose svc <name>      # recreate it
```

**Service** changes, by contrast, apply immediately, because endpoints are
recalculated and the router watches them continuously.

General rule: when a change *should* have worked and did not, check whether
the thing you edited is actually re-read at runtime. Not every field is.
</details>

---

## Task 3 — Scale it, then try to kill it

**Goal:** understand the single most important idea in Kubernetes.

Using your Task 1 deployment:

```sh
oc scale deployment/web --replicas=3
oc get pods -w                   # Ctrl-C to stop watching
```

Now **predict first**, then run. Write your prediction down before you hit
enter:

```sh
oc delete pod <one-of-the-pod-names>
oc get pods
```

Then try to win:

```sh
oc scale deployment/web --replicas=0
oc get pods
```

**Done when:** you can explain why the deleted pod came back, and why
`--replicas=0` succeeded where `oc delete pod` failed.

<details>
<summary>The answer</summary>

You cannot delete that pod. It comes straight back with a new name.

Kubernetes is **declarative**. The Deployment records a *desired state*:
"3 pods should exist." A controller watches the real world and continuously
reconciles it to match. Deleting a pod does not change the desired state — you
changed the world, and the controller changed it back within seconds.

`--replicas=0` works because it changes the desired state itself.

This is why you almost never run imperative commands against pods in anger. To
change what is running, change the spec. Everything in Kubernetes works this
way, and once this clicks, most of the rest follows.

**Self-healing is the same mechanism.** Nothing special happens when a node
dies — the controller simply notices reality no longer matches the spec.
</details>

**Stretch:** run `oc delete pod --all` and watch. Then look at
`oc get replicaset` — which object is actually doing the recreating? It is not
the Deployment directly.

---

## Task 4 — Change the app without rebuilding it

**Goal:** see why configuration is kept separate from the container image.

Your page lives in a ConfigMap, not in the image. Change it:

```sh
oc edit configmap web-content
```

Edit the HTML, save, and exit. Now refresh your browser.

**Predict first:** how long until the page changes? Do the pods restart?

```sh
oc get pods        # check the AGE and RESTARTS columns
```

Then try the other half — a **Secret**:

```sh
oc create secret generic app-secret --from-literal=API_KEY=not-a-real-key
```

Mount it into your Deployment as an environment variable (look up
`env.valueFrom.secretKeyRef`), then:

```sh
oc exec deployment/web -- env | grep API_KEY
oc get secret app-secret -o yaml       # look closely at the value
```

**Done when:** your edited page is live with no rebuild and no pod restart, and
you can state what Secrets do and do not protect you from.

<details>
<summary>What you should have seen</summary>

The page changes within about a minute, and **the pods never restart** —
`AGE` keeps climbing and `RESTARTS` stays at 0. The kubelet refreshes mounted
ConfigMap files in place, and nginx reads the file fresh on each request.

Two important caveats:

1. **`subPath` mounts do not update.** Mount a single file with `subPath` and
   it is frozen at pod creation forever. This catches people out constantly.
2. **Environment variables do not update either.** Values from
   `env.valueFrom.configMapKeyRef` are injected once at start. Changing the
   ConfigMap does nothing until the pod restarts.

**On Secrets:** `oc get secret -o yaml` shows base64, which is *encoding, not
encryption*. Anyone who can read the Secret can decode it instantly:

```sh
oc get secret app-secret -o jsonpath='{.data.API_KEY}' | base64 -d
```

Secrets protect you via RBAC (who can read them) and by keeping credentials out
of your image and your git repo. They are not a vault.

**The real lesson:** the same image runs in dev, staging and production. Only
the config differs. If you have to rebuild an image to change a setting, the
setting is in the wrong place.
</details>

---

## Task 5 — Ship an update, then roll it back

**Goal:** perform a zero-downtime release, then recover from a bad one.

Scale to 3 and watch traffic in one terminal while you work in another:

```sh
URL="https://$(oc get route web -o jsonpath='{.spec.host}')"
while true; do curl -s -o /dev/null -w "%{http_code} " "$URL"; sleep 0.2; done
```

In a second terminal, trigger a rollout:

```sh
oc set env deployment/web RELEASE=v2
oc rollout status deployment/web
```

Watch the first terminal. **Count the non-200s.** Then look at what controlled
that behaviour:

```sh
oc get deployment web -o jsonpath='{.spec.strategy}' | python3 -m json.tool
```

Now break production on purpose:

```sh
oc set image deployment/web web=registry.access.redhat.com/ubi9/nginx-124:nope
oc rollout status deployment/web      # this will not finish
oc get pods
```

**Predict first:** is the site down? How many pods are serving traffic?

Then recover:

```sh
oc rollout history deployment/web
oc rollout undo deployment/web
oc rollout status deployment/web
```

**Done when:** you have done a clean rollout with no dropped requests, and
recovered from a broken one.

<details>
<summary>What you should have seen</summary>

**The good rollout drops nothing.** `maxUnavailable: 0` means never go below
the desired count, and `maxSurge: 1` means add one new pod at a time. The new
pod must pass its readiness probe before an old one is removed.

**The bad rollout does not take the site down.** This surprises people. The new
pod sits in `ImagePullBackOff` and never becomes ready, so the old pods are
never removed. The rollout stalls, and your users notice nothing. A good
deployment strategy fails *safe*.

`oc rollout undo` reverts to the previous ReplicaSet — which is exactly why old
ReplicaSets are kept around with 0 replicas. Look at `oc get rs`.

**The critical caveat:** all of this depends on **readiness probes**. Without
one, a pod is considered ready the instant it starts, so Kubernetes happily
removes healthy old pods and replaces them with new ones that are not yet
serving. Delete your readiness probe and run the experiment again — now you
will see the failures.

Zero-downtime deploys are not free. They are a property of correct probes.
</details>

---

## Task 6 — Build a cluster from scratch (Assisted Installer)

Everything so far was about running workloads *on* a cluster. This task is
about creating the cluster itself.

**Goal:** install a working OpenShift cluster using the Assisted Installer at
[console.redhat.com](https://console.redhat.com/openshift).

### Read the documentation. This one is manual.

**This task must be completed by reading the official documentation.** No
walkthrough is provided here, and you should not follow a blog post, a YouTube
video, or an AI-generated set of steps.

That is not an arbitrary restriction. Installation is where you learn how the
pieces actually fit together — DNS, networking, VIPs, host roles, certificates.
A copy-pasted install teaches you nothing and leaves you helpless the moment a
validation fails. Every real cluster install hits at least one problem, and the
skill being built here is *reading the docs and working it out*.

- **Assisted Installer documentation (current release — year-versioned, use
  2026):**
  https://docs.redhat.com/en/documentation/assisted_installer_for_openshift_container_platform/2026
- **OpenShift 4.22 — Installing on-premise with Assisted Installer:**
  https://docs.redhat.com/en/documentation/openshift_container_platform/4.22
- **Start the install:** https://console.redhat.com/openshift

You will need a Red Hat account. A free Developer subscription is enough.

### Answer these from the docs before you touch anything

Write the answers down. If you cannot answer them, you are not ready to start.

1. What is the minimum number of hosts for a multi-node cluster? What does each
   host need in CPU, RAM, and disk?
2. What is Single Node OpenShift (SNO), and what does it cost you? When is it
   the right choice?
3. Which DNS records must exist *before* installation, and what must they
   resolve to? This is the number one cause of failed installs.
4. What are the API VIP and the Ingress VIP? When do you need them, and when do
   you not?
5. What is the discovery ISO, and what is the difference between the full and
   minimal versions?
6. Can your hosts use DHCP, or do you need static IPs? What changes if so?

### What you have to do

These are objectives, not instructions. Work out the steps yourself.

1. Provide hosts — bare metal, or VMs on libvirt/VMware/nested virtualisation.
2. Set up DNS and networking to meet the documented requirements.
3. Create the cluster definition in the Assisted Installer console.
4. Generate the discovery ISO and boot your hosts from it.
5. Get every host to register and pass all validations. **Expect to fail here
   at least once.** Read the validation message, go back to the docs, fix it.
6. Assign host roles, then start the installation.
7. Retrieve the kubeconfig and the `kubeadmin` password.

### Done when

```sh
export KUBECONFIG=/path/to/your/kubeconfig
oc get nodes                   # every node Ready
oc get clusteroperators        # every operator Available=True, Degraded=False
oc whoami --show-console       # the web console loads in a browser
```

Then do one last thing: **deploy Task 1 onto the cluster you just built.**

### Write it up

Keep notes as you go and finish with a short document covering:

- Every validation that failed, and what fixed it
- What you got wrong about DNS the first time (you will get DNS wrong)
- What you would do differently next time

The write-up is part of the task. Being able to explain an install is the
difference between having done it and understanding it.

---

## Where to go after this

Storage (`PersistentVolumeClaim`, `StorageClass`), RBAC (`Role`,
`RoleBinding`, ServiceAccounts), resource limits and what actually happens when
a pod is OOMKilled, health probe tuning, Operators and the OperatorHub, and
`oc adm` for day-two cluster administration.
