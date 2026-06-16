A working example of the sidecar pattern in Kubernetes using nginx and git-sync.

What is the Sidecar Pattern?

A sidecar is a helper container that runs alongside the main application container in the same pod. They share the same network namespace and can share volumes via emptyDir.

The main app doesn't need to know the sidecar exists — they just share a filesystem.


What We Built


nginx — serves web content from a shared volume
git-sync — clones a git repo and keeps it in sync every 60 seconds


GitHub repo
      ↓ git-sync clones + pulls every 60s
emptyDir volume (shared between containers)
      ↓ nginx serves files
Browser


How git-sync Works

git-sync uses worktrees for atomic updates. Instead of overwriting files in place (risky — nginx might read half-written files), it:


Prepares the new commit in a fresh directory .worktrees/newcommit/
Atomically flips a symlink: current → newcommit
nginx reads from current which always points to the latest complete version


This means nginx never sees a partially updated state.


Problems We Hit

Problem 1 — 403 Forbidden (permissions)

git-sync writes files as user 65533. nginx runs as a different user and couldn't read them.

Fix — fsGroup in pod securityContext:

yamlsecurityContext:
  fsGroup: 101

Makes the shared volume group-owned by GID 101, which is added to all containers in the pod.

Problem 2 — 403 Forbidden (wrong path)

nginx default config points to /usr/share/nginx/html but files land at /usr/share/nginx/html/current. Also no index.html exists in a Go repo.

Fix — ConfigMap to override nginx config:

yamlserver {
  listen 80;
  location / {
    root /usr/share/nginx/html/current;
    autoindex on;
  }
}

autoindex on shows a directory listing when there's no index.html.


Why Not Just Use Apache (httpd)?

The official git-sync Kubernetes docs use httpd:alpine for simplicity. They acknowledge it themselves:

"NOTE: apache runs as root to expose port 80, and there's not a trivial flag to change that. Real servers should not run as root when possible."

Taking their own advice, we used nginx with fsGroup and a ConfigMap instead — which requires a bit more config but follows the non-root recommendation for real servers.


Final YAML

yamlapiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
      listen 80;
      location / {
        root /usr/share/nginx/html/current;
        autoindex on;
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-gitsync
  labels:
    app: nginx-gitsync
spec:
  securityContext:
    fsGroup: 101
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-content
          mountPath: /usr/share/nginx/html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf

    - name: git-sync
      image: registry.k8s.io/git-sync/git-sync:v4.0.0
      args:
        - --repo=https://github.com/kubernetes/git-sync
        - --depth=1
        - --period=60s
        - --link=current
        - --root=/git
      volumeMounts:
        - name: shared-content
          mountPath: /git

  volumes:
    - name: shared-content
      emptyDir: {}
    - name: nginx-config
      configMap:
        name: nginx-config


Expose and Access

bashkubectl apply -f sidecar.yaml
kubectl expose pod nginx-gitsync --port=80 --type=NodePort
minikube service nginx-gitsync --url


Key Takeaways


emptyDir is the bridge between sidecar containers — same volume, different mount paths
git-sync uses worktrees + symlinks for atomic updates — nginx never sees incomplete state
fsGroup solves cross-container permission issues on shared volumes
ConfigMap overrides nginx config without rebuilding the image
The official docs use Apache for simplicity and suggest avoiding root — we followed that advice using nginx + fsGroup
