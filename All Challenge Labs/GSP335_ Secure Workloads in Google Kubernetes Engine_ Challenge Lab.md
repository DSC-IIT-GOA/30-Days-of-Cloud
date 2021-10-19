### GSP335 : Secure Workloads in Google Kubernetes Engine: Challenge Lab :-

----------------------------------------------------------------------------------------------------------------------------------------------

Follow on Twitter @techiezilla , Instagram @techiezilla.


### Task - 0 : Download the necessary files :-

```yaml
gsutil -m cp gs://cloud-training/gsp335/* .
```

### Task - 1 : Setup Cluster :-

```yaml
gcloud container clusters create demo1  --machine-type n1-standard-4 --num-nodes 2 --zone us-central1-c --enable-network-policy
gcloud container clusters get-credentials demo1 --zone us-central1-c
```

### Task - 2 : Setup WordPress :-

```yaml
gcloud sql instances create wordpress --region=us-central1
gcloud sql databases create wordpress --instance wordpress
gcloud sql users create dbpress --instance=wordpress --host=% --password='P@ssword!'
gcloud sql users create wordpress1 --instance=wordpress --host=% --password='P@ssword!'
```

```yaml
gcloud iam service-accounts create sa-wordpress --display-name sa-wordpress
```

```yaml
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT    --role roles/cloudsql.client  --member serviceAccount:sa-wordpress@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

```yaml
gcloud iam service-accounts keys create key.json    --iam-account sa-wordpress@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

```yaml
kubectl create secret generic cloudsql-instance-credentials    --from-file key.json
kubectl create secret generic cloudsql-db-credentials \
  --from-literal username=wordpress \
  --from-literal password='P@ssword!'
```

```yaml
kubectl apply -f volume.yaml
```

```yaml
sed -i s/INSTANCE_CONNECTION_NAME/${GOOGLE_CLOUD_PROJECT}:us-central1:wordpress/g wordpress.yaml
kubectl apply -f wordpress.yaml
```

### Task - 3 : Setup Ingress with TLS :-

```yaml
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install nginx-ingress stable/nginx-ingress
```

```yaml
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.0/cert-manager.yaml
```

```yaml
kubectl get svc
```

Check the service nginx-ingress-controller as an external IP address before continuing to the next step.

```yaml
sed -i s/LAB_EMAIL_ADDRESS/sa-wordpress@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com/g issuer.yaml
kubectl apply -f issuer.yaml
```

If you face any internal error, execute the command again.

```yaml
. add_ip.sh
```

```yaml
HOST_NAME=$(echo $USER | tr -d '_').labdns.xyz
sed -i s/HOST_NAME/${HOST_NAME}/g ingress.yaml
kubectl apply -f ingress.yaml
```

### Task - 4 : Set up Network Policy :-

```yaml
kubectl apply -f network-policy.yaml
```

```yaml
nano network-policy.yaml
```

Add the following code at the last :-

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: allow-nginx-access-to-internet
spec:
 podSelector:
  matchLabels:
    app: nginx-ingress
 policyTypes:
 - Ingress
 ingress:
 - {}
```

### Task - 5 : Setup Binary Authorization :-

```yaml
gcloud services enable \
  container.googleapis.com \
  containeranalysis.googleapis.com \
  binaryauthorization.googleapis.com
```

```yaml
gcloud container clusters update demo1 --enable-binauthz --zone us-central1-c
```

```yaml
gcloud container binauthz policy export > bin-auth-policy.yaml
```

```yaml
nano bin-auth-policy.yaml
```

Edit and add the four values and change as :-

```yaml
- namePattern: docker.io/library/wordpress:latest
- namePattern: us.gcr.io/k8s-artifacts-prod/ingress-nginx/*
- namePattern: gcr.io/cloudsql-docker/*
- namePattern: quay.io/jetstack/*
defaultAdmissionRule:
 enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
 evaluationMode: ALWAYS_DENY
globalPolicyEvaluationMode: ENABLE
```

```yaml
gcloud container binauthz policy import bin-auth-policy.yaml
```

### Task - 6 : Setup Pod Security Policy :-

```yaml
kubectl apply -f psp-restrictive.yaml
kubectl apply -f psp-role.yaml
kubectl apply -f psp-use.yaml
```

### Congratulations!

Follow on Twitter @techiezilla , Instagram @techiezilla.
