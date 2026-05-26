# Quay Performance Test - Full Setup Instructions

## Prerequisites
- `oc` CLI logged in with cluster-admin privileges
- Access to the target OpenShift cluster

## Step 1: Create namespace
```bash
oc new-project test-quay
```

## Step 2: Grant privileged SCC (requires cluster-admin)
These cannot be done via YAML - must be run as separate commands:
```bash
oc adm policy add-scc-to-user privileged -z quay-perf -n test-quay
oc adm policy add-scc-to-user privileged -z default -n test-quay
```
- `quay-perf` SA is used by the orchestrator pod
- `default` SA is used by the worker pods (push/pull jobs spawned by the orchestrator)

Both need privileged SCC because the containers run with `privileged: true` for podman-in-pod.

## Step 3: Update full-setup.yaml
Before deploying, update these values in `full-setup.yaml`:
- `QUAY_HOST` - your Quay registry hostname
- `QUAY_PASSWORD` - Quay admin password
- `QUAY_USERNAME` - Quay admin username
- `QUAY_OAUTH_TOKEN` - Quay OAuth token (from Quay org application)
- `QUAY_ORG` - Quay organization to run tests against
- `CUSTOM_BUILD_IMAGE` - image used as base for push tests
- `TEST_NAMESPACE` - must match the namespace you created (e.g., test-quay)
- `TAGS` - comma-separated list of image tags to pull-test
- `PUSH_PULL_NUMBERS` - number of images to push/pull
- `TARGET_HIT_SIZE` - number of API target hits
- `CONCURRENCY` - number of concurrent workers
- `TEST_BATCH_SIZE` - tags per worker pod batch
- `SKIP_PUSH` - set to "true" to skip push tests, "false" to include them

## Step 4: Deploy
```bash
oc apply -f full-setup.yaml -n test-quay
```
This creates: ServiceAccount, Role, RoleBinding, Redis (Deployment + Service), and the orchestrator Job.

## Step 5: Monitor
```bash
# Watch all pods
oc get pods -n test-quay -w

# Follow orchestrator logs
oc logs -f job/quay-perf-test-orchestrator-1 -n test-quay

# Follow worker pod logs (name appears in orchestrator logs)
oc logs -f job/test-registry-pullXXXXX -n test-quay
```

## Cleanup
```bash
oc delete job quay-perf-test-orchestrator-1 -n test-quay
oc delete job -l quay-perf-test-component-pull -n test-quay
oc delete job -l quay-perf-test-component-push -n test-quay
```

## Re-run
Delete the old job first, then re-apply:
```bash
oc delete job quay-perf-test-orchestrator-1 -n test-quay
oc apply -f full-setup.yaml -n test-quay
```
Redis and RBAC resources persist - no need to recreate them.

## Troubleshooting

### SCC Forbidden errors on pods
Ensure both SCC grants from Step 2 were applied. Verify with:
```bash
oc adm policy who-can use scc privileged -n test-quay
```

### Worker pods show wrong image output (e.g., cluster-version-operator)
`PUSH_PULL_IMAGE` must be set to the quay-load test image, not the image you want to test. Use `TAGS` for images to test.

### readOnlyRootFilesystem errors
The emptyDir volume at `/tmp/logs` handles this. If a cluster SCC enforces readOnlyRootFilesystem, the emptyDir mount provides a writable path.
