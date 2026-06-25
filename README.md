{{- define "goldengate.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "goldengate.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "goldengate.labels" -}}
app.kubernetes.io/name: {{ include "goldengate.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
app.kubernetes.io/part-of: goldengate
goldengate.oracle.com/environment: {{ .Values.global.environment | quote }}
goldengate.oracle.com/deployment-id: {{ .Values.global.deploymentId | quote }}
{{- end }}

{{- define "goldengate.selectorLabels" -}}
app.kubernetes.io/name: {{ include "goldengate.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "goldengate.sourceName" -}}
{{- printf "%s-source" (include "goldengate.fullname" .) | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "goldengate.targetName" -}}
{{- printf "%s-target" (include "goldengate.fullname" .) | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "goldengate.sourceSecretName" -}}
{{- printf "%s-secret" (include "goldengate.sourceName" .) | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "goldengate.targetSecretName" -}}
{{- printf "%s-secret" (include "goldengate.targetName" .) | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "goldengate.sourceServiceAccountName" -}}
{{- if .Values.source.serviceAccount.create }}
{{- default (include "goldengate.sourceName" .) .Values.source.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.source.serviceAccount.name }}
{{- end }}
{{- end }}

{{- define "goldengate.targetServiceAccountName" -}}
{{- if .Values.target.serviceAccount.create }}
{{- default (include "goldengate.targetName" .) .Values.target.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.target.serviceAccount.name }}
{{- end }}
{{- end }}
