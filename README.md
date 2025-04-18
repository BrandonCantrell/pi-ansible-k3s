## 🧩 Variable Structure: Bootstrap vs Post-Bootstrap

This repository uses a clean separation of variables to support a two-phase setup:

1. **Bootstrap Phase** – run as the Pi’s default user (e.g., `pi`) to prepare the system
2. **Post-Bootstrap Phase** – connect using the newly created `ansible` user

---

### Inventory Layout

```
inventories/
└── example/
    ├── inventory.yml
    └── group_vars/
        ├── all.yml                # Shared variables (used in both phases)
        └── k3s_cluster.yml        # Connection vars (used only after bootstrap)
```

---

### `group_vars/all.yml` (shared between both phases)

### `group_vars/k3s_cluster.yml` (used after bootstrap)


## First-Time Setup (Bootstrap Phase)

This repository assumes your Raspberry Pi devices are running a fresh install of **Ubuntu 22.04 Server (64-bit)** and are accessible using a default user (commonly `pi`, `ubuntu`, etc.).

The `bootstrap.yml` playbook prepares the Pi for Ansible-based management and should be run **once per node** using the initial default user.


---

### Step 1: Generate an SSH Key for Ansible Access

Before running the playbook, generate a new SSH keypair on your control machine that will be used by the `ansible` user:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_id
```

This will create:

- Private key: `~/.ssh/ansible_id`
- Public key: `~/.ssh/ansible_id.pub`

Then copy the **public key** into the `files/` directory of this repository:

```bash
cp ~/.ssh/ansible_id.pub files/ansible.pub
```

> Ensure the filename matches `{{ ansible_target_user }}.pub` — by default, this is `ansible.pub`.

---

### Step 2: Run the Bootstrap Playbook

From your control machine, run the following command to prepare each Raspberry Pi:

```bash
ansible-playbook -i inventories/example playbooks/bootstrap.yml -u pi --ask-become-pass
```

Replace `pi` with the default user for your device if it's different (e.g., `ubuntu`).

This playbook will:
- Install baseline packages (e.g., `curl`, `vim`, `git`, `htop`)
- Create a dedicated `ansible` user
- Add your public SSH key to that user
- Configure passwordless sudo
- Set the hostname to match the inventory hostname

---

### After Bootstrap

Once the playbook completes, you should be able to connect to the Pi using:

```bash
ssh ansible@<pi-ip> -i ~/.ssh/ansible_id
```

You can now run all other playbooks (like `site.yml`) using the `ansible` user defined in `group_vars/all.yml`. No further use of the default Pi user is required unless you intend to use this user to manage the cluster. 

## Creating Additional System Users

The `users.yml` playbook allows you to define and create additional system users on each Raspberry Pi node. This is useful for creating local users like `docker`, `localuser`, or others specific to your environment.

These users are defined in your `group_vars/all.yml` file under the `additional_users` variable.

---

### Example Configuration (`group_vars/all.yml`)

```yaml
additional_users:
  - name: docker
    sudo: true
  - name: localuser
    sudo: false
```

- `name`: the system username to create
- `sudo`: whether the user should be granted passwordless sudo access

---

### Running the Playbook

To create the users defined above, run:

```bash
ansible-playbook -i inventories/example playbooks/users.yml
```

This will:
- Create each user with a home directory and bash shell
- Grant sudo access (via `/etc/sudoers.d/`) if `sudo: true` is specified

You can customize the `additional_users` list for your needs. Users will only be created if they do not already exist.

>If you want to add SSH keys for these users, you can extend the playbook to include `authorized_key` tasks.
