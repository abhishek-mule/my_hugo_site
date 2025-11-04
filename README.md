# Deploy a Hugo Website with Cloud Build and Firebase Pipeline


## Overview

Create an automated pipeline that builds and deploys a Hugo static website to Firebase Hosting. Website content is stored in a GitHub repository and deployments are automated by Google Cloud Build when commits land on the `main` branch.

## Objectives

* Read a static website overview.
* Set up a Hugo site locally on a Linux VM.
* Store site content in a GitHub repository.
* Deploy the site to Firebase Hosting manually.
* Create a Cloud Build pipeline to automate build + deploy on commits.

## Prerequisites

* Personal GitHub account.
* Basic familiarity with git, GitHub, Google Cloud Console, and the Linux shell.
* Use an Incognito/private browser when running the lab (to avoid account conflicts).

---

## Why static sites + pipeline?

Static site builders (Hugo) produce pre-built assets—no web server runtime required. A CI/CD pipeline handles versioning, CDN hosting, SSL provisioning and lets you deploy reliably and repeatably.

---

## Quick architecture

Commit → GitHub repo → Cloud Build trigger → Cloud Build downloads Hugo & Firebase CLI → build static files → `firebase deploy` → Firebase Hosting (CDN + SSL)

---

## Task 1 — Manual deployment (learn the flow)

1. **Open the Linux VM**

   * Console: Compute Engine → VM Instances → SSH into provided instance.
   * Note the VM external IP (used to preview Hugo server).

2. **Install Hugo** (script provided)

```bash
cd ~
cat /tmp/installhugo.sh   # inspect
/tmp/installhugo.sh       # installs Hugo into /tmp/hugo
```

3. **Create GitHub repo & clone**

```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
export REGION=$(gcloud compute project-info describe --format="value(commonInstanceMetadata.items[google-compute-default-region])")
sudo apt-get update
sudo apt-get install git gh
# auth GitHub CLI
gh auth login
GITHUB_USERNAME=$(gh api user -q ".login")
# create + clone
gh repo create my_hugo_site --private
gh repo clone my_hugo_site
```

4. **Create site & theme**

```bash
cd ~/my_hugo_site
/tmp/hugo new site . --force
git clone https://github.com/rhazdon/hugo-theme-hello-friend-ng.git themes/hello-friend-ng
echo 'theme = "hello-friend-ng"' >> config.toml
# remove nested git metadata so theme files are tracked by your repo
sudo rm -r themes/hello-friend-ng/.git
sudo rm themes/hello-friend-ng/.gitignore
```

5. **Preview locally**

```bash
/tmp/hugo server -D --bind 0.0.0.0 --port 8080
# browse to http://[EXTERNAL_IP]:8080
```

6. **Install Firebase CLI & initialize**

```bash
curl -sL https://firebase.tools | bash
cd ~/my_hugo_site
firebase init  # choose Hosting, Use existing project, public dir: public, SPA? N
/tmp/hugo && firebase deploy
# copy the hosting URL displayed after deploy
```

---

## Task 2 — Automate with Cloud Build

### 1. Initial commit

```bash
git config --global user.name "hugo"
git config --global user.email "hugo@blogger.com"
cd ~/my_hugo_site
echo "resources" >> .gitignore
git add .
git commit -m "Add app to GitHub Repository"
git push -u origin main
```

### 2. Add `cloudbuild.yaml`

Copy the provided build file (already placed in `/tmp`):

```bash
cd ~/my_hugo_site
cp /tmp/cloudbuild.yaml .
cat cloudbuild.yaml
```

**What `cloudbuild.yaml` does (high level):**

* Uses `curl` builders to fetch the Hugo tarball and Firebase CLI in parallel.
* Uses an `ubuntu:20.04` step to unpack tools, run Hugo and call `/tmp/firebase deploy --project ${PROJECT_ID} --non-interactive --only hosting -m "Build ${BUILD_ID}"`.
* Uses substitution variable `_HUGO_VERSION` and `PROJECT_ID`.

