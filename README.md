{{- if and .Values.target.enabled .Values.target.serviceAccount.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "goldengate.targetServiceAccountName" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "goldengate.labels" . | nindent 4 }}
    app.kubernetes.io/component: target
  {{- with .Values.target.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
