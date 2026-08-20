# Ansible Automation Platform examples

Repository contains classroom examples for learning Ansible and Red Hat Ansible Automation Platform (AAP). It progresses from standalone playbooks to task includes, reusable roles, collection packaging, REST API automation, and an application lifecycle solution.

## Repository structure

```text
20260817aap/
├── README.md
├── LICENSE
├── playbooks/
│   ├── tasks/                     # Tasks included by a parent playbook
│   ├── minecraft_example/         # Build/teardown lifecycle solution
│   └── playbook-role-example/     # Playbook with a local role
└── collection_examples/
    └── rzfeeser/aap/              # Skeleton of the rzfeeser.aap collection
```

The folders show increasing levels of reuse:

1. A standalone playbook keeps a solution in one YAML file.
2. `playbooks/tasks/` extracts per-item logic that another playbook can include repeatedly.
3. `playbooks/playbook-role-example/roles/` packages related tasks as a role local to one playbook project.
4. `collection_examples/rzfeeser/aap/` packages a role, metadata, and potential plugins for installation and reuse across projects.

## Requirements and basic usage

Check the installed version:

```bash
ansible --version
```

Most remote examples target an inventory group named `planetexpress`. No classroom inventory is included, so supply one containing that group:

```bash
ansible-playbook -i /path/to/inventory playbooks/tuesdayam_example.yml
```

Playbooks targeting `localhost` generally need no inventory:

```bash
ansible-playbook playbooks/uri_example.yml
```

Depending on the example, you also need SSH access, privilege-escalation rights, Git, Docker, the `community.docker` collection, API connectivity, and valid credentials.

> **Security warning:** several classroom files contain controller URLs, passwords, and token-like values. Treat embedded credentials as compromised: rotate them, remove them from source control, and supply replacements through environment variables, Ansible Vault, or AAP credentials.

## `playbooks/`: runnable examples

### `monday_afternoon.yml`

An introductory remote-host playbook. It targets `planetexpress`, creates `~/afterlunch.txt`, and clones the API Quest Git repository into `~/api-quest`. It establishes the basic play/task/module structure used by later solutions.

### `tuesdayam_example.yml`

Introduces file management. It creates `/tmp/i-was-here/`, touches `zach.txt`, and writes timestamped content to `ansible.txt`. The same change-making tasks later appear behind preflight checks in `wednesdayam.yml` and the role example.

### `wednesdayam.yml`

Gathers facts and asserts that each target:

- Runs Debian or Ubuntu.
- Has at least 10 MB of RAM.
- Has at least two processor cores.
- Is accessed as a non-root user.

Only after those checks does it perform the Tuesday file changes. This is the bridge to the `serverprechecks` role, where the assertions are extracted for reuse. Before running, correct the core comparison from `> = 2` to `>= 2`.

### `loops01.yml`

Loops over a list of dictionaries and creates the users `fry`, `bender`, and `zoidberg`, using each item's name, comment, and shell. It requires privilege escalation. This introduces the structured-loop pattern later used by the batch AAP user solution.

### `loop02_example.yml`

Collects service facts from every `planetexpress` host, loops over the returned service dictionary, and delegates report writing to localhost. It creates `services-run-on-<hostname>.txt` relative to the directory where Ansible is run, demonstrating facts, dictionary iteration, delegation, and per-host output.

### `uri_example.yml`

Runs locally, makes an OMDb HTTP GET request with `ansible.builtin.uri`, registers the response, and prints its JSON. This is the small precursor to the authenticated AAP API workflows. Replace the sample API key before use.

### `studentmaker.yml`

The single-user AAP API prototype. Inline tasks build authentication headers, optionally obtain a bearer token from basic credentials, find or create one user, find an organization, add the user to it, grant organization administrator access, and print a summary.

It is useful for seeing the whole workflow in one file, while the split batch solution below is more reusable. The final two status messages appear reversed relative to their conditions and should be verified before operational use.

### `aap_create_org_admins_via_api.yml`

The batch orchestration layer. It builds authentication headers, optionally requests a token, generates `student06` through `student12` with the `sequence` lookup, and loops over those usernames. Each iteration includes `tasks/aap_create_org_admin_user.yml`.

The parent playbook owns shared configuration and iteration; the included file owns one user's workflow. Update the controller, authentication, organization, user defaults, and sequence range for the target environment.

### `tasks/aap_create_org_admin_user.yml`

This is an included task file, not a standalone playbook. For the `user_username` passed by the parent, it idempotently:

- Finds or creates the user and determines its ID.
- Finds the target organization and fails if it is absent.
- Adds the user to the organization if necessary.
- Grants organization administrator access if necessary.
- Prints a summary.

