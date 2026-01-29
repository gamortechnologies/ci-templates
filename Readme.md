
---

# 🧱 Separating Template Workflows from Calling Workflows

## 🧠 The core principle

> **Reusable workflows are libraries.
> Calling workflows are applications.**

Libraries must be:

* independent
* versioned
* reusable across repos

---

## 1️⃣ Create a dedicated CI templates repository

### Example repo name

```text
my-org/ci-templates
```

### Structure (important!)

```text
ci-templates/
└── .github/
    └── workflows/
        ├── terraform-matrix.yml
        ├── node-ci.yml
        └── docker-build.yml
```

📌 Only **reusable workflows** live here
📌 No `on: push`, only `workflow_call`

---

## 2️⃣ Define the reusable workflow (library code)

### 📄 `terraform-matrix.yml`


No triggers except `workflow_call`.
This file **cannot run alone** — by design.

---

## 3️⃣ Version the templates repo

```bash
git commit -m "Initial reusable Terraform CI"
git tag v1
git push origin v1
```

Now your CI library has a **stable API**.

---

## 4️⃣ Call the template from an application repo

### Example app repo

```text
my-org/my-terraform-app
└── .github/workflows/
    └── terraform-ci.yml
```

### 📄 Calling workflow

```yaml
name: Terraform CI

on:
  push:
  pull_request:

jobs:
  terraform:
    uses: my-org/ci-templates/.github/workflows/terraform-matrix.yml@v1
    with:
      terraform_versions: '["1.6.6"]'
      os_list: '["ubuntu-latest"]'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

✔ Clean separation
✔ Versioned dependency
✔ Safe upgrades

---

## 5️⃣ Why this separation is mandatory for versioning

If templates and callers live together:

| Problem            | Why it breaks               |
| ------------------ | --------------------------- |
| No real versioning | Caller always sees latest   |
| No rollback        | Tag moves affect everything |
| Tight coupling     | CI logic tied to app        |
| Unsafe changes     | Breaks pipelines silently   |

Separate repo = **real dependency boundary**.

---

## 6️⃣ How teams usually organize this

### Typical enterprise layout

```text
github.com/my-org/
├── ci-templates        ← reusable workflows (versioned)
├── terraform-network   ← calls templates
├── terraform-app       ← calls templates
├── node-api            ← calls templates
```

CI becomes **platform infrastructure**, not project glue.

---

## 🧠 Final mental model (lock this in)

| Concept           | Reality        |
| ----------------- | -------------- |
| Reusable workflow | Shared library |
| Calling workflow  | Application    |
| Version tag       | Release        |
| Repo boundary     | API boundary   |

> If it’s versioned, it must be **separate**.

---

## 🏁 Final takeaway

You don’t “copy” CI anymore —
you **depend on it**.

That’s the jump from:

* writing pipelines
  to
* **building CI platforms**

If you want next:

* central secrets handling
* backward-compatible CI changes
* composite actions vs reusable workflows
* CI template release workflow (yes, CI for CI)

Just say where you want to go 🚀
