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

