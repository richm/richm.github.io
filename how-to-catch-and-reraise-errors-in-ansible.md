How to catch and re-raise errors in Ansible
========================================

The following is based on the latest version of Ansible 2.9 and 2.10 as of November 13, 2020.

Intro
-----

There are cases where you are using a role from inside a block, and the role
itself uses a block, and you want to propagate errors from the inner block to be
handled in a `rescue` in an outer block, and you want access to the
`ansible_failed_result` from the inner block.
{% raw %}
```yaml
---
- name: test nested blocks
  hosts: localhost
  tasks:
    - block:
        - name: first outer level task
          debug:
            msg: first outer level task
        - block:
            - name: first inner level task
              debug:
                msg: first inner level task
            - name: inner task that fails
              fail:
                msg: inner task that fails
            - name: inner task that should never execute
              debug:
                msg: inner task that should never execute
        - name: second outer level task
          debug:
            msg: second outer level task
      rescue:
        - name: outer block rescue task
          debug:
            msg: outer block rescue task ansible_failed_result is {{ ansible_failed_result | d("undefined") | to_nice_json }}
        - name: check for error message
          assert:
            that: ansible_failed_result.msg == "inner task that fails"
```
{% endraw %}
This case works fine - the error from the inner block is correctly propagated to the `rescue` in the outer block with the correct context information.

Problem - ansible_failed_result is undefined
--------------------------------------------

The problem arises when you want to use `always` or `rescue` in the inner block:
```yaml
...
        - block:
            - name: first inner level task
              debug:
                msg: first inner level task
            - name: inner task that fails
              fail:
                msg: inner task that fails
            - name: inner task that should never execute
              debug:
                msg: inner task that should never execute

          always:
            - name: inner always task
              debug:
                msg: inner always task e.g. remove temporary directory
...
```
The outer `rescue` block is still called, which means an error was detected by
Ansible and handled, but the context information is gone and
`ansible_failed_result` is undefined:
```
TASK [outer block rescue task] *************************************************
task path: /home/rmeggins/ansible_sandbox/nested-block-two.yml:27
ok: [localhost] => {}

MSG:

outer block rescue task ansible_failed_result is "undefined"

TASK [check for error message] *************************************************
task path: /home/rmeggins/ansible_sandbox/nested-block-two.yml:30
fatal: [localhost]: FAILED! => {}

MSG:

The conditional check 'ansible_failed_result.msg == "inner task that fails"' failed. The error was: error while evaluating conditional (ansible_failed_result.msg == "inner task that fails"): 'ansible_failed_result' is undefined
```
Same if a `rescue` is used in the inner block, with or without the `always`.

Solution - re-raise the ansible_failed_result
---------------------------------------------
You can re-raise the error with the `ansible_failed_result` by using it as
the only value for a `fail` module `msg` argument:
{% raw %}
```yaml
...
        - block:
            - name: first inner level task
              debug:
                msg: first inner level task
            - name: inner task that fails
              fail:
                msg: inner task that fails
            - name: inner task that should never execute
              debug:
                msg: inner task that should never execute
          rescue:
            - name: re-raise the error
              fail:
                msg: "{{ ansible_failed_result }}"
          always:
            - name: inner always task
              debug:
                msg: inner always task e.g. remove temporary directory
...
```
{% endraw %}
Now we get the desired result:
```
TASK [outer block rescue task] *************************************************
task path: /home/rmeggins/ansible_sandbox/nested-block.yml:38
ok: [localhost] => {}

MSG:

outer block rescue task ansible_failed_result is {
    "_ansible_no_log": false,
    "changed": false,
    "failed": true,
    "msg": "inner task that fails"
}

TASK [check for error message] *************************************************
task path: /home/rmeggins/ansible_sandbox/nested-block.yml:41
ok: [localhost] => {
    "changed": false
}

MSG:

All assertions passed
```

Here is the complete example, with some extra tasks to show the behavior:
{% raw %}
```yaml
---
- name: test nested blocks
  hosts: localhost
  tasks:
    - block:
        - name: first outer level task
          debug:
            msg: first outer level task
        - block:
            - name: first inner level task
              debug:
                msg: first inner level task
            - name: inner task that fails
              fail:
                msg: inner task that fails
            - name: inner task that should never execute
              debug:
                msg: inner task that should never execute
          rescue:
            - name: inner block rescue task
              debug:
                msg: inner block rescue task ansible_failed_result is {{ ansible_failed_result | d("undefined") | to_nice_json }}
            - name: re-raise error
              fail:
                msg: "{{ ansible_failed_result }}"
          always:
            - name: first inner block always task
              debug:
                msg: first inner block always task ansible_failed_result is {{ ansible_failed_result | d("undefined") | to_nice_json }}
        - block:
            - name: second block inner level task
              debug:
                msg: second block inner level task
        - name: second outer level task
          debug:
            msg: second outer level task
      rescue:
        - name: outer block rescue task
          debug:
            msg: outer block rescue task ansible_failed_result is {{ ansible_failed_result | d("undefined") | to_nice_json }}
        - name: check for error message
          assert:
            that: ansible_failed_result.msg == "inner task that fails"
```
{% endraw %}
Run like this:
```
ANSIBLE_STDOUT_CALLBACK=debug \
ansible-playbook -vv nested-block-re-raise.yml
```
