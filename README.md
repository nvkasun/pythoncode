{{- if .Values.source.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "goldengate.sourceName" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "goldengate.labels" . | nindent 4 }}
    app.kubernetes.io/component: source
spec:
  serviceName: {{ include "goldengate.sourceName" . }}
  replicas: 1
  selector:
    matchLabels:
      {{- include "goldengate.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: source
  template:
    metadata:
      labels:
        {{- include "goldengate.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: source
        goldengate.oracle.com/db-type: {{ required "source.dbType is required when source.enabled=true" .Values.source.dbType | quote }}
    spec:
      serviceAccountName: {{ include "goldengate.sourceServiceAccountName" . }}

      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}

      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}

      containers:
        - name: goldengate
          image: "{{ required "source.image.repository is required when source.enabled=true" .Values.source.image.repository }}:{{ required "source.image.tag is required when source.enabled=true" .Values.source.image.tag }}"
          imagePullPolicy: {{ .Values.source.image.pullPolicy }}

          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}

          envFrom:
            - secretRef:
                name: {{ include "goldengate.sourceSecretName" . }}

          {{- with .Values.source.extraEnv }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}

          ports:
            - name: http
              containerPort: {{ .Values.source.ports.http }}
              protocol: TCP
            - name: https
              containerPort: {{ .Values.source.ports.https }}
              protocol: TCP
            - name: distribution
              containerPort: {{ .Values.source.ports.distribution }}
              protocol: TCP
            - name: metrics
              containerPort: {{ .Values.source.ports.metrics }}
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
            {{- with .Values.source.extraVolumeMounts }}
            {{- toYaml . | nindent 12 }}
            {{- end }}

          resources:
            {{- toYaml .Values.source.resources | nindent 12 }}

        {{- with .Values.source.extraContainers }}
        {{- toYaml . | nindent 8 }}
        {{- end }}

      volumes:
        - name: u02
          {{- if and .Values.source.persistence.enabled .Values.source.persistence.existingClaim }}
          persistentVolumeClaim:
            claimName: {{ .Values.source.persistence.existingClaim }}
          {{- else if and .Values.source.persistence.enabled .Values.source.persistence.createPVC }}
          persistentVolumeClaim:
            claimName: {{ include "goldengate.sourceName" . }}-pvc
          {{- else }}
          emptyDir: {}
          {{- end }}

        - name: u03
          emptyDir: {}

        {{- with .Values.source.extraVolumes }}
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
