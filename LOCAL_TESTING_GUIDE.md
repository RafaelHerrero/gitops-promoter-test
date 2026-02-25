# 🧪 Local Testing Guide for GitOps Promoter

This guide will help you test the gitops-promoter Helm chart on your local Kubernetes cluster with your own GitHub repository.

---

## 📋 Prerequisites

- ✅ Local Kubernetes cluster running (minikube, kind, Docker Desktop, or k3d)
- ✅ `kubectl` configured and pointing to your local cluster
- ✅ `helm` CLI installed (v3.x)
- ✅ GitHub account with a test repository
- ✅ GitHub CLI (`gh`) installed (optional, but helpful)

---

## Phase 1: Prepare GitHub Repository

### Step 1.1: Create or Use Existing Test Repository

**Option A: Create New Repository**
```bash
# Using GitHub CLI
gh repo create gitops-promoter-test --public --clone

cd gitops-promoter-test

# Initialize with README
echo "# GitOps Promoter Test" > README.md
git add README.md
git commit -m "Initial commit"
git push -u origin main
```

**Option B: Use Existing Repository**
```bash
# Clone your existing test repo
git clone https://github.com/YOUR_USERNAME/YOUR_TEST_REPO.git
cd YOUR_TEST_REPO
```

### Step 1.2: Create Environment Branches

```bash
# Ensure you're on main branch
git checkout main

# Create environment branches
git checkout -b environment/development
git push -u origin environment/development

git checkout main
git checkout -b environment/staging
git push -u origin environment/staging

git checkout main
git checkout -b environment/production
git push -u origin environment/production

# Return to main
git checkout main
```

**Verify branches exist:**
```bash
git branch -r | grep environment
# Expected output:
#   origin/environment/development
#   origin/environment/staging
#   origin/environment/production
```

---

## Phase 2: Set Up GitHub Authentication

You need either a **GitHub App** (recommended) or **Personal Access Token** (simpler).

### Option A: GitHub Personal Access Token (Simpler) ⭐

**Step 2.1: Create Personal Access Token**

1. Go to: https://github.com/settings/tokens/new
2. Note description: `gitops-promoter-local-test`
3. Expiration: 30 days (or as needed)
4. Select scopes:
   - ✅ `repo` (Full control of private repositories)
   - ✅ `read:org` (Read org and team membership) - if using org repos
