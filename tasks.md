# DevOps Roadmap

> **How to use this guide:** Each chapter starts with tasks. Do them first. Hit the wall. Then read on — the tool or concept that follows exists specifically to solve the problem you just ran into.

---

## Prerequisites

Install these before starting. You won't need all of them immediately — each chapter will tell you what's required.

- Python 3.10+
- Git
- VirtualBox 7.x
- Docker + Docker Compose plugin
- Terraform 1.5+ — https://developer.hashicorp.com/terraform/install

---

## The App

You'll work with a single Python web app throughout this roadmap, adding to it as you go. Create it now.

```
mkdir myapp && cd myapp
```

`app.py`:
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/status")
def status():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

Install Flask and run it:
```
pip install flask
python app.py
```

In a second terminal:
```
curl http://localhost:8080/status
```

You should see `{"status": "ok"}`. Leave the app running.

---

## Chapter 1: Version Control

### Make some changes

Make the following changes **one at a time**, in this order. Don't track anything, don't save backups — just edit the file.

**Change 1.** Add a version field to the response:
```python
return jsonify({"status": "ok", "version": 1})
```

**Change 2.** Write a test. Create `test_app.py`:
```python
import pytest
from app import app

@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as client:
        yield client

def test_status(client):
    response = client.get("/status")
    assert response.status_code == 200
    assert response.json["status"] == "ok"
```

Install pytest and run the test:
```
pip install pytest
pytest
```

It passes.

**Change 3.** Rename the field — your API consumer wants `health` instead of `status`:
```python
return jsonify({"health": "ok", "version": 1})
```

Also update the test to match the new field name:
```python
assert response.json["health"] == "ok"
```

**Change 4.** Add a response header. Rewrite the endpoint:
```python
from flask import Flask, jsonify, make_response

@app.get("/status")
def status():
    resp = make_response(jsonify({"health": "ok", "version": 1}))
    resp.headers["X-App-Name"] = "myapp"
    return resp
```

**Change 5.** Two more headers requested. Add them:
```python
resp.headers["X-Environment"] = "development"
resp.headers["X-Request-Id"] = "static-id-for-now"
```

Now run the tests:
```
pytest
```

They fail. The test checks for `status`, which you renamed to `health` two changes ago.

> What was the last state where both the feature and the test worked? Scroll up in your terminal. Look at the file. You've overwritten it four times. You don't know.

This is the problem version control solves.

---

### Enter Git

Start over. Undo all your changes — restore `app.py` to the original single-line response and delete `test_app.py`. Now repeat the same changes, but commit after each one.

Initialize the repository:
```
git init
git add app.py
git commit -m "initial commit: status endpoint"
```

**Change 1** — add version field:
```
git add app.py
git commit -m "add version field to status response"
```

**Change 2** — add the test file:
```
git add test_app.py
git commit -m "add test for status endpoint"
```

Run `pytest` — it passes. Commit it while it's green.

**Change 3** — rename field, update the test to match:
```
git add app.py test_app.py
git commit -m "rename status to health per API spec"
```

Run `pytest` — still passes. Good.

**Change 4 and 5** — add headers:
```
git add app.py
git commit -m "add response headers: X-App-Name, X-Environment, X-Request-Id"
```

Now break it intentionally to see recovery in action:
```python
return jsonify({"state": "running", "version": 2})   # oops: test asserts "health" key, which is now missing
```

Tests fail. Find when they last passed:
```
git log --oneline
```

You have a complete history of every working state. Go back to any one of them:
```
git stash                          # save your current broken state
git checkout <commit-hash>         # step back in time
pytest                             # passes
git checkout main                  # come back to now
git stash pop                      # restore your broken changes
```

See exactly what changed between two commits:
```
git diff HEAD~1 HEAD               # last commit vs the one before
git diff <hash-a> <hash-b>         # any two points in history
```

Fix the broken test, commit the fix. From this point on — commit every time something works. Small, frequent commits.

---

## Chapter 2: Dependency Management

Your app works. A colleague wants to run it. You tell them: "install Flask and pytest, then run it."

### Feel the problem

Simulate a fresh machine:
```
cd ..
mkdir myapp-fresh && cd myapp-fresh
cp ../myapp/app.py .
cp ../myapp/test_app.py .
pip install flask pytest
pytest
python app.py
```

