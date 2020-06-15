How to include vars and tasks in Ansible
========================================

The following is based on the latest version of Ansible 2.9 (2.9.9) as of June 15, 2020.

Intro
-----

In the Ansible [System Roles project](https://linux-system-roles.github.io), the role code uses `include_vars` and `include_task` to include vars and tasks files which depend on the platform and
version.  For example, you may have a role which manages a system service,
and most of the code is platform independent, but the list of packages to
pass to the `package` module, and the name of the service to manage with the
`service` module, are platform specific.  We use an idiom like this in our
role `tasks/main.yml`:
```yaml
- name: Set version specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"
    - "{{ ansible_distribution }}_{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution }}.yml"
    - "{{ ansible_os_family }}.yml"
    - "default.yml"

- name: Ensure role packages
  package:
    name: "{{ __rolename_packages }}"
    state: present

- name: Ensure role services are enabled and running
  service:
    name: "{{ item }}"
    state: started
    enabled: yes
  loop: "{{ __rolename_services }}"
```
This uses the Ansible [first_found](https://docs.ansible.com/ansible/latest/plugins/lookup/first_found.html)
lookup plugin in the form of a `with_first_found` loop to load
the first matching variables file, where the order of the lookup is from most platform/version specific
to least.  That is, include the file that most specifically defines the variables used
by this role for the platform and version.
The file `vars/main.yml` is included by default when the role is included, and
contains platform independent variables, or variables with default values.  The
files such as `vars/RedHat.yml`, `vars/Fedora.yml`, `vars/RedHat_8.yml`, etc. contain
variable definitions specific to that platform/version.  That allows use to define
variables such as `__rolename_packages` and `__rolename_services` to be platform
dependent.

Problems
--------

One problem is that you may have to duplicate some definitions.  For example, if you have
some definitions that are the same in both `vars/CentOS_8.yml` and `vars/RedHat_8.yml`,
you will have to define everything in both files.  In some cases, you can symlink one
file to the other.

Another problem is when you have multiple roles that use the above idiom that include
each other, and not every role defines every platform/version specific file.  What can happen is that when one role includes
another role, the lookup path `ansible_search_path` will contain the path for both roles.
In this example, `role_a` includes `role_b`.
```yaml
- name: in role_a tasks/main.yml
  debug:
    msg: in role_a tasks/main.yml

- name: show ansible_search_path before
  debug:
    msg: ansible_search_path before {{ ansible_search_path }}
```
At this point, `ansible_search_path` will contain `['/base/roles/role_a','/base/roles/role_a/tasks']`.  Now, include `role_b`:
```yaml
- name: include role_b
  include_role:
    name: role_b
```
Where `role_b` looks like this:
```yaml
- name: show ansible_search_path in role_b
  debug:
    msg: ansible_search_path role_b {{ ansible_search_path }}
```
At this point, `ansible_search_path` will contain `['/base/roles/role_b','/base/roles/role_a','/base/roles/role_b/tasks']`.  Note the `/base/roles/role_a` in there.  Now, use the variable inclusion idiom:
```yaml
- name: Set version-specific variables for role_b
  include_vars: "{{ item }}"
  with_first_found:
    - files:
        - "{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"
        - "{{ ansible_distribution }}_{{ ansible_distribution_major_version }}.yml"
        - "{{ ansible_distribution }}.yml"
        - "{{ ansible_os_family }}.yml"
        - "default.yml"
```
If `role_b` does not define `vars/Fedora_31.yml`, but `role_a` does, `role_b` **will include** `vars/Fedora_31.yml` from `role_a`, and **will not include any vars** from `role_b`.

Solutions
---------

It is best to explicitly specify the path to use, rather than relying on the default behavior:
```yaml
- name: Set version-specific variables for role_b
  include_vars: "{{ item }}"
  with_first_found:
    - files:
        - "{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"
        - "{{ ansible_distribution }}_{{ ansible_distribution_major_version }}.yml"
        - "{{ ansible_distribution }}.yml"
        - "{{ ansible_os_family }}.yml"
        - "default.yml"
      paths:
        - "{{ role_path }}/vars"
```
The `role_path` is a built-in which will be `/base/rolename`, so you are guaranteed to
look in `/base/role_b/vars` for the files, and find only the files for your role.

Alternatives
------------

`with_first_found` will fail if no files are found.  If you don't want to create files
for every possible combination of platform/version, or you don't want to create `vars/default.yml`,
you can use the `skip: true` flag:
```yaml
- name: Set version-specific variables for role
  include_vars: "{{ item }}"
  with_first_found:
    - files:
        - "{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"
        - "{{ ansible_distribution }}.yml"
      paths:
        - "{{ role_path }}/vars"
      skip: true
```
In this example, I rely on `vars/main.yml` for most everything, and only provide some
platform/version customizations.

Another way to do this is the following:
```yaml
- name: Set version-specific variables for role
  include_vars: "{{ item }}"
  loop:
    - "{{ role_path }}/vars/{{ ansible_os_family }}.yml"
    - "{{ role_path }}/vars/{{ ansible_distribution }}.yml"
    - "{{ role_path }}/vars/{{ ansible_distribution }}_{{ ansible_distribution_major_version }}.yml"
    - "{{ role_path }}/vars/{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"
  when: item is file
```
The files in the `loop` are in order from least specific to most specific.
Each file in the `loop` list will allow you to add or override additional
variables to specialize the values for platform and/or version.  Using the
`when: item is file` test means that you do not have to provide all of the
`vars/` files, only the ones you need.  For example, if every platform except
Fedora uses `srv_name` for the service name, you can define `__myrole_service:
srv_name` in `vars/main.yml` then define `__myrole_service: srv2_name` in
`vars/Fedora.yml`. In cases where this would lead to duplicate vars files for
similiar distibutions (e.g. CentOS 7 and RHEL 7), use symlinks to avoid the
duplication.

Tasks
-----

Platform specific tasks, however, are different.  You probably want to perform
platform specific tasks once, for the most specific match.  In that case, use
`with_first_found` with the file list in order of most specific to least
specific, including a "default":
```yaml
- name: Perform platform/version specific tasks
  include_tasks: "{{ item }}"
  with_first_found:
    - files:
        - "setup_{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"
        - "setup_{{ ansible_distribution }}_{{ ansible_distribution_major_version }}.yml"
        - "setup_{{ ansible_distribution }}.yml"
        - "setup_{{ ansible_os_family }}.yml"
        - "setup_default.yml"
      paths:
        - "{{ role_path }}/tasks"
```
And same with the vars files above, if you don't want to have to provide the default,
or only some files, use `skip: true`:
```yaml
- name: Perform platform/version specific tasks
  include_tasks: "{{ item }}"
  with_first_found:
    - files:
        - "setup_{{ ansible_distribution }}_{{ ansible_distribution_version }}.yml"
        - "setup_{{ ansible_distribution }}.yml"
      paths:
        - "{{ role_path }}/tasks"
      skip: true
```
