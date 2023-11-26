Development notes
=================


Run from local machine
----------------------

Installing the latest Ansible version on Ubuntu is not particularily easy, so
you might want to run the playbook from your local machine, with the latest
Ansible version.

When you need to run the playbook distant, you may forward your local TCP port
`2222` to the distant localhost's `2222` port, that in turn, will be forwarded
by virtualbox to.

```shell
ssh -L 2222:127.0.0.1:2222 staging.example.org
```

In order to get the private key, you'll have to copy the disant `.vagrant`
directory first, running something like this:

```shell
rsync -a "staging.example.org:gitlab-ansible/.vagrant" "."
```

This is to be done after each `vagrant up`, as Vagrant re-generate a new
private key anytime it brings a box up.


Generating the `ansible.cfg` stub
---------------------------------

For creating the base `ansible.cfg` file that shows all possible vonfigs with
comments, we've used:

```shell
ansible-config init --disabled -t all > ansible.cfg
```