It inherits variables such as `controller_host`, `api_headers`, `validate_certs`, user fields, and `target_org_name` from `aap_create_org_admins_via_api.yml`. Run the parent playbook rather than this file directly.

## `playbooks/minecraft_example/`: lifecycle solution

This folder groups related playbooks for one application's full lifecycle.

- `main.yml` is the dispatcher. It imports the build playbook when `tobuild=present` and the teardown playbook when `tobuild=absent`. The variable has no active default, so pass it explicitly.
- `minecraft.yml` creates `/opt/minecraft`, starts `itzg/minecraft-server:latest` with persistent data and port `25565`, waits for the port, and displays connection information.
- `minecraft-taredown.yml` removes the container and deletes `/opt/minecraft/`, including saved world data. Its filename preserves the repository's current `taredown` spelling.

```bash
ansible-playbook playbooks/minecraft_example/main.yml -e tobuild=present
ansible-playbook playbooks/minecraft_example/main.yml -e tobuild=absent
```

The teardown path is destructive; back up valuable world data first.

## `playbooks/playbook-role-example/`: local role solution

This folder refactors the Wednesday prechecks into a reusable role stored beside its consumer.

- `playbook-with-role.yml` gathers facts, runs the local `serverprechecks` role, and then creates the `/tmp/i-was-here/` files. Ansible discovers the adjacent `roles/` directory automatically.
- `roles/serverprechecks/tasks/main.yml` contains the Debian/Ubuntu, memory, CPU, and non-root assertions extracted from `wednesdayam.yml`.
- `roles/serverprechecks/meta/main.yml` contains generated Galaxy metadata; its author, description, license, version, and tags are placeholders.
- `roles/serverprechecks/tests/test.yml` is generated test scaffolding, but names `linux-svr-generic-prechecks` rather than `serverprechecks`; align the name before use.
- `roles/serverprechecks/tests/inventory` is an empty test-inventory scaffold.
- `roles/serverprechecks/README.md` is an uncustomized role-documentation template.

This layout is appropriate when a role belongs to one playbook project. The collection folder demonstrates distributing essentially the same role.

## `collection_examples/`: distributable solution

`collection_examples/rzfeeser/aap/` is a skeleton for the collection `rzfeeser.aap`.

- `galaxy.yml` is the collection build/publication manifest: namespace, name, version, README, authors, license, dependencies, links, tags, and exclusions. Many values remain placeholders.
- `README.md` is the collection-level documentation placeholder.
- `meta/runtime.yml` can declare the minimum Ansible version, plugin routing and deprecation, import redirects, and action groups. Its examples are currently commented out.
- `plugins/` is where custom modules and other plugin types would live. It currently contains only an explanatory README, not working plugins.
- `roles/serverprechecks/` is the collection-packaged copy of the precheck role, including tasks, metadata, tests, and documentation scaffolding.

The collection copy's CPU assertion also uses `> = 2`; change it to `>= 2`. Its test role name and generated metadata/documentation need the same cleanup as the local role.

Once the collection is completed, built, and installed, consumers can use the fully qualified role name:

```yaml
roles:
  - rzfeeser.aap.serverprechecks
```

The relationship is therefore: the local role demonstrates reuse inside one project, while the collection copy demonstrates packaging that solution for reuse across projects and AAP execution environments.

## How the solutions relate

| Concept | Introductory form | Expanded/reusable form |
|---|---|---|
| Remote changes | `monday_afternoon.yml` | `tuesdayam_example.yml` manages several files |
| Preflight safety | Checks inline in `wednesdayam.yml` | Checks extracted to the local `serverprechecks` role |
| Role distribution | `playbook-role-example/roles/serverprechecks/` | `collection_examples/rzfeeser/aap/roles/serverprechecks/` |
| Loops | `loops01.yml` loops over user dictionaries | The batch AAP playbook loops over generated usernames and includes tasks |
| HTTP APIs | `uri_example.yml` makes one GET request | AAP examples perform authenticated lookup/create/associate workflows |
| AAP users | `studentmaker.yml` handles one user inline | Parent playbook plus task include handles a reusable batch |
| Application lifecycle | Separate Minecraft build and teardown files | `minecraft_example/main.yml` selects the required path |

## Before production use

- Rotate and remove embedded secrets.
- Replace classroom URLs, organizations, usernames, and API keys.
- Supply and test the missing inventory.
- Correct both malformed `> =` comparisons.
- Complete generated role and collection metadata and documentation.
- Align role tests with the actual role name.
- Pin image and collection versions where reproducibility matters.
- Test destructive playbooks on disposable systems first.

## License

This repository is distributed under GNU GPL version 3. See `LICENSE`.
