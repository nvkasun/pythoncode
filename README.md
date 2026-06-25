{{- if .Values.target.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "goldengate.targetName" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "goldengate.labels" . | nindent 4 }}
    app.kubernetes.io/component: target
spec:
  type: {{ .Values.target.service.type }}
  selector:
    {{- include "goldengate.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: target
  ports:
    - name: http
      port: {{ .Values.target.ports.http }}
      targetPort: http
      protocol: TCP
    - name: https
      port: {{ .Values.target.ports.https }}
      targetPort: https
      protocol: TCP
    - name: receiver
      port: {{ .Values.target.ports.receiver }}
      targetPort: receiver
      protocol: TCP
    - name: metrics
      port: {{ .Values.target.ports.metrics }}
      targetPort: metrics
      protocol: TCP
{{- end }}
