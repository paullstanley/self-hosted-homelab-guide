# Monitoring with Grafana and Prometheus

You end up with **https://grafana.home.example.com**: one page showing every node, pod, Service, ingress route and Pi-hole in the cluster, with history. Prometheus collects the numbers every minute and keeps 15 days. Grafana draws them. Nothing about how DNS or the apps fail over changes.

Example addresses and names are explained in [Conventions](../start-here/conventions.md); substitute your own.

| | |
| --- | --- |
| **Applies to** | k3s v1.34 with embedded etcd (three Raspberry Pi 4 servers and an arm64 Lima VM), the Traefik bundled with k3s, MetalLB, Helm chart `kube-prometheus-stack` 92.3.1 (Grafana 13.2.4), `mojo2600/pihole` 2.38.0 with the `ekofr/pihole-exporter` v1.2.0 sidecar |
| **Also works for** | Any Kubernetes 1.25 or newer with a default StorageClass and an ingress controller. Clusters that run the control plane as pods (kubeadm) should leave the controller-manager, scheduler and kube-proxy scrapes on. Not tested by the author |
| **Time** | 30 to 45 minutes, most of it image pulls on the Pis |
| **You need first** | [HA k3s cluster](../kubernetes/k3s-ha-cluster.md), [Load balancers](../kubernetes/load-balancers.md) (Traefik on its own address), [Pi-hole on k3s](../apps/pihole.md) for the name and the Pi-hole dashboard |

> **Not verified:** this page and its files were written on 13 October 2026 and **have not yet been applied by the author**. What was checked:
>
> - **Versions and settings** were read from upstream on 13 October 2026: the kube-prometheus-stack chart 92.3.1 and its `values.yaml`, the Grafana chart 13.5.1 it bundles, the k3s v1.34.3 Traefik manifest, the k3s metrics and server docs, the k3s etcd source, the MoJo2600 Pi-hole chart 2.38.0 and the pihole-exporter source. Links under "References".
> - **Every key** in `files/monitoring/*.yaml` was checked against the chart's own `values.yaml` (and the Grafana, kube-state-metrics and node-exporter subcharts). The files were **not** rendered with `helm template` (Helm could not be downloaded in the environment where they were written). Step 5 and Step 8 are the checks.
> - **The Traefik metrics port** (`metrics`, 9100) comes from the Traefik chart 37.1.0 defaults that k3s v1.34.3 ships. Step 7 checks it on the live pod.

---

## What gets installed

One Helm release, `kps` (kube-prometheus-stack), in a new namespace `monitoring`:

| Part | What it does | Where it runs |
| --- | --- | --- |
| Prometheus | Scrapes every target every 60 s, keeps 15 days (at most 18 GB) on a 25 Gi volume | server-1 |
| Grafana | The web page. Its own small SQLite database (users, preferences, dashboards made in the UI) on a 2 Gi volume | server-1 |
| Prometheus Operator | Turns `ServiceMonitor` and `PodMonitor` objects into Prometheus scrape settings | server-1 |
| kube-state-metrics | The state of every Kubernetes object: pods, Deployments, Services, Ingresses, PVCs, restarts | server-1 |
| node-exporter | CPU, memory, disk, network and temperature of each machine | All four nodes (host network, port 9100) |
| Alert rules | The chart's standard rules (pod crash-looping, node disk filling, target down, and more). They evaluate and show in Grafana; nothing sends them anywhere yet ("Later") | Inside Prometheus |

Plus, from other files:

| Thing | File | Why |
| --- | --- | --- |
| `grafana.home.example.com` → 192.168.50.12 | `files/pihole/values.yaml` | Same pattern as every web app behind Traefik ([Load balancers](../kubernetes/load-balancers.md), last section) |
| A Pi-hole exporter next to each Pi-hole pod, and a `PodMonitor` | `files/pihole/values.yaml` | Grafana shows all three Pi-holes, summed or one by one. This is the "one view of all the pods' stats" that the sticky cookie can't give ([Pi-hole on k3s](../apps/pihole.md)) |
| `redirect-https` Middleware in `monitoring` | `files/traefik/middleware-redirect-https.yaml` | http:// to https://, like Pi-hole and Homebridge |
| Ports 9100 and 2381 blocked from the LAN | `files/firewall/k3s-firewall.sh` | Metrics ports have no login |
| `grafana.home.example.com` on a laptop behind a corporate VPN | `files/clients/work-mac-hosts.txt` | The VPN's DNS can't see home names ([Client devices](../apps/client-devices.md)) |
| `etcd-expose-metrics: true` | `files/k3s/config/*.yaml` | Phase 2 only |

