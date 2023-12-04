GitLab playbook
===============

This playbook and role installs the latest GitLab
“[omnibus](#omnibus-package)” all-in-one [RPM][about_rpm] package on
[AlmaLinux][alma_linux], [Rocky Linux][rocky_linux] or RHEL.

It is inspired by [a recent blog post][blog_post] (June 2023) by Josh Swanson
from Red Hat. Here we've reused the code from the blog post, fixed it, and we
demonstrate how to use it in real life in an AlmaLinux or Rocky Linux virtual
machine, managed by [Vagrant][about_vagrant].

It turns out that the [very popular][gitlab_role_galaxy]
“[ansible-role-gitlab][gitlab_role_repo]” for installing GitLab “omnibus” has
been discontinued more recently (Sept 2023). This is one more opportunity for
this project to be useful.

[blog_post]: https://www.redhat.com/en/blog/installing-gitlab-ce-rhel-9
[about_rpm]: https://en.wikipedia.org/wiki/RPM_Package_Manager
[alma_linux]: https://almalinux.org/
[rocky_linux]: https://rockylinux.org/
[about_vagrant]: https://www.vagrantup.com/
[gitlab_role_repo]: https://github.com/geerlingguy/ansible-role-gitlab
[gitlab_role_galaxy]: https://galaxy.ansible.com/geerlingguy/gitlab/


Prerequisites
-------------

You may run the Vagant box on your local machine, but here in this example we
run it on a distant machine, a bare-metal one, for more power and cheaper
price (compared to public clouds).

1. Provision your bare-metal box, like for instance [dedibox][dedibox]. Create
   it with Ubuntu Jammy (22.04) system. Install VirtualBox on it. We use an
   [Ansible role][virtualbox_ansible] for that, but it's up to you.

2. Create a DNS record named after `gitlab_external_fqdn` and pointing to the IP
   address of your VM. We use [Gandi][gandi] and [DNSControl][dnscontrol] for
   that, but it's up to you.

3. Install Vagrant and in that regards, `apt install vagrant` sould be fine.

4. You need your user to be sudoer with no password. For example, you may
   achieve this with such command:

        ```shell
        sudo sh -c "echo ${USER} ALL=(ALL) NOPASSWD: ALL > /etc/sudoers.d/${USER}"
        ```

[dedibox]: https://www.scaleway.com/en/dedibox/
[virtualbox_ansible]: https://github.com/gstackio/gstack-bosh-environment/tree/master/ddbox-standalone-bosh-env/provision/roles/virtualbox
[gandi]: https://gandi.net/
[dnscontrol]: https://github.com/StackExchange/dnscontrol



Usage
-----

1. Clone this repo, then go into it:

        ```shell
        git clone https://github.com/gstackio/gitlab-ansible.git
        cd gitlab-ansible
        ```

2. Ensure you have the required Vagrant plugins installed, then create the
   virtual machine:

        ```shell
        vagrant plugin install vagrant-reload vagrant-host-shell
        vagrant up
        ```

   Notice: when the VM is created, a networking shim is applied to the host's
   iptables, so that external HTTP/HTPS traffic is properly routed to the VM.
   The applied command looks like this:

        ```shell
        sudo iptables --table nat --insert PREROUTING 1 \
            --in-interface <interface-name>  --destination <public-ip> \
            --protocol tcp --match multiport --destination-ports 80,443 \
            --jump DNAT --to-destination 192.168.56.82
        ```

   You need not run the above command, as it is already bundled into the
   `vagrant up` stanza, but you may have to modify this in your context.

3. Write your variables file.

    First, we create the `vars.yml` file with the value for our
    `gitlab_external_fqdn` related to the DNS record above.

        ```shell
        cat > vars.yml <<EOF
        gitlab_external_fqdn: gitlab.example.org
        EOF
        ```

    We also need a strong secrets for the `gitlab_root_password` and the
    `gitlab_root_api_token`. The Bosh CLI is a nice tool for completing some
    YAML file, adding a random-generated secret when it's missing.

        ```bash
        chmod 600 vars.yml
        bosh interpolate "-" --vars-store="vars.yml" > /dev/null \
            <<<"--- { variables: [ { name: gitlab_root_password, type: password } ] }"
        bosh interpolate "-" --vars-store="vars.yml" > /dev/null \
            <<<"--- { variables: [ { name: gitlab_root_api_token, type: password, options: { length: 32 } } ] }"
        ```

    _(Here we use the `<<<` [Bash-ism][bashism] for injecting a string as
    standard input, and we use a special trick with the `--- ` prefix for
    writing a YAML file as a one-liner.)_

4. Invoke the Ansible Playbook

        ```shell
        ansible-playbook install-gitlab.yml --extra-vars @vars.yml
        ```

    _(We use the `@` prefix to specify a file containing our variables, where
    as usually the `-e` flag introduces explicit `var=value` key-pairs.)_

That's it! A Let's Encrypt TLS certificate will be provisioned and you'll be
able to log into your GitLab using on the `https://gitlab.example.org/`
URL.

Going further, you can optionally setup monitoring with Prometheus.

[bashism]: https://en.wiktionary.org/wiki/bashism


### Cleanup

You may uninstall GitLab completely:

 ```shell
 ansible-playbook uninstall-gitlab.yml
 ```

Or just destroy the VM completely:

```shell
vagrant destroy
```


### Limitations

The resulting GitLab installation is the most simple one. Full-blown,
monolithic installation, that cannot scale.


### Omnibus package

The name “omnibus” comes from the [Chef Omnibus][chef_omnibus] technology that
is used behind the scene to install GitLab. The names “_all-in-one_” or
“_official Linux_” package are more idiomatic and GitLab is thinking about
[adopting these names][rename_omnibus] in the future.

[chef_omnibus]: https://github.com/chef/omnibus
[rename_omnibus]: https://gitlab.com/groups/gitlab-org/-/epics/10716



Contributing
------------

Feel free to submit issues and pull requests.



Author and License
------------------

Copyright © 2023-present, Benjamin Gandon, Gstack

This Ansible playbook and role are released under the terms of the
[Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0).

<!--
# Local Variables:
# indent-tabs-mode: nil
# End:
-->
