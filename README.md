{{- if .Values.source.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "goldengate.sourceSecretName" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "goldengate.labels" . | nindent 4 }}
    app.kubernetes.io/component: source
type: Opaque
stringData:
  OGG_ADMIN: {{ required "source.ogg.adminUser is required when source.enabled=true" .Values.source.ogg.adminUser | quote }}
  OGG_ADMIN_PWD: {{ required "source.ogg.adminPassword is required when source.enabled=true" .Values.source.ogg.adminPassword | quote }}
  OGG_DEPLOYMENT: {{ required "source.ogg.deploymentName is required when source.enabled=true" .Values.source.ogg.deploymentName | quote }}
  OGG_DOMAIN: {{ required "source.ogg.domain is required when source.enabled=true" .Values.source.ogg.domain | quote }}
  DEPLOYMENT_ROLE: "source"
  DATABASE_TYPE: {{ required "source.dbType is required when source.enabled=true" .Values.source.dbType | quote }}
{{- end }}
