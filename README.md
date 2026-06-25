{{- if .Values.target.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "goldengate.targetName" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "goldengate.labels" . | nindent 4 }}
    app.kubernetes.io/component: target
spec:
  serviceName: {{ include "goldengate.targetName" . }}
  replicas: 1
  selector:
    matchLabels:
      {{- include "goldengate.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: target
  template:
    metadata:
      labels:
        {{- include "goldengate.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: target
        goldengate.oracle.com/db-type: {{ required "target.dbType is required when target.enabled=true" .Values.target.dbType | quote }}
    spec:
      serviceAccountName: {{ include "goldengate.targetServiceAccountName" . }}

      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}

      containers:
        - name: goldengate
          image: "{{ required "target.image.repository is required when target.enabled=true" .Values.target.image.repository }}:{{ required "target.image.tag is required when target.enabled=true" .Values.target.image.tag }}"
          imagePullPolicy: {{ .Values.target.image.pullPolicy }}

          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}

          envFrom:
            - secretRef:
                name: {{ include "goldengate.targetSecretName" . }}

          {{- with .Values.target.extraEnv }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          ports:
            - name: http
              containerPort: {{ .Values.target.ports.http }}
              protocol: TCP
            - name: https
              containerPort: {{ .Values.target.ports.https }}
              protocol: TCP
            - name: receiver
              containerPort: {{ .Values.target.ports.receiver }}
              protocol: TCP
            - name: metrics
              containerPort: {{ .Values.target.ports.metrics }}
              protocol: TCP

          readinessProbe:
            tcpSocket:
              port: https
            initialDelaySeconds: 60
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 10

          livenessProbe:
            tcpSocket:
              port: https
            initialDelaySeconds: 120
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 10

          volumeMounts:
            - name: u02
              mountPath: /u02
            - name: u03
              mountPath: /u03
            {{- with .Values.target.extraVolumeMounts }}
            {{- toYaml . | nindent 12 }}
            {{- end }}

          resources:
            {{- toYaml .Values.target.resources | nindent 12 }}

        {{- with .Values.target.extraContainers }}
        {{- toYaml . | nindent 8 }}
        {{- end }}

      volumes:
        - name: u02
          {{- if and .Values.target.persistence.enabled .Values.target.persistence.existingClaim }}
          persistentVolumeClaim:
            claimName: {{ .Values.target.persistence.existingClaim }}
          {{- else if and .Values.target.persistence.enabled .Values.target.persistence.createPVC }}
          persistentVolumeClaim:
            claimName: {{ include "goldengate.targetName" . }}-pvc
          {{- else }}
          emptyDir: {}
          {{- end }}

        - name: u03
          emptyDir: {}

        {{- with .Values.target.extraVolumes }}
        {{- toYaml . | nindent 8 }}
        {{- end }}

      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- end }}
