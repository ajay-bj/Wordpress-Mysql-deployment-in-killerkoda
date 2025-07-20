# Wordpress-Mysql-deployment-in-killerkoda
Wordpress-Mysql-deployment-in-killerkoda




create two files 1. mysql-db.yaml (for db namespace) &  2. wordpress-app.yaml (for wordpress namespace)

vim mysql-db.yaml

kubectl apply -f mysql-db.yaml

kubectl get pods -n db


vim wordpress-app.yaml

kubectl apply -f wordpress-app.yaml

kubectl get pods -n wordpress

k get svc -A





to test myswl database and seeing tables inside

kubectl get pods -n db
k exec -it mysql-68b5fcd6b-qmtrl -n db -- bash

mysql -u wpuser -p
password=wppassword

Check Databases sqlcommands
SHOW DATABASES;
USE wordpress;
SHOW TABLES;








# WordPress + MySQL on KillerCoda (Kubernetes Lab)

> **Goal:** Deploy a working **WordPress site backed by a MySQL database** across **two Kubernetes namespaces** (`db` and `wordpress`) in a **KillerCoda lab cluster**. Perfect for classroom demos, workshops, and beginner Kubernetes practice!

---

## 🎯 What You'll Learn

* Creating and using **namespaces** in Kubernetes.
* Deploying **stateful apps** (MySQL) with PersistentVolumeClaims.
* Deploying **WordPress frontend** that connects across namespaces to the database.
* Exposing apps with a **NodePort Service** so students can access the UI in the browser.
* Connecting to MySQL using the **CLI inside the cluster**.

---

## 🗂️ Repo Structure (recommended)

```
Wordpress-Mysql-deployment-in-killerkoda/
├─ mysql-db.yaml          # MySQL in db namespace
├─ wordpress-app.yaml     # WordPress in wordpress namespace
├─ extras/
│  ├─ phpmyadmin.yaml     # (optional) phpMyAdmin UI
│  └─ cleanup.sh          # (optional) delete all resources
└─ README.md              # this guide
```

---

## 🚀 Quick Start (Copy-Paste Friendly)

```bash
# 1. Deploy MySQL (db namespace)
kubectl apply -f mysql-db.yaml
kubectl get pods -n db

# 2. Deploy WordPress (wordpress namespace)
kubectl apply -f wordpress-app.yaml
kubectl get pods -n wordpress

# 3. Check services (note NodePort)
kubectl get svc -A

# 4. Open WordPress in browser
#    Replace localhost with node IP if needed
curl http://localhost:30081
```

---

## 🧱 YAML 1: MySQL in `db` Namespace (`mysql-db.yaml`)

<details>
<summary><strong>Click to expand mysql-db.yaml</strong></summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: db
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: db
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: db
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootpassword
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wpuser
        - name: MYSQL_PASSWORD
          value: wppassword
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: db
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
```

</details>

---

## 🌐 YAML 2: WordPress in `wordpress` Namespace (`wordpress-app.yaml`)

<details>
<summary><strong>Click to expand wordpress-app.yaml</strong></summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - image: wordpress:latest
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql.db.svc.cluster.local
        - name: WORDPRESS_DB_USER
          value: wpuser
        - name: WORDPRESS_DB_PASSWORD
          value: wppassword
        - name: WORDPRESS_DB_NAME
          value: wordpress
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30081
  selector:
    app: wordpress
```

</details>

---

## 🔌 Cross-Namespace DNS Explained

Kubernetes gives every Service a cluster DNS name:

```
<SERVICE>.<NAMESPACE>.svc.cluster.local
```

So because **MySQL Service = mysql** and **namespace = db**, the **DB host** you give WordPress is:

```
mysql.db.svc.cluster.local
```

This is set in the WordPress Deployment env var `WORDPRESS_DB_HOST`.

---

## 📡 Exposing WordPress to Students

We used a **NodePort** (30081) so each student can open the site in their browser.

**Access URL examples:**

* From KillerCoda shell (curl):

  ```bash
  curl http://localhost:30081
  ```
* From your own machine (if Node IP exposed):

  ```
  http://<node-public-ip>:30081
  ```

---

## 🛠️ Verify Deployments

### MySQL

```bash
kubectl get pods -n db
kubectl logs -n db deploy/mysql
```

### WordPress

```bash
kubectl get pods -n wordpress
kubectl logs -n wordpress deploy/wordpress
```

---

## 🧪 Connect to MySQL (CLI)

### 1. Get the Pod name

```bash
kubectl get pods -n db
```

### 2. Exec into the MySQL Pod

```bash
kubectl exec -it <mysql-pod-name> -n db -- bash
```

### 3. Use MySQL client

```bash
mysql -u wpuser -p
# password: wppassword
```

OR use root:

```bash
mysql -u root -p
# password: rootpassword
```

### 4. Basic SQL Checks

```sql
SHOW DATABASES;
USE wordpress;
SHOW TABLES;
```

---

## 💡 Bonus: Connect From a Client Pod (No Exec Into DB)

If you prefer to keep DB pod clean, launch a temporary client:

