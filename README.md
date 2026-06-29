First confirm the certificate folder inside the image

Run this debug pod. This will not start GoldenGate; it will just keep the container alive so we can inspect the filesystem.

kubectl delete pod ogg-oracle-debug -n ogg --ignore-not-found

kubectl run ogg-oracle-debug \
  -n ogg \
  --image=229410149234.dkr.ecr.eu-west-1.amazonaws.com/ogg-oracle:23.26.2.0.1 \
  --restart=Never \
  --command -- sh -c "sleep 3600"

Then inspect:

kubectl exec -it ogg-oracle-debug -n ogg -- sh

Inside the container run:

ls -la /u01/oggf
ls -la /u01/oggf/certificate || true
find /u01/oggf -maxdepth 4 -type f | grep -i pem || true
find /u02 -maxdepth 4 -type f | grep -i pem || true

This will confirm whether /u01/oggf/certificate/ca-key.pem exists in the image or not.

Quick fix for testing

For testing, create a CA key and certificate, then mount it into the pod at the path GoldenGate is asking for.

Run this from your shell:

openssl genrsa -out ca-key.pem 4096

openssl req -x509 -new -nodes \
  -key ca-key.pem \
  -sha256 \
  -days 3650 \
  -out ca-cert.pem \
  -subj "/CN=ogg-test-ca"

kubectl delete secret ogg-ca-cert -n ogg --ignore-not-found

kubectl create secret generic ogg-ca-cert \
  -n ogg \
  --from-file=ca-key.pem=ca-key.pem \
  --from-file=ca-cert.pem=ca-cert.pem

Then update your test pod like this:

apiVersion: v1
kind: Pod
metadata:
  name: ogg-oracle-run-test
  namespace: ogg
spec:
  restartPolicy: Always

  containers:
    - name: ogg-oracle
      image: 229410149234.dkr.ecr.eu-west-1.amazonaws.com/ogg-oracle:23.26.2.0.1
      imagePullPolicy: Always

      env:
        - name: OGG_ADMIN
          value: "oggadmin"
        - name: OGG_ADMIN_PWD
          value: "Welcome12345"
        - name: OGG_DEPLOYMENT
          value: "OracleTest"
        - name: OGG_DOMAIN
          value: "ogg-oracle-run-test.ogg.svc.cluster.local"
        - name: OGG_SECURE_DEPLOYMENT
          value: "false"

      ports:
        - name: http
          containerPort: 8080
        - name: https
          containerPort: 8443

      volumeMounts:
        - name: u02
          mountPath: /u02
        - name: u03
          mountPath: /u03
        - name: ogg-ca-cert
          mountPath: /u01/oggf/certificate
          readOnly: true

  volumes:
    - name: u02
      emptyDir: {}
    - name: u03
      emptyDir: {}
    - name: ogg-ca-cert
      secret:
        secretName: ogg-ca-cert

Apply it:

kubectl delete pod ogg-oracle-run-test -n ogg --ignore-not-found
kubectl apply -f ogg-oracle-run-test.yaml
kubectl get pods -n ogg -w

Then check logs:

kubectl logs -f ogg-oracle-run-test -n ogg -c ogg-oracle

If it becomes Running, then check ports:

kubectl port-forward pod/ogg-oracle-run-test -n ogg 8443:8443

Open:

https://localhost:8443

Login:

username: oggadmin
password: Welcome12345
