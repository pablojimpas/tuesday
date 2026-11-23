<!--
SPDX-FileCopyrightText: 2026 Pablo Jiménez Pascual <https://github.com/pablojimpas/ash_opencollection>

SPDX-License-Identifier: MIT
-->

# Tuesday Walkthrough: Organization -> Project -> Task -> Comment in OpenCollection

In this beginner-friendly tutorial, you will explore how **Tuesday**—a multi-tenant project and task management application built with the Ash Framework—maps its domain hierarchy into a concrete, schema-valid OpenCollection v1.0.0 API suite.

All concrete collection files referenced in this guide live in [`collections/tuesday/`](../../collections/tuesday/opencollection.yml).

---

## 1. What You Will Learn

1. How Tuesday models its 4-level hierarchy: **`Organization` -> `Project` -> `Task` -> `Comment`**.
2. How attribute-based multitenancy (`x-organization: {{organization_id}}`) flows cleanly through generated request headers.
3. How topological sorting and `set-variable` actions chain IDs across requests (`organization_id` -> `project_id` -> `task_id` -> `comment_id`).
4. How to open the suite in Bruno or validate and run it in pure Elixir without Node.js.

---

## 2. Tuesday's Domain Model (`Organization -> Project -> Task -> Comment`)

Tuesday organizes these four core Ash resources across two domains (`Tuesday.Workspace` and `Tuesday.Projects`):

1. **`Tuesday.Workspace.Organization`** (`lib/tuesday/workspace/organization.ex`):
   - Attributes: `name` (string, required), `slug` (string), `plan_type` (`:free | :premium | :enterprise`), `can_standard_member_create_project` (boolean).
   - Actions: `:create_org_with_owner`, `:read`, `:update_org`, `:change_org_plan`, `:destroy`.
2. **`Tuesday.Projects.Project`** (`lib/tuesday/projects/project.ex`):
   - Multitenancy: `strategy :attribute`, `attribute :organization_id`, `global? true`.
   - Relationship: `belongs_to :organization, Tuesday.Workspace.Organization` (`allow_nil?: false`).
   - Attributes: `name`, `description`, `start_date`, `end_date`, `status` (`:active | :archived | :completed`).
   - Actions: `:create_project`, `:list_projects`, `:update_project`, `:archive_project`, `:complete_project`, `:activate_project`.
3. **`Tuesday.Projects.Task`** (`lib/tuesday/projects/task.ex`):
   - Multitenancy: `strategy :attribute`, `attribute :organization_id`, `global? true`.
   - Relationships: `belongs_to :project, Tuesday.Projects.Project` (`allow_nil?: false`), `belongs_to :parent_task, Tuesday.Projects.Task`.
   - Attributes: `title`, `description`, `is_complete` (boolean), `priority` (`1..5`), `story_point` (`1..21`), `start_date`, `due_date`.
   - Actions: `:create_task`, `:list_tasks`, `:list_project_tasks`, `:update_task`, `:complete_task`, `:destroy`.
4. **`Tuesday.Projects.Comment`** (`lib/tuesday/projects/comment.ex`):
   - Multitenancy: `strategy :attribute`, `attribute :organization_id`, `global? true`.
   - Relationships: `belongs_to :task, Tuesday.Projects.Task` (`allow_nil?: false`), `belongs_to :author, Tuesday.Workspace.OrganizationMember`.
   - Attributes: `body` (string, required).
   - Actions: `:create_comment`, `:read`, `:update_comment`, `:destroy`.

---

## 3. Concrete Collection Layout (`collections/tuesday/`)

Tuesday's concrete collection contains 23 OpenCollection v1.0.0 YAML files organized in strict topological order:

```text
collections/tuesday/
├── opencollection.yml
├── environments/
│   └── local.yml
├── 01-organizations/
│   ├── folder.yml
│   ├── 01_create_organization.yml
│   ├── 02_get_organization.yml
│   └── 03_change_org_plan.yml
├── 02-projects/
│   ├── folder.yml
│   ├── 01_create_project.yml
│   ├── 02_list_projects.yml
│   └── 03_update_project.yml
├── 03-tasks/
│   ├── folder.yml
│   ├── 01_create_task.yml
│   ├── 02_list_project_tasks.yml
│   └── 03_complete_task.yml
├── 04-comments/
│   ├── folder.yml
│   ├── 01_create_comment.yml
│   ├── 02_list_task_comments.yml
│   └── 03_update_comment.yml
└── 05-cleanup/
    ├── folder.yml
    ├── 01_delete_comment.yml
    ├── 02_delete_task.yml
    ├── 03_archive_project.yml
    └── 04_delete_organization.yml
```