```bash
kubectl run mysql-client --rm -it \
  --image=mysql:5.7 \
  --restart=Never \
  -- bash
```

Then inside that container:

```bash
mysql -h mysql.db.svc.cluster.local -u wpuser -p
# password: wppassword
```

---

## 📘 Classroom Flow (Suggested Teaching Script)

1. **Intro to Namespaces** – why separate app vs DB.
2. **Apply MySQL YAML** – show `kubectl get ns`, `kubectl get pods -n db`.
3. **Explain PV/PVC** – persistent state.
4. **Apply WordPress YAML** – highlight `WORDPRESS_DB_HOST` cross-namespace.
5. **Access UI** – students open browser; complete WP setup wizard.
6. **Query DB** – show data tables after WordPress install.
7. **Q\&A + Troubleshooting** – show how to check logs.

---

## 🧾 Environment Variables Recap

| Component | Variable                | Value                        | Purpose                     |
| --------- | ----------------------- | ---------------------------- | --------------------------- |
| MySQL     | `MYSQL_ROOT_PASSWORD`   | `rootpassword`               | Root DB access (lab only!)  |
| MySQL     | `MYSQL_DATABASE`        | `wordpress`                  | DB auto-created at startup  |
| MySQL     | `MYSQL_USER`            | `wpuser`                     | App user account            |
| MySQL     | `MYSQL_PASSWORD`        | `wppassword`                 | Password for `wpuser`       |
| WordPress | `WORDPRESS_DB_HOST`     | `mysql.db.svc.cluster.local` | Cross-namespace DB hostname |
| WordPress | `WORDPRESS_DB_USER`     | `wpuser`                     | App DB user                 |
| WordPress | `WORDPRESS_DB_PASSWORD` | `wppassword`                 | DB password                 |
| WordPress | `WORDPRESS_DB_NAME`     | `wordpress`                  | DB name                     |

> ⚠️ **Security Note:** Use these credentials *only* in lab/demo environments.

---

## 🧹 Cleanup (End of Lab)

```bash
kubectl delete -f wordpress-app.yaml
kubectl delete -f mysql-db.yaml
# Optionally delete namespaces if you want full cleanup:
kubectl delete ns wordpress
tkubectl delete ns db
```

---

## 🧯 Troubleshooting Guide

| Symptom                                              | Likely Cause                           | Fix                                                                   |
| ---------------------------------------------------- | -------------------------------------- | --------------------------------------------------------------------- |
| Pod `ImagePullBackOff`                               | Node can’t pull image (network / typo) | Check image name, try `kubectl describe pod`, verify internet access. |
| WordPress `Error establishing a database connection` | Wrong `WORDPRESS_DB_HOST` or creds     | Confirm env vars; `kubectl exec -n wordpress` into pod; ping DB host. |
| MySQL CrashLoop                                      | Bad env vars / PVC permission          | Ensure password + DB env vars; delete & recreate PVC (lab).           |
| WordPress page loads but setup fails                 | DB not ready yet                       | Wait for MySQL pod `Running` before applying WordPress.               |

---

## 🧭 Optional: Add phpMyAdmin (GUI for MySQL)

Create `extras/phpmyadmin.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
  namespace: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
      - name: phpmyadmin
        image: phpmyadmin:apache
        env:
        - name: PMA_HOST
          value: mysql
        - name: PMA_USER
          value: wpuser
        - name: PMA_PASSWORD
          value: wppassword
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin
  namespace: db
spec:
  type: NodePort
  selector:
    app: phpmyadmin
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30082
```

Apply it:

```bash
kubectl apply -f extras/phpmyadmin.yaml
```

Access:

```
http://localhost:30082
```

Log in with `wpuser` / `wppassword` (or root creds if needed).

---

## 🧑‍🏫 Teaching Tips

* Have students work in **pairs**: one deploys DB, the other deploys WordPress.
* Ask: *"Why use namespaces?"* Collect answers: isolation, RBAC, quotas.
* After install, show how blog posts in WordPress create tables & rows in MySQL.
* Challenge: Change NodePort to LoadBalancer (if cloud), or switch to Ingress.

---

## 📌 Lab Checklist (printable for students)

* [ ] Applied `mysql-db.yaml`
* [ ] MySQL pod Running (`kubectl get pods -n db`)
* [ ] Applied `wordpress-app.yaml`
* [ ] WordPress pod Running (`kubectl get pods -n wordpress`)
* [ ] Opened `http://localhost:30081`
* [ ] Completed WordPress setup wizard
* [ ] Connected to MySQL via CLI
* [ ] Ran `SHOW TABLES;` and saw WordPress tables

---

## 📣 Contributing

PRs welcome! Ideas:

* Helm chart version
* StatefulSet upgrade for MySQL
* StorageClass demo
* Ingress + TLS example

---

## 📝 License

Choose a license that fits your teaching goals (MIT is common for labs). Add a `LICENSE` file in the repo root.

---

**Happy teaching, Ajay!** Let me know if you want a version with screenshots, animated GIFs, or cloud‑ready manifests (EKS, GKE, AKS).