Sample important fragment (already in your repo):

```yaml
steps:
- name: 'gcr.io/cloud-builders/curl'
  args: ['--quiet','-O','firebase','https://firebase.tools/bin/linux/latest']
- name: 'gcr.io/cloud-builders/curl'
  args: ['--quiet','-O','hugo.tar.gz','https://github.com/gohugoio/hugo/releases/download/v${_HUGO_VERSION}/hugo_extended_${_HUGO_VERSION}_Linux-64bit.tar.gz']
- name: 'ubuntu:20.04'
  args: ['bash','-c','mv hugo.tar.gz /tmp; tar -C /tmp -xzf /tmp/hugo.tar.gz; mv firebase /tmp; chmod 755 /tmp/firebase; /tmp/hugo; /tmp/firebase deploy --project ${PROJECT_ID} --non-interactive --only hosting -m "Build ${BUILD_ID}"']
substitutions:
  _HUGO_VERSION: 0.96.0
```

### 3. Connect GitHub → Cloud Build

```bash
# create connection
gcloud builds connections create github cloud-build-connection --project=$PROJECT_ID --region=$REGION
# describe to get actionUri, then follow link to authorize Cloud Build GitHub App
gcloud builds connections describe cloud-build-connection --region=$REGION
# After installing GitHub App, create repository mapping
gcloud builds repositories create hugo-website-build-repository \
  --remote-uri="https://github.com/${GITHUB_USERNAME}/my_hugo_site.git" \
  --connection="cloud-build-connection" --region=$REGION
```

**Important:** When authorizing, install the Cloud Build GitHub App and grant access only to `my_hugo_site` (or desired repo).

### 4. Create a trigger

Create a Cloud Build trigger that listens for commits to `main`:

```bash
gcloud builds triggers create github --name="commit-to-main-branch1" \
   --repository=projects/$PROJECT_ID/locations/$REGION/connections/cloud-build-connection/repositories/hugo-website-build-repository \
   --build-config='cloudbuild.yaml' \
   --service-account=projects/$PROJECT_ID/serviceAccounts/$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
   --region=$REGION \
   --branch-pattern='^main$'
```

### 5. Grant Cloud Build service account permissions

Give Cloud Build service account the Firebase Hosting admin role so it can deploy:

* Service account: `PROJECT_NUMBER@cloudbuild.gserviceaccount.com`
* Role: `roles/firebasehosting.admin`

(Use IAM in console or `gcloud projects add-iam-policy-binding`.)

---

## Test the pipeline

1. Make a change locally (e.g., `config.toml` site title) and push:

```bash
cd ~/my_hugo_site
# edit config.toml
git add .
git commit -m "I updated the site title"
git push origin main
```

2. Check build status & logs:

```bash
gcloud builds list --region=$REGION
# find latest build ID then:
gcloud builds log --region=$REGION <BUILD_ID>
# to grep hosting URL from logs:
gcloud builds log "$(gcloud builds list --format='value(ID)' --filter=$(git rev-parse HEAD) --region=$REGION)" --region=$REGION | grep "Hosting URL"
```

3. Open the Hosting URL (may take a few minutes for CDN/SSL to propagate).

---

## Troubleshooting & tips

* If you see a generic Firebase welcome page after deploy, wait a few minutes and refresh (CDN propagation).
* Ensure GitHub App installation completed and Cloud Build connection shows `ACTIVE`.
* Check Cloud Build logs for step failures; tools are downloaded each build so network errors show up early.
* Upgrade `_HUGO_VERSION` in `cloudbuild.yaml` to change Hugo version used by pipeline.

---

## Summary

You built a Hugo site, deployed it manually to Firebase, and automated build + deploy using Cloud Build with a GitHub-connected trigger. Future pushes to `main` will run the pipeline and update your site on Firebase Hosting.

---

*Happy deploying!*