Does it work? Probably. Now check what you installed:
```
pip show flask
```

Go back and check your original:
```
cd ../myapp
pip show flask
```

Same version? Maybe — if you installed at the same time. If your colleague installs a month from now, Flask may have released a new version. Flask 2.x to 3.x has breaking changes. And Flask itself depends on other packages, which depend on others. You have no control over any of it.

Now imagine this with 20 dependencies. Some of them conflict with each other depending on which version gets picked. One of them has a bug in the version that happens to be "latest" today.

> "Works on my machine" is not a fluke. It's the natural result of not declaring your environment.

### Enter dependency pinning

Back in `myapp`:
```
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install flask pytest
pip freeze > requirements.txt
cat requirements.txt
```

Every package, every transitive dependency, pinned to an exact version. This file *is* your environment.

Now simulate your colleague again:
```
cd ../myapp-fresh
python -m venv venv
source venv/bin/activate
pip install -r ../myapp/requirements.txt
pytest
python app.py
```

Identical versions. Guaranteed. No surprises.

Add `requirements.txt` to your repo. First create `.gitignore`:
```
echo "venv/" > .gitignore
echo "__pycache__/" >> .gitignore
echo "*.pyc" >> .gitignore
```

Then commit both:
```
cd ../myapp
git add requirements.txt .gitignore
git commit -m "pin dependencies, add venv to gitignore"
```

You never commit the virtual environment, only the lockfile — `.gitignore` keeps `venv/` out of the repo.

> From now on: every project starts with a venv. Every dependency goes into requirements.txt. The file is the truth.

---

## Chapter 3: Infrastructure Provisioning

Your app runs locally. Time to run it somewhere else. You need three machines: Ubuntu 22.04, Fedora 38, and Alpine 3.19.

### Feel the problem

Do this manually for all three. Open VirtualBox. For each machine, create a new VM — download the ISO for that distro, create the VM, attach the ISO, go through the installer, wait for it to boot, find the IP address, SSH in.

Once you're in, install Python and your app:

**Ubuntu 22.04:**
```
ssh youruser@<ubuntu-ip>
sudo apt update
sudo apt install -y python3 python3-pip
```

Copy your app over and run it:
```
# run from your host machine
scp -r ~/myapp youruser@<ubuntu-ip>:~/
ssh youruser@<ubuntu-ip> "cd myapp && pip3 install -r requirements.txt && python3 app.py"
```

Verify:
```
curl http://<ubuntu-ip>:8080/status
```

**Fedora 38:**
```
ssh youruser@<fedora-ip>
sudo dnf install -y python3 python3-pip
```

`apt` is now `dnf`. The flags differ. Copy and run:
```
scp -r ~/myapp youruser@<fedora-ip>:~/
ssh youruser@<fedora-ip> "cd myapp && pip3 install -r requirements.txt && python3 app.py"
```

Verify:
```
curl http://<fedora-ip>:8080/status
```

**Alpine 3.19:**
```
ssh youruser@<alpine-ip>
sudo apk update
sudo apk add python3 py3-pip
```

`apk` now. Different package names. Alpine uses `musl` instead of `glibc` — some Python packages that compile C extensions will behave differently or fail to install entirely. Copy and run:
```
scp -r ~/myapp youruser@<alpine-ip>:~/
ssh youruser@<alpine-ip> "cd myapp && pip3 install -r requirements.txt && python3 app.py"
```

Verify:
```
curl http://<alpine-ip>:8080/status
```

> Write down every step you took for each machine. Be honest — did you do them in the same order? Did you make the same choices in the VirtualBox installer? Is Python the same version on all three? Did `pip install` produce the same result on Alpine as on Ubuntu?

You can't guarantee it. Each machine was provisioned by hand, differently, with no record. If one of them breaks, you don't know how to rebuild it. If a colleague needs to reproduce it, they have to start from scratch.

### Enter Terraform

Make sure VirtualBox is installed. Before running Terraform, create a host-only network if you don't have one:
```
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0
```

