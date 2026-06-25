global:
  environment: dev
  deploymentId: ""

imagePullSecrets: []

source:
  enabled: true
  dbType: oracle

  image:
    repository: 229410149234.dkr.ecr.eu-west-1.amazonaws.com/ogg-oracle
    tag: "23.26.2.0.2"
    pullPolicy: IfNotPresent

  ogg:
    deploymentName: OracleSource
    domain: source.goldengate.local
    adminUser: ggadmin
    adminPassword: change-me

  serviceAccount:
    create: true
    name: ""
    annotations: {}

  ports:
    http: 8080
    https: 8443
    distribution: 9013
    metrics: 9015

  service:
    type: ClusterIP

  persistence:
    enabled: false
    existingClaim: ""
    createPV: false
    createPVC: false
    size: 20Gi
    storageClassName: ""
    accessModes:
      - ReadWriteMany
    nfs:
      server: ""
      path: ""

  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "2"
      memory: 4Gi

  extraEnv: []
  extraVolumeMounts: []
  extraContainers: []

target:
  enabled: true
  dbType: postgresql

  image:
    repository: 229410149234.dkr.ecr.eu-west-1.amazonaws.com/ogg-postgresql
    tag: "23.26.2.0.1"
    pullPolicy: IfNotPresent

  ogg:
    deploymentName: PostgresTarget
    domain: target.goldengate.local
    adminUser: ggadmin
    adminPassword: change-me

  serviceAccount:
    create: true
    name: ""
    annotations: {}

  ports:
    http: 8080
    https: 8443
    receiver: 9014
    metrics: 9015

  service:
    type: ClusterIP

  persistence:
    enabled: false
    existingClaim: ""
    createPV: false
    createPVC: false
    size: 20Gi
    storageClassName: ""
    accessModes:
      - ReadWriteMany
    nfs:
      server: ""
      path: ""

  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: "2"
      memory: 4Gi

  extraEnv: []
  extraVolumeMounts: []
  extraContainers: []

podSecurityContext: {}

securityContext: {}

nodeSelector: {}

tolerations: []

affinity: {}
