{{- if .Values.source.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "goldengate.sourceName" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "goldengate.labels" . | nindent 4 }}
    app.kubernetes.io/component: source
spec:
  type: {{ .Values.source.service.type }}
  selector:
    {{- include "goldengate.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: source
  ports:
    - name: http
      port: {{ .Values.source.ports.http }}
      targetPort: http
      protocol: TCP
    - name: https
      port: {{ .Values.source.ports.https }}
      targetPort: https
      protocol: TCP
    - name: distribution
      port: {{ .Values.source.ports.distribution }}
      targetPort: distribution
      protocol: TCP
    - name: metrics
      port: {{ .Values.source.ports.metrics }}
      targetPort: metrics
      protocol: TCP
{{- end }}
