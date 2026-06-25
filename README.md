{{- if .Values.target.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "goldengate.targetSecretName" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "goldengate.labels" . | nindent 4 }}
    app.kubernetes.io/component: target
type: Opaque
stringData:
  OGG_ADMIN: {{ required "target.ogg.adminUser is required when target.enabled=true" .Values.target.ogg.adminUser | quote }}
  OGG_ADMIN_PWD: {{ required "target.ogg.adminPassword is required when target.enabled=true" .Values.target.ogg.adminPassword | quote }}
  OGG_DEPLOYMENT: {{ required "target.ogg.deploymentName is required when target.enabled=true" .Values.target.ogg.deploymentName | quote }}
  OGG_DOMAIN: {{ required "target.ogg.domain is required when target.enabled=true" .Values.target.ogg.domain | quote }}
  DEPLOYMENT_ROLE: "target"
  DATABASE_TYPE: {{ required "target.dbType is required when target.enabled=true" .Values.target.dbType | quote }}
{{- end }}