### Why these choices

- **Everything that stores data on one node.** Pick the one with the most RAM and disk; in the build that is server-1, a Pi 4 with 8 GB and a 2 TB disk, while server-2 and server-3 have 2 GB each. A `monitoring-host=true` label pins Prometheus, Grafana, the operator and kube-state-metrics there. If server-1 is down, the dashboards are down; DNS, Homebridge and everything else carry on. The 2 TB disk is a spinning drive: fine for 60-second scrapes of a cluster this size.
- **No new MetalLB address.** Grafana is a web app, so it goes behind Traefik on .12 ([Load balancers](../kubernetes/load-balancers.md)). Prometheus itself is not published at all; use the port-forward in Step 8 when you need it.
- **One writer per SQLite file.** Grafana runs one pod with `Recreate`, so an upgrade stops the old pod before starting the new one and two Grafanas never open the same database. Prometheus keeps its own data in a single pod too. The same rule as Pi-hole: databases are never shared ([Pi-hole on k3s](../apps/pihole.md)).
- **k3s is one process.** The controller-manager, scheduler and kube-proxy run inside k3s with one shared metrics registry, so the API server scrape already returns their metrics. The k3s docs: "It is only necessary to scrape a single K3s metric endpoint in order to get metrics for all embedded Kubernetes components." Their separate scrapes, and the rules that would call them "down", are switched off.
- **Etcd is the exception.** Embedded etcd only serves metrics on `127.0.0.1:2381` until k3s gets `etcd-expose-metrics: true`. That needs a k3s restart on every server, so it is a separate, optional phase.
- **Dashboards as code.** Grafana loads its dashboards from fixed versions at start-up, so a rebuild shows exactly the same ones. Dashboards you make in the UI live in Grafana's own volume.
- **No Alertmanager yet.** Where alerts should go (phone, email) is still undecided. The rules run anyway, and Grafana's **Alerting → Alert rules** shows what is firing.

### Versions (13 October 2026)

| Component | Version | Source |
| --- | --- | --- |
| kube-prometheus-stack chart | 92.3.1 (newest) | `oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack` |
| Prometheus Operator | v0.94.1 | Chart `appVersion` |
| Grafana chart / Grafana | 13.5.1 / 13.2.4 (distroless image) | Subchart from `oci://ghcr.io/grafana-community/helm-charts` |
| kube-state-metrics chart | 8.6.0 | Subchart |
| prometheus-node-exporter chart | 4.59.0 | Subchart |
| Kubernetes dashboards | dotdc/grafana-dashboards-kubernetes v3.0.8 (newest) | GitHub |
| Node Exporter Full | rfmoz/grafana-dashboards, commit `99a25cc` | GitHub |
| Traefik dashboard | `contrib/grafana/traefik-kubernetes.json` from Traefik v3.5.1 | GitHub |
| Pi-hole exporter | `ekofr/pihole-exporter:v1.2.0` (newest; Pi-hole v6 API since v1.0.0) | Docker Hub |
| Pi-hole dashboard | eko/pihole-exporter `grafana/dashboard.json`, commit `bc1bf69` | GitHub |

The chart needs Kubernetes 1.25 or newer (the cluster runs v1.34.3) and Helm 3.8 or newer for `oci://`.

