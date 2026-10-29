# Demo Helm Chart

A small nginx chart used to practice the full path from an empty GitHub repository to a Helm chart repository.

Helm does not install a folder of YAML files from GitHub directly. It installs a **package** (a `.tgz` file) listed in an `index.yaml` catalog. This repo keeps both:

| File | What it is |
| --- | --- |
| `demo/` | Chart source (`Chart.yaml`, `values.yaml`, `templates/`) |
| `demo-0.1.0.tgz` | The packaged chart (`helm package`) |
| `index.yaml` | The catalog Helm reads (`helm repo index`) |

GitHub Pages serves those two files. That published URL is the Helm repository.

The chart deploys one nginx replica and a ClusterIP Service on port 80. It is only for testing the packaging flow.

---

## Prerequisites

Install these before you start:

- [Git](https://git-scm.com/downloads)
- [Helm](https://helm.sh/docs/intro/install/)
- A GitHub account
- A Kubernetes cluster and `kubectl`, only if you want to install the chart (Docker Desktop, minikube, or kind)

Check the tools:

```bash
git --version
helm version
```

---

## 1. Create the GitHub repository

1. Open [github.com/new](https://github.com/new).
2. Repository name: `demo-helm-chart`
3. Visibility: **Public**. GitHub Pages on a free account serves a public repository.
4. Leave **Add a README** unchecked. You will push this folder yourself.
5. Click **Create repository**.
6. Copy the repository URL. It looks like:

```text
https://github.com/<username>/demo-helm-chart.git
```

For this project the username is `johnbedeir`, so the URL is:

```text
https://github.com/johnbedeir/demo-helm-chart.git
```

---

## 2. Put the project under Git

On your machine, go to the folder that should become the repository (this folder):

```bash
cd demo-helm-chart
git init
git branch -M main
git remote add origin https://github.com/<username>/demo-helm-chart.git
```

If you created the empty repository first and prefer to clone it, use that instead, then do the next steps inside the clone:

```bash
git clone https://github.com/<username>/demo-helm-chart.git
cd demo-helm-chart
```

---

## 3. Turn the repository into a Helm chart

`helm create` writes a valid chart. The folder name becomes the chart name.

```bash
helm create demo
```

That creates:

```text
demo/
  Chart.yaml          # chart name, version, description
  values.yaml         # defaults (image, replicas, service)
  .helmignore         # files Helm skips when packaging
  templates/          # Kubernetes manifests with {{ }} placeholders
```

Open `demo/Chart.yaml`. Two versions matter:

- `version` is the **chart** version. Helm uses this in the package filename (`demo-0.1.0.tgz`). Bump it every time you publish a change.
- `appVersion` is the version of the app you are deploying. The default image tag follows this value.

The generated chart uses the public `nginx` image, so you can install it without building anything.

Check the chart before you package it:

```bash
helm lint demo
```

Render the manifests locally if you want to see the YAML Helm would apply:

```bash
helm template my-demo ./demo
```

---

## 4. Package the chart

Packaging turns the `demo/` folder into one archive. Run this from the repository root, not from inside `demo/`:

```bash
helm package demo
```

You should get:

```text
demo-0.1.0.tgz
```

The name is `<chart name>-<version>.tgz` from `Chart.yaml`. This archive is the Helm package.

---

## 5. Build the Helm repository index

A package alone is not a Helm repository. Helm looks for `index.yaml`, which lists every package and its download URL.

The URL must be the GitHub Pages address, not the `github.com` page:

```bash
helm repo index . --url https://<username>.github.io/demo-helm-chart
```

Example for this repository:

```bash
helm repo index . --url https://johnbedeir.github.io/demo-helm-chart
```

That writes `index.yaml` in the same folder as `demo-0.1.0.tgz`. The `urls` entry inside it points at the package, for example:

```text
https://johnbedeir.github.io/demo-helm-chart/demo-0.1.0.tgz
```

This folder already contains `demo-0.1.0.tgz` and `index.yaml` from that command. If your GitHub username or repository name is different, run `helm repo index` again with your URL before you push.

---

## 6. Commit and push

```bash
git add .
git commit -m "Add demo Helm chart package"
git push -u origin main
```

Push the chart source, the `.tgz`, and `index.yaml`. GitHub Pages serves the files that are on the branch you select in the next step.

---

## 7. Turn the repository into a Helm repository with GitHub Pages

1. Open the repository on GitHub.
2. Go to **Settings** → **Pages**.
3. Under **Build and deployment**, set **Source** to **Deploy from a branch**.
4. Branch: `main`. Folder: `/` (root).
5. Save.

After a minute or two, the site is:

```text
https://<username>.github.io/demo-helm-chart
```

Confirm the catalog is public:

```bash
curl https://<username>.github.io/demo-helm-chart/index.yaml
```

You should see the `demo` chart and version `0.1.0`.

---

## 8. Install from the GitHub Helm repository

Add the Pages URL as a Helm repo, then install by chart name:

```bash
helm repo add demo https://<username>.github.io/demo-helm-chart
helm repo update
helm search repo demo
helm install my-demo demo/demo
```

Check that nginx is running:

```bash
kubectl get pods
kubectl get svc
```

Port-forward if you want to open it in a browser:

```bash
kubectl port-forward svc/my-demo 8080:80
```

Then open `http://localhost:8080`.

Remove the release when you are done:

```bash
helm uninstall my-demo
```

---

## Test locally before GitHub Pages

You can install the chart from the folder, with no GitHub repository:

```bash
helm install my-demo ./demo
helm uninstall my-demo
```

Use this when you only want to confirm the templates. The GitHub Pages steps above are what make `helm repo add` work.

---

## Publish a new version

1. Change the chart or `demo/values.yaml`.
2. Bump `version` in `demo/Chart.yaml` (for example `0.1.0` → `0.1.1`).
3. Package and refresh the index from the repository root. Keep the old `.tgz` files so older versions stay installable.

```bash
helm package demo
helm repo index . --url https://<username>.github.io/demo-helm-chart
git add .
git commit -m "Release demo chart 0.1.1"
git push
```

4. Wait for GitHub Pages to update, then:

```bash
helm repo update
helm upgrade my-demo demo/demo
```
