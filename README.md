# Few Tips with Ansible

### Pipes combos

#### Ternary Combo

```sh
# shwo container with restaarting mode
- name: '[check_install.yml] : Verify if containers is up {{ docker_container_name }}'
  docker_container_info: 
    name: "{{ docker_container_name }}"
  register: container_info
# Check if docker deployment is OK
- name: '[check_install.yml] : Check if docker deployment is OK'
  debug:
    msg: "Deploiement du container {{ docker_container_name }} est {{ (container_info.exists and container_info.container.State.Running) | ternary('OK', 'KO') }}"
```

## Attributes

### Selestattr vs Map

#### Selectattr -> ansible_facts

Take the all the object where the attribute is located

```sh
- name: 'Setting fact mount'
  set_fact:
    mount: "{{ ansible_mounts | selectattr('mount')|list }}"
```

#### Map

Take only the attribute and his value.

```sh
---
- hosts: node1
  tasks:
  - name: ansible_mount
    set_fact:
      mount_name: "{{ ansible_mounts | map(attribute='mount') | list }}"
      size_available: "{{ ansible_mounts | map(attribute='size_available') | list}}"
      full_size: "{{ ansible_mounts | map(attribute='size_total') | list }}"
  - name: check vars
    debug:
      msg: |
        {% if item.0 == '/' %}root{% else %}{{ item.0 }}{% endif %} - {{ item.1 }} - {{ item.2 }}
    with_together:
    - "{{ mount_name }}"
    - "{{ size_available }}"
    - "{{ full_size }}"
```

## Loops tips & tricks

### Mix external and internal loops -> [example](https://github.com/sofackj/mini-projects#ansible-project-i--may-be-used-for-docker-containers-ansible_and_vars)

- **Structure as below**

```sh
roles/
└── deploy_app
    └── tasks
        ├── deploy.yml
        └── main.yml
```

- **Files content**

**```main.yml```**

```sh
- include_tasks: deploy.yml
  # In deploy.yml {{ item }} will be used as variable
  with_items: "{{ ext_list }}"
```

**```deploy.yml```**

```sh
- name: what is nicu
  debug:
    # {{ item }} represents an element of ext_list
    # {{ local_item }} represents an element of int_list via the use of the loop_control module
    msg: "My ID is {{ item }} for {{ local_item }}"
  loop: "{{ int_list }}"
  loop_control:
    loop_var: local_item
```

### Use of with_together

#### Two lists

```sh
- name: item.0 returns from the 'a' list, item.1 returns from the '1' list
  ansible.builtin.debug:
    msg: "{{ item.0 }} and {{ item.1 }}"
  with_together:
    - ['a', 'b', 'c', 'd']
    - [1, 2, 3, 4]
```

#### Three lists combined with condition

```sh
- name: 'Ensures that the available size is greater than 30% of the total size'
  assert:
    that: item.0 > item.1|float * 0.3
    fail_msg: disk size : 70 %
  with_together:
    - "{{ size_available }}"
    - "{{ full_size }}"
    - "{{ all_partition }}"
  when: item.2 == docker_mount_fs[0]
```

### Ansible and Perfomance
Check this [link]([https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html](https://devopssec.fr/article/accelerer-performances-playbook))

```sh
- hosts: web
  gather_facts: False
  gather_subset:
    - network
    - virtual
```

### Debug or Fail module : Way to present the msg

```sh
- name: Print some debug information 
    vars: 
      msg:
      - msg 1
      - msg 2
      - msg 3
          ...
```

### Check the value of a variable

```sh
# Verify is variables are defined
- name: '[check_variables.yml] : Check all required vars'
  fail: 
    msg: "'{{ item }}' doit etre definie"
  when: (item is undefined) or (item is none) or ((item | trim) == '')
  loop: "{{ docker_required_vars | default([]) }}"
```

### Assert module
Check the documentation [here](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html)

```sh
- name: After version 2.7 both 'msg' and 'fail_msg' can customize failing assertion message
  ansible.builtin.assert:
    that:
      - my_param <= 100
      - my_param >= 0
    fail_msg: "'my_param' must be between 0 and 100"
    success_msg: "'my_param' is between 0 and 100"
```