Once you expose your Ash domains via `AshJsonApi`, you can regenerate or drift-check your collection at any time with:

```bash
mix ash_opencollection.codegen \
  --output ./collections/tuesday \
  --domains Tuesday.Workspace,Tuesday.Projects \
  --prefix /api/v1
```

---

## 4. Walking Through Chained Variables & Multitenancy

### Step 1: Create the Organization (`01-organizations/01_create_organization.yml`)

The first request creates an `Organization` via `:create_org_with_owner` and extracts `organization_id` from `res.body.data.id`:

```yaml
info:
  name: Create Organization with Owner
  type: http
  seq: 1
http:
  method: POST
  url: "{{base_url}}/api/v1/organizations"
  headers:
    - name: Accept
      value: application/vnd.api+json
    - name: Content-Type
      value: application/vnd.api+json
  body:
    type: json
    data: |-
      {
        "data": {
          "attributes": {
            "member": {
              "email": "{{owner_email}}",
              "role": "owner",
              "username": "{{owner_username}}"
            },
            "name": "Acme Engineering",
            "plan_type": "enterprise"
          },
          "type": "organization"
        }
      }
runtime:
  actions:
    - type: set-variable
      phase: after-response
      selector:
        expression: res.body.data.id
        method: jsonq
      variable:
        name: organization_id
        scope: runtime
  assertions:
    - expression: res.status
      operator: eq
      value: "201"
    - expression: res.body.data.id
      operator: isDefined
```

### Step 2: Create a Tenant-Scoped Project (`02-projects/01_create_project.yml`)

Every project belongs to an organization and uses `organization_id` multitenancy. `02-projects/01_create_project.yml` passes both the `x-organization: {{organization_id}}` header and `"organization_id": "{{organization_id}}"`, then captures `project_id`.

### Step 3: Create & Complete a Task (`03-tasks/01_create_task.yml` & `03_complete_task.yml`)

[`collections/tuesday/03-tasks/01_create_task.yml`](../../collections/tuesday/03-tasks/01_create_task.yml) uses `{{organization_id}}` and `{{project_id}}` to create a task with `priority: 4`, then extracts `task_id`:

```yaml
http:
  method: POST
  url: "{{base_url}}/api/v1/tasks"
  headers:
    - name: Accept
      value: application/vnd.api+json
    - name: Content-Type
      value: application/vnd.api+json
    - name: x-organization
      value: "{{organization_id}}"
  body:
    type: json
    data: |-
      {
        "data": {
          "attributes": {
            "description": "Automate collection generation and CI drift detection",
            "due_date": "2026-11-15",
            "priority": 4,
            "project_id": "{{project_id}}",
            "start_date": "2026-11-01",
            "title": "Design OpenCollection Generator"
          },
          "type": "task"
        }
      }
  auth:
    type: bearer
    token: "{{token}}"
```

[`03_complete_task.yml`](../../collections/tuesday/03-tasks/03_complete_task.yml) sends a JSON:API 1.0 `PATCH` with `"id": "{{task_id}}"` inside `"data"` and asserts `res.body.data.attributes.is_complete` with operator `isTrue`.

### Step 4: Post & Update a Task Comment (`04-comments/01_create_comment.yml`)

[`collections/tuesday/04-comments/01_create_comment.yml`](../../collections/tuesday/04-comments/01_create_comment.yml) posts to `{{base_url}}/api/v1/tasks/{{task_id}}/comments` and captures `comment_id` for subsequent reads, updates, and cleanup in `05-cleanup/`.

---

## 5. Opening in Bruno or Validating in Elixir

### Option A: Open in Bruno Desktop

Open the Bruno desktop app, click **Open Collection**, and choose `./collections/tuesday`. All five folders, environment variables (`environments/local.yml`), headers, and declarative assertions appear ready to run.

### Option B: Validate & Run in Pure Elixir

Every YAML file in `collections/tuesday/` can be parsed and verified with `open_collection`:

```elixir
for path <- Path.wildcard("collections/tuesday/**/*.yml") do
  doc = path |> File.read!() |> OpenCollection.parse!()
  {:ok, _} = OpenCollection.Schema.validate(doc)
  {:ok, _} = OpenCollection.Declarative.validate(doc)
end
```

And when your Phoenix endpoint is running (or mounted in `ConnCase`), execute the entire collection in ExUnit with `AshOpenCollection.Test`:

```elixir
defmodule TuesdayWeb.ApiCollectionTest do
  use TuesdayWeb.ConnCase, async: false
  use AshOpenCollection.Test

  test_collection "collections/tuesday", env: "local"
end
```
