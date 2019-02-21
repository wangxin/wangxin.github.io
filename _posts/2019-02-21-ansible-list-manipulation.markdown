---
layout: post
title:  "Ansible list manipulation"
date:   2019-02-21 11:00 +0800
categories: ansible, automation
---

It is very convenient to filter or modify a list using list comprehension in python. The same could be done in ansible playbook. However, the official documentation does not have a explicit guide for these operations.

I found a very good post explained how to do this:
[How to filter, join and map lists in Ansible](https://www.tailored.cloud/devops/how-to-filter-and-map-lists-in-ansible/)

Examples 

### Filter

The `select` filter can be used to filter a list. Indeed the `select` filter is not from ansible, it is from Jinjia. We can use all the Jinjia filters in ansible playbook. This filter takes two or more arguments. The first argument is name of a test. Rest of the arguments are for the test function.
List of tests we can use:
* [Ansible tests](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tests.html)
* [Jinjia tests](http://jinja.pocoo.org/docs/2.10/templates/#builtin-tests)

Examples
```
"{{ a_list_of_string | select('match', '^Start_of_string.*') | list }}"
"{{ a_list_of_string | select('search', 'subet_of_string') | list }}"
```

The `match` and `search` tests are explained in [Ansible tests](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tests.html). `match` requires a regular expression as argument for complete match. `search` only requires matching a subset of the string

### Map
The `map` filter is also from Jinjia. It takes one or more arguments. The first argument is name of another filter. A simple example:
```
"{{ a_list_of_string | map('upper') | list }}"
```
