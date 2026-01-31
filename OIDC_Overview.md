
---

## What Is OIDC?

**OIDC (OpenID Connect)** is a secure identity protocol that allows GitHub Actions to authenticate with AWS **without long-term credentials**.

### How the Flow Works

1. A **GitHub Actions workflow starts**
2. The workflow **requests an OIDC token** from GitHub’s OIDC provider
3. GitHub returns a **signed JWT token** containing identity claims such as:

   * Repository name
   * Branch
   * Environment
4. The workflow sends this token to **AWS STS** and requests to assume an **IAM role**
5. AWS STS:

   * Validates the token
   * Checks the IAM role trust policy
   * Verifies claims (repo, branch, audience, etc.)
6. If everything matches, AWS STS issues **temporary AWS credentials**
7. The workflow uses these credentials to access AWS services such as:

   * S3
   * ECR
   * Lambda

✅ The credentials are **short-lived**
✅ **No static secrets**
✅ Automatically expire when the job ends

---

## GitHub Actions → AWS OIDC Grant Flow (High-Level)

![Image](https://devopscube.com/content/images/size/w1200/2025/05/ACTIONS-OIDC-YT-2.png)


---

## Step-by-Step Grant Flow (What the Diagram Shows)

```
┌────────────────────┐
│ GitHub Repository  │
│ (Workflow starts)  │
└─────────┬──────────┘
          │
          │ 1️⃣ Request OIDC token
          ▼
┌─────────────────────────────┐
│ GitHub OIDC Provider        │
│ token.actions.githubusercontent.com │
└─────────┬───────────────────┘
          │
          │ 2️⃣ Signed JWT token
          │    (repo, branch, env, aud)
          ▼
┌─────────────────────────────┐
│ GitHub Actions Runner       │
│ (Job execution)             │
└─────────┬───────────────────┘
          │
          │ 3️⃣ AssumeRoleWithWebIdentity
          │    (JWT token)
          ▼
┌─────────────────────────────┐
│ AWS STS                     │
│ - Validates token           │
│ - Checks IAM trust policy   │
└─────────┬───────────────────┘
          │
          │ 4️⃣ Temporary credentials
          ▼
┌─────────────────────────────┐
│ AWS Services                │
│ (S3, ECR, Lambda, etc.)     │
└─────────────────────────────┘
```

---

## Key Security Checks in the Flow

### 🔐 Token Validation (AWS STS)

AWS verifies:

* **Issuer**: GitHub OIDC provider
* **Audience**: `sts.amazonaws.com`
* **Subject (`sub`)**:

  ```
  repo:ORG/REPO:ref:refs/heads/BRANCH
  ```

### 🛡 IAM Trust Policy Example (Conceptual)

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:org/repo:*"
    }
  }
}
```

---

## 1️⃣ What is **`aud` (Audience)?**

### Short answer

**`aud` is the entity the token is *meant for*** — **not** the one that issues it.

In GitHub → AWS OIDC:

```
aud = sts.amazonaws.com
```

That means:

> “This token was issued **specifically for AWS STS**.”

---

### Why `aud` matters

OIDC tokens are reusable *in theory*, so AWS must ensure:

* The token was **not meant for another service**
* The token cannot be replayed elsewhere

AWS STS checks:

```text
Is aud == sts.amazonaws.com ?
```

If not → ❌ **Rejected**


---

## 2️⃣ What is **`sub` (Subject)?**

### Short answer

**`sub` uniquely identifies *who* the token represents**

In GitHub Actions, `sub` describes:

* Repository
* Branch / tag / PR
* Workflow context

Example:

```
sub = repo:gamortechnologies/my-repo:ref:refs/heads/main
```

---

### Why `sub` exists

OIDC needs a **stable, unique identity** to attach permissions to.

Think of it as:

> “Which *exact workload* is asking for access?”

Without `sub`, AWS would only know:

* “This came from GitHub” ❌

With `sub`, AWS knows:

* “This came from **this repo, on this branch**” ✅

---

## 3️⃣ Why AWS *needs* `sub`

AWS uses `sub` in the **IAM trust policy** to decide **who is allowed to assume the role**.

Example:

```json
"Condition": {
  "StringLike": {
    "token.actions.githubusercontent.com:sub":
      "repo:gamortechnologies/my-repo:ref:refs/heads/main"
  }
}
```

This means:
✔ Only this repo
✔ Only this branch
✔ Only this workflow context

---

> “GitHub issued a token **for AWS STS (`aud`)** on behalf of **this exact repo & branch (`sub`)**.”

---

## 5️⃣ Why Both Are Required (Together)

If AWS only checked:

### ❌ `aud` without `sub`

* Any GitHub repo could access your AWS account

### ❌ `sub` without `aud`

* Token could be replayed against other services

### ✅ `aud` + `sub`

* Correct service
* Correct workload
* Correct scope

---

## 6️⃣ One-Line Intuition (Memorable)

> **`aud` protects *where* the token can be used**
> **`sub` protects *who* is allowed to use it**

---

## 7️⃣ Bonus: Common GitHub `sub` Formats

| Context     | Example                              |
| ----------- | ------------------------------------ |
| Branch      | `repo:org/repo:ref:refs/heads/main`  |
| Tag         | `repo:org/repo:ref:refs/tags/v1.0.0` |
| PR          | `repo:org/repo:pull_request`         |
| Environment | `repo:org/repo:environment:prod`     |

---