**Which Grafana chart.** The Grafana chart in `grafana/helm-charts` (Artifact Hub "grafana/grafana", last version 10.5.15) carries a migration notice: it moved to [grafana-community/helm-charts](https://github.com/grafana-community/helm-charts), and updates after 30 January 2026 happen only there. kube-prometheus-stack 92.3.1 already bundles the community chart (13.5.1), so nothing here uses the old one. Don't install Grafana separately from `grafana/helm-charts` next to this stack; Grafana's own values live under `grafana:` in this file.

---

## Phase 1: Prometheus and Grafana

### Step 1. Check what is there

**Run on: server-1**

```bash
helm version --short
sudo kubectl get storageclass
sudo kubectl get ns monitoring
sudo kubectl get crd | grep monitoring.coreos.com
df -h /var/lib/rancher
free -h
```

| Line | Want |
| --- | --- |
| `helm version` | v3.8 or newer (v3.x or v4.x) |
| `get storageclass` | `local-path (default)` |
| `get ns monitoring` | `NotFound`. If it exists, something was installed before: `sudo kubectl get all -n monitoring` and decide first |
| `get crd` | Nothing. If CRDs are listed, an older Prometheus Operator was installed; read "Troubleshooting" before going on |
| `df -h` | Well over 30 GB free |
| `free -h` | At least 2 GB available |

If `helm` cannot reach the cluster, put `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` in front of the session once and run helm with `sudo -E` ([Pi-hole on k3s](../apps/pihole.md)).

### Step 2. Label the node that keeps the data

**Run on: server-1**

```bash
sudo kubectl label node server-1 monitoring-host=true
sudo kubectl get nodes -L monitoring-host
```

Only server-1 shows `true`. Without the label, Prometheus and Grafana stay `Pending`.

### Step 3. Namespace, Grafana login and the redirect rule

Make up a Grafana admin password and save it in your password manager first ([Backups and secrets](backups-and-secrets.md)).

**Run on: server-1**, in the root of this repo. Replace `<GRAFANA_ADMIN_PASSWORD>`, keep the single quotes.

```bash
sudo kubectl create namespace monitoring
sudo kubectl -n monitoring create secret generic grafana-admin --from-literal=admin-user=admin --from-literal=admin-password='<GRAFANA_ADMIN_PASSWORD>'
sudo kubectl apply -f files/traefik/middleware-redirect-https.yaml
```

`apply` prints `unchanged` for the pihole and homebridge copies and `created` for `monitoring`. The password never goes in a file in this repo.

### Step 4. Install

**Run on: server-1**, in the root of this repo.

```bash
sudo helm upgrade --install kps oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack --version 92.3.1 -n monitoring -f files/monitoring/kube-prometheus-stack-values.yaml
```

It pulls the chart from GitHub's registry, installs the CRDs, then everything else. It ends with `STATUS: deployed`. The first image pulls on the Pis take a few minutes.

### Step 5. Watch it start

**Run on: server-1**

```bash
sudo kubectl -n monitoring get pods -o wide -w
```

(Ctrl+C when everything is `Running`.) You want:

| Pod | Count | Node |
| --- | --- | --- |
| `kps-grafana-…` | 1, `3/3` | server-1 |
| `prometheus-kps-kube-prometheus-stack-prometheus-0` | 1, `2/2` | server-1 |
| `kps-kube-prometheus-stack-operator-…` | 1 | server-1 |
| `kps-kube-state-metrics-…` | 1 | server-1 |
| `kps-prometheus-node-exporter-…` | 4, one per node | every node, the Mac VM included |

Exact names can differ slightly; the counts and nodes are what matter. Then the volumes:

```bash
sudo kubectl -n monitoring get pvc
```

Two, both `Bound`: Grafana's (2Gi) and Prometheus's (25Gi).

Grafana's first start downloads the extra dashboards from GitHub; if its pod sits in `Init:Error`, see "Troubleshooting".

### Step 6. Pi-hole: the Grafana name and the exporter

`files/pihole/values.yaml` now has the `grafana.home.example.com` line and the exporter. It also carries the DNS settings bogus-private, DNSSEC, the local domain, NTP off and a 24-hour web login ([Pi-hole on k3s](../apps/pihole.md)).

**First, the Pi-hole password Secret.** This file reads the admin password from the Secret `pihole-admin`. If you ever installed Pi-hole with the password written inline in a values file, the Secret may not exist.

**Run on: server-1**

```bash
sudo kubectl -n pihole get secret pihole-admin
```

If it says `NotFound`, create it with the password you log in to Pi-hole with (or a new one, which rotates it). Replace `<PIHOLE_ADMIN_PASSWORD>`, keep the single quotes:

```bash
sudo kubectl -n pihole create secret generic pihole-admin --from-literal=password='<PIHOLE_ADMIN_PASSWORD>'
```

**Then upgrade Pi-hole.** **Run on: server-1**, in the root of this repo.

```bash
helm upgrade --install pihole mojo2600/pihole -n pihole --version 2.38.0 -f files/pihole/values.yaml
sudo kubectl -n pihole rollout status deploy/pihole --timeout=600s
sudo kubectl -n pihole get pods -o wide
```

The pods are replaced one at a time (the PodDisruptionBudget and three replicas keep DNS answering). Each pod now shows `3/3` (Pi-hole, cloudflared, exporter), one per Pi.

**Check the name.** **Run on: any computer at home**

```sh
dig +short grafana.home.example.com @192.168.50.11
```

`192.168.50.12`.

**A laptop on a corporate VPN** may not see home names; add the name to its hosts file. The full line is in `files/clients/work-mac-hosts.txt`; how to replace it: [Client devices](../apps/client-devices.md).

### Step 7. Check the Traefik metrics port

The Traefik PodMonitor scrapes the port named `metrics`. k3s annotates the Traefik pod with port `8082`, which was Traefik v2's; v3 serves metrics on 9100.

**Run on: server-1**

```bash
sudo kubectl -n kube-system get pod -l app.kubernetes.io/name=traefik -o jsonpath='{range .items[0].spec.containers[0].ports[*]}{.name}={.containerPort}{"\n"}{end}'
```

You want a line `metrics=9100` among `web`, `websecure` and `traefik`. If the name is different, change `port: metrics` in the PodMonitor at the end of `files/monitoring/kube-prometheus-stack-values.yaml` to that name and run Step 4 again. If the command prints nothing, the label is different: `sudo kubectl -n kube-system get pods --show-labels | grep traefik`.

### Step 8. Log in and check every target

Open **https://grafana.home.example.com** (accept the certificate warning once, as for Pi-hole). Log in as `admin` with the password from Step 3.

**Check that Prometheus reaches everything.** **Run on: server-1**

```bash
sudo kubectl -n monitoring port-forward svc/prometheus-operated 9090:9090 >/dev/null 2>&1 &
sleep 3
curl -s 'http://127.0.0.1:9090/api/v1/targets?state=active' | python3 -c 'import json,sys;[print(t["health"],t["labels"].get("job"),t["scrapeUrl"]) for t in json.load(sys.stdin)["data"]["activeTargets"]]' | sort
kill %1
```

Every line should start with `up`:

| Job | Lines | Note |
| --- | --- | --- |
| `apiserver` | 4 | One per k3s server, port 6443 |
| `kubelet` | about 12 | Three paths (metrics, cadvisor, probes) on each of four nodes |
| `coredns` | One per CoreDNS pod | Port 9153 |
| `node-exporter` | 4 | Port 9100 on each node's own address |
| `kube-state-metrics` | 1 | |
| `kps-…-operator`, `kps-…-prometheus`, `kps-grafana` | 1 each | The stack watching itself |
| `monitoring/traefik` | One per Traefik pod | Port 9100 (Step 7) |
| `pihole/pihole-prometheus-exporter` | 3 | Port 9617, one per Pi-hole (Step 6) |

A `down` line: "Troubleshooting".

**Then the dashboards.** In Grafana, **Dashboards**:

| Folder | Dashboard | Shows |
| --- | --- | --- |
| Kubernetes | Kubernetes / Views / Global | The whole cluster on one page: nodes, CPU and memory, pods by namespace, network |
| Kubernetes | Views / Namespaces, Views / Pods, Views / Nodes | Drill down. Pick `pihole` in Namespaces to see the three Pi-hole pods side by side |
| Kubernetes | System / API Server, System / CoreDNS | The control plane and cluster DNS |
| Nodes | Node Exporter Full | Each machine in detail: disk use and I/O, temperature, network, load |
| Apps | Traefik Official Kubernetes Dashboard | Requests, errors and response times per entrypoint and per Service behind Traefik: Pi-hole UI, Homebridge, Seerr, Grafana |
| Apps | Pi-hole Exporter | Queries, blocked, clients, top domains, per Pi-hole pod |
| General | The chart's own set (Kubernetes / Compute Resources / …, Networking, Persistent Volumes, Node Exporter, Prometheus, Grafana) | Requests and limits against use, per namespace, workload and pod |

Ingresses: the Traefik dashboard shows traffic per backend Service, which is what an Ingress routes to. kube-state-metrics also keeps `kube_ingress_info` and `kube_ingress_path` (Explore → `kube_ingress_path`) for which hosts and paths exist.

**What it costs.** **Run on: server-1**

```bash
sudo kubectl top pods -n monitoring
sudo kubectl top nodes
```

Write the numbers down the first time, so a later change stands out.

---

## Phase 2 (optional): etcd metrics

Adds the health of the four etcd members (leader changes, slow disk writes, database size): the part of an HA cluster most worth watching. It needs one line in each server's k3s config and a k3s restart, **one server at a time**. Four etcd members tolerate one being down, so one restart at a time never loses the cluster.

### Step 1. One server at a time

On each server in turn: server-1, server-2, server-3, then the Mac VM (`limactl shell k3s-vm` on the Mac). The repo copies in `files/k3s/config/` already have the line.

**Run on: that server**

```bash
grep -q '^etcd-expose-metrics:' /etc/rancher/k3s/config.yaml || echo 'etcd-expose-metrics: true' | sudo tee -a /etc/rancher/k3s/config.yaml
sudo systemctl restart k3s
sleep 30
curl -s "http://$(hostname -I | awk '{print $1}'):2381/metrics" | grep -m1 '^etcd_server_has_leader'
sudo kubectl get nodes
```

You want `etcd_server_has_leader 1` and all four nodes `Ready` **before** moving to the next server. On the Mac VM `hostname -I` lists 192.168.50.146 first only if `lima0` comes first; if the curl prints nothing there, use `curl -s http://192.168.50.146:2381/metrics | grep -m1 '^etcd_server_has_leader'`.

### Step 2. Turn on the etcd scrape

**Run on: server-1**, in the root of this repo.

```bash
sudo helm upgrade --install kps oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack --version 92.3.1 -n monitoring -f files/monitoring/kube-prometheus-stack-values.yaml -f files/monitoring/kube-prometheus-stack-etcd.yaml
```

From now on **always pass both files**, in that order; leaving the second one out turns the etcd scrape off again. Phase 1 Step 8's target check now also lists `kube-etcd` four times, `up`. Grafana's General folder has an **etcd** dashboard.

### Step 3. The node firewall

If the node firewall is on (`sudo ufw status` says `active`), run the updated `files/firewall/k3s-firewall.sh` on each node as [Node firewall](../kubernetes/node-firewall.md) describes. It adds rules that keep 9100 and 2381 away from the LAN while pods and the other nodes still reach them. If the firewall is off, nothing to do now; the rules are in the script for when it goes on.

---

## Check it

**Run on: server-1**

```bash
sudo kubectl -n monitoring get pods,pvc,ingress
sudo helm -n monitoring list
```

All pods `Running`, both PVCs `Bound`, the Ingress on `grafana.home.example.com`, release `kps` at chart `kube-prometheus-stack-92.3.1`.

**Run on: any computer at home**

```sh
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' http://grafana.home.example.com/
curl -sk -o /dev/null -w '%{http_code}\n' https://grafana.home.example.com/login
```

`301 https://grafana.home.example.com/`, then `200`.

In Grafana: **Kubernetes / Views / Global** shows four nodes; the target check in Phase 1 Step 8 is all `up`; **Alerting → Alert rules** shows the rules with few or none firing.

## Pitfalls

- **Installing before the label.** Prometheus, Grafana, the operator and kube-state-metrics stay `Pending`. Step 2.
- **Leaving out the second `-f` after Phase 2.** The etcd scrape quietly disappears.
- **Two Grafanas on one SQLite file.** Don't change `deploymentStrategy` back to `RollingUpdate` and don't raise Grafana's replicas; both can start a second pod against the same database.
- **Putting Grafana on the Cloudflare tunnel.** It is a LAN page. If it ever needs to be reachable from outside, put an identity provider in front (Cloudflare Access) first.
- **Publishing Prometheus.** It has no login. Use the port-forward.
- **Editing provisioned dashboards in the UI.** They are read-only on purpose and come back on every restart. Use **Save as** to make your own copy, which lives in Grafana's volume.
- **`helm upgrade` without `--version`.** It takes whatever is newest, including a new major version with CRD changes. Always pass the version in this file.
- **Deleting the PVCs to "start fresh".** Grafana's holds users and your own dashboards.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `helm`: `failed to do request` / `unsupported protocol scheme` on `oci://` | Helm older than 3.8, or server-1 can't reach `ghcr.io` | `helm version`; update Helm ([Software and firmware](software-and-firmware.md)). `curl -sI https://ghcr.io` from server-1 |
| Pods `Pending`, Events "didn't match Pod's node affinity/selector" | No node has `monitoring-host=true` | Step 2 |
| PVC `Pending` | The local-path provisioner isn't running, or the pod isn't scheduled yet (local-path waits for the pod) | `sudo kubectl -n kube-system get pods -l app=local-path-provisioner`; check the pod first |
| `kps-grafana` stuck in `Init:Error` or `Init:CrashLoopBackOff` | The `download-dashboards` init container couldn't fetch a dashboard from GitHub (no DNS, no internet, a URL changed) | `sudo kubectl -n monitoring logs deploy/kps-grafana -c download-dashboards`. A 404 names the dashboard; fix its URL in the values file and Step 4. Nodes use 1.1.1.1 for their own DNS ([HA k3s cluster](../kubernetes/k3s-ha-cluster.md)) |
| Grafana login fails | The Secret's keys aren't `admin-user` and `admin-password`, or the password differs from what you type | `sudo kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data}'` (key names only). Recreate the Secret, then `sudo kubectl -n monitoring rollout restart deploy/kps-grafana`. The password is only read on Grafana's first start with an empty database; after that change it in the UI or with `grafana cli admin reset-admin-password` inside the pod |
| `https://grafana.home.example.com` → 404 from Traefik | No Ingress, or the host differs | `sudo kubectl -n monitoring get ingress`. Host must be exactly `grafana.home.example.com` |
| The browser loops between http and https | Middleware missing in `monitoring`, or `root_url` changed | `sudo kubectl -n monitoring get middleware`; Step 3 |
| Name doesn't resolve | Pi-hole not upgraded yet, or (a laptop on a corporate VPN) its hosts file | Step 6 |
| `traefik` target `down` | The port name isn't `metrics`, or Traefik's metrics entrypoint was turned off | Step 7 |
| `pihole-prometheus-exporter` targets `down` | Exporter container not running, or it can't log in | `sudo kubectl -n pihole logs deploy/pihole -c exporter --tail=30`. "Unauthorized": the `pihole-admin` Secret doesn't match Pi-hole's password (Step 6) |
| Pi-hole pods stuck `CreateContainerConfigError` after Step 6 | The `pihole-admin` Secret doesn't exist | Step 6, the Secret. The old pods keep answering DNS until the new ones start |
| `node-exporter` on `192.168.50.146` `down` | The Mac VM is off or asleep, or the firewall on it | `sudo kubectl get nodes`. The Mac sleeping takes the node away ([Mac Lima VM](../hardware/mac-lima-vm.md)) |
| `kube-etcd` targets `down` (Phase 2) | That server hasn't got `etcd-expose-metrics: true` yet, or k3s wasn't restarted | Phase 2 Step 1 on that server |
| Lots of "target down" alerts for controller-manager, scheduler or proxy | The values file wasn't used, or `defaultRules` was changed | Step 4 with this repo's file |
| Prometheus `OOMKilled` | More series than 2 GiB holds | `limits.memory` up to 3Gi in the values file (server-1 has the room), Step 4 |
| `prometheus-…-0` restarts with "no space left" | The 25 Gi volume filled before `retentionSize` caught it | Lower `retentionSize`, or raise the volume size (local-path doesn't enforce sizes, the disk does) |
| CRDs already present (Step 1) | An older Prometheus Operator | Its CRDs must match v0.94.1: read `UPGRADE.md` in the chart repo, or remove the old install first |

## Updates

Pinned on purpose. To move to a newer chart:

1. Look up the newest version: https://github.com/prometheus-community/helm-charts/releases (tags `kube-prometheus-stack-…`).
2. A new **major** version (93.x after 92.x) changes CRDs. Read `UPGRADE.md` in the chart folder first and apply the CRDs it names before the upgrade.
3. Change `--version` everywhere on this page and in the values file's header, run Step 4 (with both files after Phase 2), then "Check it".
4. Record it in [Software and firmware](software-and-firmware.md).

Dashboards: change the tag or commit in their URLs, Step 4, then `sudo kubectl -n monitoring rollout restart deploy/kps-grafana`.

## Backups

| What | Where | Back up? |
| --- | --- | --- |
| Grafana admin password | Password manager | Yes, it's only there and in the Secret |
| Grafana's database (users, your own dashboards) | PVC on server-1, under `/var/lib/rancher/k3s/storage/` | Only if you build dashboards in the UI. Better: export them (**Share → Export**) into this repo |
| Prometheus data | PVC on server-1 | No. It's 15 days of numbers and refills by itself |
| Everything else | This repo | Already |

## Undo

**Run on: server-1**

```bash
sudo helm -n monitoring uninstall kps
sudo kubectl -n monitoring delete pvc --all
sudo kubectl delete namespace monitoring
```

CRDs stay (the chart's README); delete them with the list in the chart README's "Uninstall Helm Chart" only if nothing else uses them. Then in `files/pihole/values.yaml` set `monitoring.podMonitor.enabled` and `monitoring.sidecar.enabled` to `false` and upgrade Pi-hole (Step 6); otherwise the Pi-hole upgrade fails on the missing `PodMonitor` type. Remove the label: `sudo kubectl label node server-1 monitoring-host-`.

## Later

- **Alerts to your phone.** Turn on Alertmanager with a receiver (ntfy, Pushover, email) and route `critical` there; or use Grafana's own alerting contact points.
- **Logs.** Loki plus Grafana Alloy would let Grafana search logs from every pod, Pi-hole's query log included. The heaviest piece; worth it only with alerts in place.
- **Pi-hole with one read/write primary and read-only replicas** (Pi-hole's `misc.readOnly`, a sync tool such as nebula-sync), if you edit lists in the web UI often. Never share `/etc/pihole` between pods.
- **After a k3s upgrade** the bundled Traefik changes version. Check Step 7 again afterwards.

## References

Opened on 13 October 2026.

- kube-prometheus-stack chart (README, `values.yaml`, `Chart.yaml` at 92.3.1): https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
- Releases: https://github.com/prometheus-community/helm-charts/releases
- Grafana chart (values: `admin.existingSecret`, `persistence`, `deploymentStrategy`, `dashboards`): https://github.com/grafana-community/helm-charts/tree/main/charts/grafana
- The old Grafana chart and its migration notice: https://github.com/grafana/helm-charts/tree/main/charts/grafana ; Grafana's Helm docs: https://grafana.com/docs/helm-charts/
- k3s metrics, one registry per process: https://docs.k3s.io/reference/metrics
- k3s server flags (`etcd-expose-metrics`): https://docs.k3s.io/cli/server
- k3s etcd metrics listener (`pkg/etcd/etcd.go`, v1.34.3+k3s1): https://github.com/k3s-io/k3s/blob/v1.34.3%2Bk3s1/pkg/etcd/etcd.go
- k3s bundled Traefik manifest (v1.34.3+k3s1): https://github.com/k3s-io/k3s/blob/v1.34.3%2Bk3s1/manifests/traefik.yaml
- Traefik chart 37.1.0 values (metrics port 9100): https://github.com/traefik/traefik-helm-chart/blob/v37.1.0/traefik/values.yaml
- Kubernetes dashboards: https://github.com/dotdc/grafana-dashboards-kubernetes
- Node Exporter Full: https://github.com/rfmoz/grafana-dashboards and https://grafana.com/grafana/dashboards/1860
- Traefik dashboards: https://grafana.com/grafana/dashboards/17347-traefik-official-kubernetes-dashboard/
- Pi-hole exporter: https://github.com/eko/pihole-exporter ; dashboard https://grafana.com/grafana/dashboards/10176-pi-hole-exporter/
- MoJo2600 Pi-hole chart (monitoring values, sidecar template): https://github.com/MoJo2600/pihole-kubernetes/blob/master/charts/pihole/README.md
- Sizing on Pis (retention on 2 GB nodes): https://forums.suse.com/t/lightweight-metrics-options-for-k3s-pi-cluster/18796