5. Click "Generate token"
6. **COPY THE TOKEN** (you won't see it again!)

**Step 2.2: Store Token Securely**

```bash
# Save token to a file (do NOT commit this!)
echo "YOUR_GITHUB_TOKEN_HERE" > ~/.github-promoter-token

# Protect the file
chmod 600 ~/.github-promoter-token

# Verify
cat ~/.github-promoter-token
```

**Step 2.3: Create Kubernetes Secret**

```bash
# Create argocd namespace
kubectl create namespace argocd

# Create secret with GitHub token
kubectl create secret generic github-promoter-token \
  -n argocd \
  --from-file=token=~/.github-promoter-token

# Verify secret created
kubectl get secret github-promoter-token -n argocd
```

---

### Option B: GitHub App (Advanced)

**Step 2.1: Create GitHub App**

1. Go to: https://github.com/settings/apps/new
2. Fill in:
   - **GitHub App name**: `gitops-promoter-test`
   - **Homepage URL**: `https://github.com/YOUR_USERNAME/gitops-promoter-test`
   - **Webhook**:
     - Active: ❌ (uncheck for now)
   - **Repository permissions**:
     - Contents: `Read and write`
     - Pull requests: `Read and write`
     - Metadata: `Read-only`
   - **Where can this GitHub App be installed?**: `Only on this account`
3. Click **Create GitHub App**
4. Note the **App ID** (e.g., `123456`)
5. Click **Generate a private key** → Download the `.pem` file

**Step 2.2: Install GitHub App to Repository**

1. On the app settings page, click **Install App**
2. Select your test repository
3. Click **Install**
4. Note the **Installation ID** from URL:
   - URL format: `https://github.com/settings/installations/12345678`
   - Installation ID: `12345678`

**Step 2.3: Create Kubernetes Secret**

```bash
# Create namespace
kubectl create namespace argocd

# Create secret with GitHub App credentials
kubectl create secret generic github-app-secret \
  -n argocd \
  --from-literal=app-id=YOUR_APP_ID \
  --from-literal=installation-id=YOUR_INSTALLATION_ID \
  --from-file=private-key=/path/to/downloaded-private-key.pem

# Verify
kubectl get secret github-app-secret -n argocd
```

---

## Phase 3: Prepare Helm Values File

### Step 3.1: Create Local Test Values

Create a file: `local-test-values.yaml`

**For GitHub Personal Access Token:**

```yaml
# Disable all addons - we only want gitops-promoter
addons:
  karpenter:
    enabled: false
  external-secrets:
    enabled: false
  cert-manager:
    enabled: false
  cilium:
    enabled: false

# GitOps Promoter Configuration
gitopsPromoter:
  enabled: true
  installCRDs: true  # Install CRDs in local cluster
  
  namespace: promoter-system
  createNamespace: true
  
  # ScmProvider configuration (GitHub)
  scmProvider:
    enabled: true
    name: local-scm
    secretRef:
      name: github-promoter-token  # Secret created in Phase 2
    github:
      appID: 0  # 0 for PAT (not using GitHub App)
      installationID: 0  # 0 for PAT
      repoName: gitops-promoter-test  # ⚠️ CHANGE THIS to your repo name
      githubOrgName: YOUR_GITHUB_USERNAME  # ⚠️ CHANGE THIS to your username/org
  
  # PromotionStrategy configuration
  promoterStrategy:
    name: test-strategy
    gitRepositoryRef:
      name: test-repo
    environments:
      - branch: environment/development
        autoMerge: true  # Auto-merge PRs to development
      - branch: environment/staging
        autoMerge: true  # Auto-merge PRs to staging
      - branch: environment/production
        autoMerge: false  # Require manual approval for production
  
  # Hydrator (optional - for ArgoCD integration)
  hydrator:
    enabled: false  # Disable for basic testing without ArgoCD
    applicationSelector:
      matchLabels:
        tier: infrastructure
  
  # Pull request templates
  pullRequestTemplate:
    title: "Promote {{ .ChangeTransferPolicy.Status.Proposed.Dry.Subject }} to {{ .ChangeTransferPolicy.Spec.ActiveBranch }}"
    description: |
      This PR is promoting the environment branch `{{ .ChangeTransferPolicy.Spec.ActiveBranch }}`
      which is currently on dry sha {{ .ChangeTransferPolicy.Status.Active.Dry.Sha }}
      to dry sha {{ .ChangeTransferPolicy.Status.Proposed.Dry.Sha }}.
      
      Commit message: {{ .ChangeTransferPolicy.Status.Proposed.Dry.Subject }}
      Author: {{ .ChangeTransferPolicy.Status.Proposed.Dry.Author }}
```

**For GitHub App:**

```yaml
# Disable all addons
addons:
  karpenter:
    enabled: false
  external-secrets:
    enabled: false
  cert-manager:
    enabled: false
  cilium:
    enabled: false

gitopsPromoter:
  enabled: true
  installCRDs: true
  
  namespace: promoter-system
  createNamespace: true
  
  scmProvider:
    enabled: true
    name: local-scm
    secretRef:
      name: github-app-secret  # Secret created in Phase 2
    github:
      appID: 123456  # ⚠️ CHANGE THIS to your GitHub App ID
      installationID: 12345678  # ⚠️ CHANGE THIS to your Installation ID
      repoName: gitops-promoter-test  # ⚠️ CHANGE THIS
      githubOrgName: YOUR_GITHUB_USERNAME  # ⚠️ CHANGE THIS
  
  promoterStrategy:
    name: test-strategy
    gitRepositoryRef:
      name: test-repo
    environments:
      - branch: environment/development
        autoMerge: true
      - branch: environment/staging
        autoMerge: true
      - branch: environment/production
        autoMerge: false
  
  hydrator:
    enabled: false
  
  pullRequestTemplate:
    title: "Promote {{ .ChangeTransferPolicy.Status.Proposed.Dry.Subject }} to {{ .ChangeTransferPolicy.Spec.ActiveBranch }}"
    description: |
      Promoting SHA {{ .ChangeTransferPolicy.Status.Proposed.Dry.Sha }}
      to {{ .ChangeTransferPolicy.Spec.ActiveBranch }}
```

### Step 3.2: Customize Values

**⚠️ IMPORTANT: Edit `local-test-values.yaml` and change:**

1. `repoName`: Your repository name
2. `githubOrgName`: Your GitHub username or organization
3. `appID` / `installationID`: (if using GitHub App)

---

## Phase 4: Install GitOps Promoter

### Step 4.1: Verify Cluster is Ready

```bash
# Check cluster connection
kubectl cluster-info

# Check nodes are ready
kubectl get nodes

# Verify namespace doesn't exist yet
kubectl get namespace argocd 2>/dev/null || echo "Namespace doesn't exist yet (OK)"
```

### Step 4.2: Install Helm Chart

```bash
# From the repository root directory
cd /Users/rherrero/git/schuberg-philis/vpf-cloud-container/vpf-cloud-container-platform

# Install the chart
helm install gitops-promoter-test charts/sbp \
  --namespace argocd \
  --create-namespace \
  --values local-test-values.yaml \
  --wait \
  --timeout 5m

# Expected output:
# NAME: gitops-promoter-test
# LAST DEPLOYED: ...
# NAMESPACE: argocd
# STATUS: deployed
```

### Step 4.3: Verify Installation

```bash
# Check controller pod is running
kubectl get pods -n promoter-system

# Expected output:
# NAME                                          READY   STATUS    RESTARTS   AGE
# promoter-controller-manager-xxxxxxxxxx-xxxxx  2/2     Running   0          1m

# Check CRDs installed
kubectl get crds | grep promoter.argoproj.io | wc -l
# Expected: 12

# Check custom resources created
kubectl get promotionstrategy,scmprovider,gitrepository -n argocd

# Expected output:
# NAME                                            READY
# promotionstrategy.promoter.argoproj.io/test-strategy   True
#
# NAME                                    READY
# scmprovider.promoter.argoproj.io/local-scm   True
#
# NAME                                        PROVIDER    READY
# gitrepository.promoter.argoproj.io/test-repo   local-scm   True
```

### Step 4.4: Check Resource Status

```bash
# Check PromotionStrategy status
kubectl get promotionstrategy test-strategy -n argocd -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'

# Expected: {"status":"True","type":"Ready"}

# Check ScmProvider status
kubectl get scmprovider local-scm -n argocd -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'

# Expected: {"status":"True","type":"Ready"}

# Check GitRepository status
kubectl get gitrepository test-repo -n argocd -o jsonpath='{.status.conditions[?(@.type=="Ready")]}'

# Expected: {"status":"True","type":"Ready"}
```

### Step 4.5: Check Controller Logs

```bash
# View recent controller logs
kubectl logs -n promoter-system -l control-plane=controller-manager -c manager --tail=50

# Look for successful reconciliation messages:
# INFO  Reconciling PromotionStrategy  {"name": "test-strategy"}
# INFO  Reconciling PromotionStrategy End  {"duration": "..."}
# DEBUG Reconciliation successful
```

---

## Phase 5: Test Promotion Workflow

### Step 5.1: Make a Test Commit

```bash
# Navigate to your test repository
cd /path/to/your/test/repo

# Ensure you're on main branch
git checkout main

# Create a test file
echo "Test change at $(date)" > test-change.txt
git add test-change.txt
git commit -m "Test promotion workflow"
git push origin main
```

### Step 5.2: Monitor Promotion Activity

```bash
# Watch controller logs in real-time
kubectl logs -n promoter-system -l control-plane=controller-manager -c manager --follow

# In another terminal, watch for ChangeTransferPolicies
watch -n 5 'kubectl get changetransferpolicies -n argocd'

# In another terminal, watch for PullRequests
watch -n 5 'kubectl get pullrequests -n argocd'
```

### Step 5.3: Verify Promotion

**After 1-2 minutes (polling interval), you should see:**

```bash
# Check ChangeTransferPolicies created
kubectl get changetransferpolicies -n argocd

# Expected: 3 policies (one per environment)
# test-strategy-environment-development-xxxxxxxx
# test-strategy-environment-staging-xxxxxxxx
# test-strategy-environment-production-xxxxxxxx

# Check PullRequest resources
kubectl get pullrequests -n argocd

# Check promotion status
kubectl get promotionstrategy test-strategy -n argocd -o yaml | grep -A 20 "environments:"
```

### Step 5.4: Check GitHub for Pull Requests

```bash
# Using GitHub CLI
gh pr list --repo YOUR_USERNAME/gitops-promoter-test

# Or open in browser
open https://github.com/YOUR_USERNAME/gitops-promoter-test/pulls
```

**Expected behavior:**
1. ✅ PR created for `environment/development-next` → `environment/development`
2. ✅ PR auto-merged (because `autoMerge: true`)
3. ✅ PR created for `environment/staging-next` → `environment/staging`
4. ✅ PR auto-merged (because `autoMerge: true`)
5. ✅ PR created for `environment/production-next` → `environment/production`
6. ⏸️ PR **waiting** for manual merge (because `autoMerge: false`)

### Step 5.5: View Promotion History

```bash
# View detailed environment status
kubectl get promotionstrategy test-strategy -n argocd -o jsonpath='{range .status.environments[*]}{"\nEnvironment: "}{.branch}{"\n  Active SHA:   "}{.active.dry.sha}{"\n  Proposed SHA: "}{.proposed.dry.sha}{"\n  PR State:     "}{.pullRequest.state}{"\n  PR URL:       "}{.pullRequest.url}{"\n"}{end}'
```

### Step 5.6: Manually Approve Production PR

```bash
# Get production PR number
kubectl get pullrequests -n argocd -o jsonpath='{range .items[?(@.status.state=="open")]}{.status.id}{" - "}{.status.url}{"\n"}{end}'

# Merge using GitHub CLI
gh pr merge <PR_NUMBER> --repo YOUR_USERNAME/gitops-promoter-test --merge

# Or merge manually in GitHub UI
open https://github.com/YOUR_USERNAME/gitops-promoter-test/pulls
```

---

## Phase 6: Verify Everything Works

### Step 6.1: Check Final State

```bash
# Check all environments are synchronized
kubectl get promotionstrategy test-strategy -n argocd -o yaml | grep -E "branch:|sha:" | head -20

# View promotion history
kubectl get promotionstrategy test-strategy -n argocd -o jsonpath='{.status.environments[0].history}' | jq '.'
```

### Step 6.2: Verify Branches in GitHub

```bash
# Check that environment branches have your commit
cd /path/to/your/test/repo

# Fetch latest
git fetch --all

# Check development
git log origin/environment/development --oneline | head -5

# Check staging
git log origin/environment/staging --oneline | head -5

# Check production (after manual merge)
git log origin/environment/production --oneline | head -5
```

---

## 🔧 Troubleshooting

### Issue: Controller Pod Not Starting

```bash
# Check pod status
kubectl get pods -n promoter-system

# Describe pod
kubectl describe pod -n promoter-system -l control-plane=controller-manager

# Check events
kubectl get events -n promoter-system --sort-by='.lastTimestamp'
```

**Common causes:**
- CRDs not installed: `helm upgrade --set gitopsPromoter.installCRDs=true`
- Image pull issues: Check image registry access
- Resource limits: Check cluster has enough resources

---

### Issue: ScmProvider Not Ready

```bash
# Check ScmProvider status
kubectl describe scmprovider local-scm -n argocd

# Check secret exists and has correct format
kubectl get secret github-promoter-token -n argocd -o yaml

# Check controller logs for auth errors
kubectl logs -n promoter-system -l control-plane=controller-manager -c manager --tail=100 | grep -i "auth\|token\|github"
```

**Common causes:**
- Invalid GitHub token: Regenerate token with correct scopes
- Token expired: Create new token
- Wrong secret name: Verify `secretRef.name` in values matches secret name

---

### Issue: No Pull Requests Created

```bash
# Check if controller is polling
kubectl logs -n promoter-system -l control-plane=controller-manager -c manager --tail=200 | grep -i "reconcil"

# Check GitRepository status
kubectl get gitrepository test-repo -n argocd -o yaml

# Check for errors in ChangeTransferPolicy
kubectl get changetransferpolicies -n argocd
kubectl describe changetransferpolicy <name> -n argocd
```

**Common causes:**
- Repository not accessible: Check GitHub permissions
- Branches don't exist: Verify environment branches created
- Controller not reconciling: Check controller logs for errors

---

### Issue: GitHub Authentication Failed

```bash
# Test GitHub token manually
TOKEN=$(kubectl get secret github-promoter-token -n argocd -o jsonpath='{.data.token}' | base64 -d)
curl -H "Authorization: token $TOKEN" https://api.github.com/user

# Expected: Your GitHub user info (JSON)
# If error: Token is invalid or expired
```

---

## 🧹 Cleanup

### Remove Everything

```bash
# Uninstall Helm release
helm uninstall gitops-promoter-test -n argocd

# Delete CRDs (if you want to start fresh)
kubectl delete crds -l app.kubernetes.io/part-of=promoter

# Delete namespace
kubectl delete namespace argocd
kubectl delete namespace promoter-system

# Delete GitHub secret from local file
rm ~/.github-promoter-token
```

### Keep Installation but Reset State

```bash
# Delete ChangeTransferPolicies
kubectl delete changetransferpolicies -n argocd --all

# Delete PullRequests
kubectl delete pullrequests -n argocd --all

# Restart controller
kubectl rollout restart deployment promoter-controller-manager -n promoter-system
```

---

## 📊 Success Checklist

After completing this guide, you should have:

- ✅ Local Kubernetes cluster running
- ✅ GitOps Promoter controller deployed and healthy
- ✅ All 12 CRDs installed
- ✅ GitHub authentication working
- ✅ PromotionStrategy, ScmProvider, and GitRepository resources ready
- ✅ Test commit triggering automatic promotion
- ✅ Pull requests created in GitHub
- ✅ Auto-merge working for dev/staging
- ✅ Manual approval working for production
- ✅ Environment branches updated with commits

---

## 🎯 Next Steps

Once local testing works:

1. **Fix template bugs** (see main plan)
2. **Vendor CRDs** into chart
3. **Update documentation**
4. **Create comprehensive tests**
5. **Open PR** with all fixes

---

## 📚 Additional Resources

- [GitOps Promoter Documentation](https://github.com/argoproj-labs/gitops-promoter)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)

---

## 💡 Tips

- **Start simple**: Test with just one environment first
- **Watch logs**: Keep `kubectl logs --follow` running during tests
- **Use GitHub CLI**: Makes checking PRs much faster
- **Small commits**: Easier to track through promotion pipeline
- **Polling frequency**: Default is ~1 minute, be patient
- **Clean state**: Delete and recreate if things get messy

---

**Questions or issues?** Check the Troubleshooting section or review controller logs for specific errors.
