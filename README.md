global:
  environment: ""
  deploymentId: ""

imagePullSecrets: []

source:
  enabled: false
  dbType: ""

  image:
    repository: ""
    tag: ""
    pullPolicy: IfNotPresent

  ogg:
    deploymentName: ""
    domain: ""
    adminUser: ""
    adminPassword: ""

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

  resources: {}

  extraEnv: []
  extraVolumeMounts: []
  extraVolumes: []
  extraContainers: []

target:
  enabled: false
  dbType: ""

  image:
    repository: ""
    tag: ""
    pullPolicy: IfNotPresent

  ogg:
    deploymentName: ""
    domain: ""
    adminUser: ""
    adminPassword: ""

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

  resources: {}

  extraEnv: []
  extraVolumeMounts: []
  extraVolumes: []
  extraContainers: []

podSecurityContext: {}

securityContext: {}

nodeSelector: {}

tolerations: []

affinity: {}