> Note: check Vagrant Cloud (https://app.vagrantup.com) for current box versions before running. The URLs below use pinned versions — substitute newer ones if the download fails.

Create a new directory alongside your app:
```
cd ..
mkdir infra && cd infra
```

`main.tf`:
```hcl
terraform {
  required_providers {
    virtualbox = {
      source  = "terra-farm/virtualbox"
      version = "0.2.1"
    }
  }
}

resource "virtualbox_vm" "ubuntu" {
  name   = "ubuntu-01"
  image  = "https://app.vagrantup.com/ubuntu/boxes/focal64/versions/20230119.0.1/providers/virtualbox.box"
  cpus   = 1
  memory = "1024 mib"

  network_adapter {
    type           = "hostonly"
    host_interface = "vboxnet0"
  }
}

resource "virtualbox_vm" "fedora" {
  name   = "fedora-01"
  image  = "https://app.vagrantup.com/generic/boxes/fedora38/versions/4.3.12/providers/virtualbox.box"
  cpus   = 1
  memory = "1024 mib"

  network_adapter {
    type           = "hostonly"
    host_interface = "vboxnet0"
  }
}

resource "virtualbox_vm" "alpine" {
  name   = "alpine-01"
  image  = "https://app.vagrantup.com/generic/boxes/alpine319/versions/4.3.12/providers/virtualbox.box"
  cpus   = 1
  memory = "512 mib"

  network_adapter {
    type           = "hostonly"
    host_interface = "vboxnet0"
  }
}

output "ubuntu_ip" {
  value = virtualbox_vm.ubuntu.network_adapter.0.ipv4_address
}

output "fedora_ip" {
  value = virtualbox_vm.fedora.network_adapter.0.ipv4_address
}

output "alpine_ip" {
  value = virtualbox_vm.alpine.network_adapter.0.ipv4_address
}
```

```
terraform init
terraform plan
terraform apply
```

Terraform downloads all three boxes and provisions all three VMs. Watch the output — it runs them in parallel where possible. When it finishes:

```
terraform output
```

You get the IP address of each machine. No clicking, no guessing, no writing IPs down from a console screen.

Try changing Alpine's memory to `"1024 mib"` and running:
```
terraform plan
```

It shows exactly one change: Alpine's memory. Ubuntu and Fedora are untouched. Apply it:
```
terraform apply
```

This is **declarative infrastructure**: you describe the desired state, Terraform computes the diff, applies only what changed.

### Shipping code to the provisioned VMs

Deploy your app to all three. The boxes use `vagrant` as the default user with SSH key auth — check the specific box documentation on Vagrant Cloud if login fails.

```
UBUNTU_IP=$(terraform output -raw ubuntu_ip)
FEDORA_IP=$(terraform output -raw fedora_ip)
ALPINE_IP=$(terraform output -raw alpine_ip)
```

Copy the app to each and get it running:

**Ubuntu:**
```
scp -r ~/myapp vagrant@$UBUNTU_IP:~/
ssh vagrant@$UBUNTU_IP "sudo apt update && sudo apt install -y python3 python3-pip && cd myapp && pip3 install -r requirements.txt && nohup python3 app.py > app.log 2>&1 &"
curl http://$UBUNTU_IP:8080/status
```

**Fedora:**
```
scp -r ~/myapp vagrant@$FEDORA_IP:~/
ssh vagrant@$FEDORA_IP "sudo dnf install -y python3 python3-pip && cd myapp && pip3 install -r requirements.txt && nohup python3 app.py > app.log 2>&1 &"
curl http://$FEDORA_IP:8080/status
```

**Alpine:**
```
scp -r ~/myapp vagrant@$ALPINE_IP:~/
ssh vagrant@$ALPINE_IP "sudo apk add python3 py3-pip && cd myapp && pip3 install --break-system-packages -r requirements.txt && nohup python3 app.py > app.log 2>&1 &"
curl http://$ALPINE_IP:8080/status
```

The `nohup ... &` keeps the process running after the SSH session closes. Without it the app dies the moment you disconnect.

You needed three different sets of commands: `apt` vs `dnf` vs `apk`, different package names, different pip flags. Write them down. Now ask yourself: if these VMs were destroyed and you had to rebuild them, would you remember the exact steps? Would a colleague?

Terraform removed the clicking. But the install steps still live only in your head — or in a notes file that nobody will find.

### Encoding provisioning in Terraform

Destroy the VMs:
```
terraform destroy
```

Now capture the manual steps you just ran as code. Create a provisioning script for each distro in the `infra` directory:

`user_data_ubuntu.sh`:
```bash
#!/bin/sh
apt-get update -y
apt-get install -y python3 python3-pip docker.io docker-compose
systemctl enable docker
systemctl start docker
usermod -aG docker vagrant
```

`user_data_fedora.sh`:
```bash
#!/bin/sh
dnf install -y python3 python3-pip docker docker-compose
systemctl enable docker
systemctl start docker
usermod -aG docker vagrant
```

`user_data_alpine.sh`:
```bash
#!/bin/sh
apk update
apk add python3 py3-pip docker docker-compose
rc-update add docker boot
service docker start
addgroup vagrant docker
```

These run once on first boot. Now wire them into `main.tf` — add `user_data` to each resource:

```hcl
resource "virtualbox_vm" "ubuntu" {
  name      = "ubuntu-01"
  image     = "https://app.vagrantup.com/ubuntu/boxes/focal64/versions/20230119.0.1/providers/virtualbox.box"
  cpus      = 1
  memory    = "1024 mib"
  user_data = file("${path.module}/user_data_ubuntu.sh")

  network_adapter {
    type           = "hostonly"
    host_interface = "vboxnet0"
  }
}

resource "virtualbox_vm" "fedora" {
  name      = "fedora-01"
  image     = "https://app.vagrantup.com/generic/boxes/fedora38/versions/4.3.12/providers/virtualbox.box"
  cpus      = 1
  memory    = "1024 mib"
  user_data = file("${path.module}/user_data_fedora.sh")

  network_adapter {
    type           = "hostonly"
    host_interface = "vboxnet0"
  }
}

resource "virtualbox_vm" "alpine" {
  name      = "alpine-01"
  image     = "https://app.vagrantup.com/generic/boxes/alpine319/versions/4.3.12/providers/virtualbox.box"
  cpus      = 1
  memory    = "512 mib"
  user_data = file("${path.module}/user_data_alpine.sh")

  network_adapter {
    type           = "hostonly"
    host_interface = "vboxnet0"
  }
}
```

Apply again:
```
terraform apply
```

The VMs come up with Python and Docker already installed. No SSH, no manual commands. Anyone can run `terraform apply` and get identical machines.

> The provisioning steps are now in files, version-controlled alongside the infrastructure definition. `terraform destroy && terraform apply` is a full rebuild from scratch with no institutional knowledge required.

Before committing, add a `.gitignore` in the `infra/` directory to keep Terraform's local state and cache out of the repo:
```
cd ../infra
echo ".terraform/" > .gitignore
echo "terraform.tfstate" >> .gitignore
echo "terraform.tfstate.backup" >> .gitignore
```

Then commit:
```
cd ../myapp
git add ../infra/
git commit -m "add terraform config with provisioning scripts"
```

---

## Chapter 4: Containerization

You got the app running on Ubuntu and Fedora. Alpine was already giving you trouble with pip. Even where it "worked", you can't be sure it works *the same* — different libc, different SSL library, different Python build, different default locale.

The deeper problem: you're shipping code, but what actually runs is code + Python + system libraries + OS. You only control the first part. The rest depends on what happens to be installed on the machine.

> Every time you write `apt install` or `dnf install` in a deploy script, you're building a fragile dependency on the host. That script will break the moment the distro updates a package or you move to a different machine.

### Enter Docker

Docker packages your code together with its entire runtime: the base OS, Python, all libraries. The result is an image that runs identically regardless of what's on the host — as long as Docker is installed.

Back in `myapp`, create a `Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]
```

Build the image and run it locally:
```
docker build -t myapp:latest .
docker run -p 8080:8080 myapp:latest
```

```
curl http://localhost:8080/status
```

Works. Now run your tests inside the same image — not on your machine, inside the container:
```
docker run myapp:latest pytest
```

If tests pass in the image, they'll pass in production. The image is the environment.

### Ship the image to the VMs

Spin your VMs back up:
```
cd ../infra && terraform apply
UBUNTU_IP=$(terraform output -raw ubuntu_ip)
FEDORA_IP=$(terraform output -raw fedora_ip)
ALPINE_IP=$(terraform output -raw alpine_ip)
```

Docker is already installed on all three — you encoded those install commands into `user_data` scripts at the end of chapter 3. This is the payoff: you felt the pain of running different commands per distro, then captured that knowledge as code so it never has to be done by hand again.

Save your image as a file and copy it to each VM:
```
cd ~/myapp
docker save myapp:latest | gzip > myapp.tar.gz

scp myapp.tar.gz vagrant@$UBUNTU_IP:~/
scp myapp.tar.gz vagrant@$FEDORA_IP:~/
scp myapp.tar.gz vagrant@$ALPINE_IP:~/
```

Load and run on each — notice the commands are now **identical** regardless of distro:

```
ssh vagrant@$UBUNTU_IP "docker load < myapp.tar.gz && docker run -d -p 8080:8080 myapp:latest"
ssh vagrant@$FEDORA_IP "docker load < myapp.tar.gz && docker run -d -p 8080:8080 myapp:latest"
ssh vagrant@$ALPINE_IP "docker load < myapp.tar.gz && docker run -d -p 8080:8080 myapp:latest"
```

Verify all three:
```
curl http://$UBUNTU_IP:8080/status
curl http://$FEDORA_IP:8080/status
curl http://$ALPINE_IP:8080/status
```

Same response. Same behaviour. The distro is irrelevant — the container brings its own environment.

Add `.dockerignore` to keep the image clean:
```
venv/
__pycache__/
*.pyc
.git/
```

Commit:
```
git add Dockerfile .dockerignore
git commit -m "containerize app"
```

> You wrote distro-specific install commands three times in chapter 3. Now you wrote `docker load && docker run` three times and it worked everywhere. That's the point.

---

## Chapter 5: Multi-Service Orchestration

Your app is stateless and simple. Let's change that. You'll add a request counter backed by Redis.

Update `app.py`:
```python
from flask import Flask, jsonify, make_response
import redis
import os

app = Flask(__name__)
r = redis.Redis(host=os.getenv("REDIS_HOST", "localhost"), port=6379)

@app.get("/status")
def status():
    count = r.incr("requests")
    resp = make_response(jsonify({"status": "ok", "requests": int(count)}))
    resp.headers["X-App-Name"] = "myapp"
    return resp

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

Add Redis to your dependencies:
```
pip install redis
pip freeze > requirements.txt
```

### Feel the problem

Try to run the app:
```
python app.py
curl http://localhost:8080/status
```

It crashes — Redis isn't running. Start Redis:
```
docker run -d -p 6379:6379 redis:7
```

Try again. It works. Now try to run the whole thing with Docker:
```
docker build -t myapp:latest .
docker run -p 8080:8080 myapp:latest
```

It can't reach Redis. Redis is running on your host, the container has its own network. You need to pass the right hostname:
```
docker run -p 8080:8080 -e REDIS_HOST=host.docker.internal myapp:latest
```

Maybe that works, maybe it doesn't — `host.docker.internal` is platform-specific. On Linux it may not resolve at all.

> You now have two services that need to know about each other, start in the right order, and share a network. You're managing this with a growing list of `docker run` flags you'll forget tomorrow.

Now imagine adding a database. A message queue. A cache. Environment variables multiplying. You have to remember to start everything before the app, in the right order, and tear it all down when you're done.

### Enter Docker Compose

Create `docker-compose.yml` in `myapp`:
```yaml
services:
  app:
    image: myapp:${TAG:-latest}
    ports:
      - "8080:8080"
    environment:
      REDIS_HOST: redis
    depends_on:
      - redis

  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

Note that the app service uses `image:` rather than `build: .`. This means Compose expects the image to already exist — you build it separately before running Compose. This distinction matters: on a VM or a colleague's machine you have the image but not the source code, so `build:` wouldn't work there anyway.

The `${TAG:-latest}` syntax tells Compose which image version to use. If the `TAG` environment variable is set, it uses that; otherwise it falls back to `latest`. You'll see this wired up properly in chapter 6 when we introduce image tagging.

Stop everything you started manually. Build the image, then bring up the stack:
```
docker build -t myapp:latest .
docker compose up
```

Both services start. The app can reach Redis at the hostname `redis` — Compose puts them on the same network automatically. `depends_on` ensures Redis starts first.

```
curl http://localhost:8080/status
curl http://localhost:8080/status
curl http://localhost:8080/status
```

The counter increments across requests.

Tear everything down:
```
docker compose down
```

Update `test_app.py` to mock Redis so unit tests don't require a running Redis:
```python
import pytest
from unittest.mock import patch, MagicMock
from app import app

@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as client:
        yield client

def test_status(client):
    mock_redis = MagicMock()
    mock_redis.incr.return_value = 1
    with patch("app.r", mock_redis):
        response = client.get("/status")
    assert response.status_code == 200
    assert response.json["status"] == "ok"
    assert response.json["requests"] == 1
```

Run the tests — they should pass without Redis running:
```
pytest
```

Commit:
```
git add docker-compose.yml app.py requirements.txt test_app.py
git commit -m "add redis counter, docker compose setup"
```

> The entire stack — app, database, networking — is declared in one file. Anyone can clone your repo and have a running system with one command.

---

## Chapter 6: Scripted Automation

You now have a workflow:

1. Run tests
2. Build the Docker image
3. Bring up the stack with Compose

You do this by hand. Sometimes you forget to run tests first. Sometimes you build the image and forget to restart the containers. Sometimes you ship broken code because you skipped step 1 "just this once."

> The problem isn't discipline. The problem is that manual processes have no memory and no enforcement.

### Feel the problem

Deliberately do it wrong a few times:

- Change the app, skip the tests, run `docker build -t myapp:latest . && docker compose up` — broken code is running
- Run the tests against the old image — they pass, but the running container has different code
- Forget to rebuild after changing `requirements.txt` — the image has stale dependencies

These are real mistakes that happen in real teams.

### Enter build.sh and deploy.sh

Two scripts, two responsibilities: `build.sh` validates and packages your code locally; `deploy.sh` ships it to the VMs. You run them independently or together.

`build.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

echo "--- Running tests ---"
pytest

echo "--- Building image ---"
docker build -t myapp:latest .

echo "--- Restarting local stack ---"
docker compose down
TAG=latest docker compose up -d

echo "--- Done. App running at http://localhost:8080/status ---"
```

`deploy.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

# NOTE: We're shipping the image by saving it to a file and copying it over
# with scp. This works for a local lab but is not how this is done in practice.
# In a real environment the image would be pushed to an image registry
# (Docker Hub, AWS ECR, GitHub Container Registry, etc.) and each server
# would pull it from there. Image registries are outside the scope of this
# tutorial.

INFRA_DIR="../infra"

echo "--- Reading VM IPs from Terraform state ---"
UBUNTU_IP=$(cd "$INFRA_DIR" && terraform output -raw ubuntu_ip)
FEDORA_IP=$(cd "$INFRA_DIR" && terraform output -raw fedora_ip)
ALPINE_IP=$(cd "$INFRA_DIR" && terraform output -raw alpine_ip)

echo "--- Saving image ---"
docker save myapp:latest | gzip > /tmp/myapp.tar.gz

for IP in "$UBUNTU_IP" "$FEDORA_IP" "$ALPINE_IP"; do
  echo "--- Deploying to $IP ---"
  scp /tmp/myapp.tar.gz docker-compose.yml vagrant@"$IP":~/
  ssh vagrant@"$IP" "docker load < myapp.tar.gz && TAG=latest docker compose down && TAG=latest docker compose up -d"
done

echo "--- Verifying ---"
curl -sf http://"$UBUNTU_IP":8080/status && echo " Ubuntu OK"
curl -sf http://"$FEDORA_IP":8080/status && echo " Fedora OK"
curl -sf http://"$ALPINE_IP":8080/status && echo " Alpine OK"

echo "--- Done ---"
```

Make both executable:
```
chmod +x build.sh deploy.sh
```

Run them:
```
./build.sh          # test + build + restart local stack
./deploy.sh         # ship to all three VMs
```

`set -euo pipefail` in both scripts means: if any step fails, stop. `build.sh` won't build a broken image. `deploy.sh` won't deploy a stale one if it exits early.

Test the enforcement: break `app.py` so tests fail, then run `./build.sh`. It stops at the test step — the image is never rebuilt, the local stack is never restarted. Fix the code, run again.

Commit:
```
git add build.sh deploy.sh
git commit -m "add build and deploy scripts"
```

### Feel the problem: what is actually running?

Make a small change to `app.py` — add a `"version"` field to the response. Run the scripts you just wrote:
```
./build.sh && ./deploy.sh
```

It works. Make another change — rename the field to `"v"`. Run them again:
```
./build.sh && ./deploy.sh
```

SSH into one of the VMs and check what's running:
```
ssh vagrant@$UBUNTU_IP "docker ps"
```

You'll see something like:
```
IMAGE          CREATED
myapp:latest   2 minutes ago
```

Questions you cannot answer:
- Which commit is in that image?
- What was deployed before this?
- If this version is broken, what do you roll back to?

`latest` is a moving target. It tells you nothing about what code is inside. You can't roll back to `latest` from last Tuesday — that image is gone, overwritten by every subsequent build.

### Enter image tagging

The fix is to tag every image with the git commit hash that produced it. Short, unique, permanently traceable to the source.

Update `build.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

TAG=$(git rev-parse --short HEAD)

echo "--- Running tests ---"
pytest

echo "--- Building image (tag: $TAG) ---"
docker build -t myapp:"$TAG" -t myapp:latest .

echo "--- Restarting local stack ---"
docker compose down
TAG="$TAG" docker compose up -d

echo "--- Built and deployed myapp:$TAG ---"
echo "--- App running at http://localhost:8080/status ---"
```

Update `deploy.sh` to require a tag as an argument:
```bash
#!/usr/bin/env bash
set -euo pipefail

# NOTE: We're shipping the image by saving it to a file and copying it over
# with scp. This works for a local lab but is not how this is done in practice.
# In a real environment the image would be pushed to an image registry
# (Docker Hub, AWS ECR, GitHub Container Registry, etc.) and each server
# would pull it from there. Image registries are outside the scope of this
# tutorial.

TAG=${1:?Usage: ./deploy.sh <image-tag>  (e.g. ./deploy.sh \$(git rev-parse --short HEAD))}
INFRA_DIR="../infra"

echo "--- Deploying myapp:$TAG ---"

echo "--- Reading VM IPs from Terraform state ---"
UBUNTU_IP=$(cd "$INFRA_DIR" && terraform output -raw ubuntu_ip)
FEDORA_IP=$(cd "$INFRA_DIR" && terraform output -raw fedora_ip)
ALPINE_IP=$(cd "$INFRA_DIR" && terraform output -raw alpine_ip)

echo "--- Saving image ---"
docker save myapp:"$TAG" | gzip > /tmp/myapp.tar.gz

for IP in "$UBUNTU_IP" "$FEDORA_IP" "$ALPINE_IP"; do
  echo "--- Deploying to $IP ---"
  scp /tmp/myapp.tar.gz docker-compose.yml vagrant@"$IP":~/
  ssh vagrant@"$IP" "docker load < myapp.tar.gz && TAG=$TAG docker compose down && TAG=$TAG docker compose up -d"
done

echo "--- Verifying ---"
curl -sf http://"$UBUNTU_IP":8080/status && echo " Ubuntu OK"
curl -sf http://"$FEDORA_IP":8080/status && echo " Fedora OK"
curl -sf http://"$ALPINE_IP":8080/status && echo " Alpine OK"

echo "--- Done. Deployed myapp:$TAG to all VMs ---"
```

Run the full flow:
```
./build.sh
./deploy.sh $(git rev-parse --short HEAD)
```

Or in one line:
```
./build.sh && ./deploy.sh $(git rev-parse --short HEAD)
```

Now check the VMs:
```
ssh vagrant@$UBUNTU_IP "docker ps"
```

You'll see `myapp:abc1234` — an exact, traceable identifier. To roll back to any previous version:
```
./deploy.sh <previous-commit-hash>
```

As long as you built that version at some point, the image exists locally and can be redeployed. Every deployment is an explicit, deliberate choice.

Commit the updated scripts:
```
git add build.sh deploy.sh
git commit -m "tag images with git commit hash"
```

> This is the core idea behind every CI/CD system you'll encounter — Jenkins, GitHub Actions, GitLab CI. They run the same sequence of steps in the same order, every time, with no human in the loop. These two scripts are that, without the external tooling.

---

## What's Next

You've now experienced and solved six fundamental problems:

| Problem | Solution |
|---------|----------|
| Can't recover broken code | Git |
| Works on my machine | venv + requirements.txt |
| Manual VM provisioning doesn't scale | Terraform |
| App behaves differently per OS | Docker |
| Multi-service startup is chaos | Docker Compose |
| Manual deploy steps get skipped | build.sh + deploy.sh |

These concepts carry directly into every cloud environment and every team setup you'll encounter. The tools change — Kubernetes instead of Compose, GitHub Actions instead of build.sh, Terraform with AWS instead of VirtualBox — but the problems they solve are identical to what you've just felt.
