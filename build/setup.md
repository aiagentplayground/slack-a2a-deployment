


helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
    --namespace kagent \
    --create-namespace


helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
    --namespace kagent \
    --set providers.default=anthropic \
    --set providers.anthropic.model="claude-haiku-4-5" \
    --set providers.anthropic.apiKey=$ANTHROPIC_API_KEY 

kubectl patch svc kagent-ui -n kagent -p '{"spec": {"type": "LoadBalancer"}}'

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: grok-api-keys
  namespace: kagent
type: Opaque
stringData:
  PROVIDER_API_KEY: ${XAI_API_KEY}
EOF

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: anthropic-api-keys
  namespace: kagent
type: Opaque
stringData:
  ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
EOF

export PROVIDER_API_KEY=$XAI_API_KEY

kubectl create secret generic kagent-xai-provider -n kagent \
    --from-literal PROVIDER_API_KEY=$XAI_API_KEY

kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: xai-grok-config
  namespace: kagent
spec:
  apiKeySecret: kagent-xai-provider
  apiKeySecretKey: PROVIDER_API_KEY
  model: grok-4-fast-reasoning
  provider: OpenAI
  openAI:
    baseUrl: "https://api.x.ai/v1"
EOF
