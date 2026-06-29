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

  volumes:
    - name: u02
      emptyDir: {}
    - name: u03
      emptyDir: {}
