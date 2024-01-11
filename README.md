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

### Setup your bare-metal host

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


### Setup the guest VM

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



Usage
-----

### Deploy GitLab

1. Ensure you have the required Ansible collections and roles properly
   installed.

        ```shell
        ansible-galaxy install -r requirements.yml
        ```

2. Setup your inventory.

    Following the example provided in `inventory/main.yml`, configure your
    GitLab host and customize the associated variables, especially
    `gitlab_external_fqdn`.

        ```yaml
        gitlab:
          hosts:
            localhost: # <- whatever IP/DNS for targetting the GitLab host through SSH
              # ...
              gitlab_external_fqdn:  gitlab.example.org # <- the DNS record setup above
        ```

3. Write private variables file.

    Following the example provided in `vars.example.yml`, create a
    `vars/prod.yml` file (or `vars/staging.yml`, or `vars/sandbox.yml`,
    depending on the environment you're configuring) and make it private.

        ```bash
        cp vars.example.yml vars/prod.yml
        chmod 600 vars/prod.yml
        ```

    We need strong secrets for `gitlab_root_password` and
    `gitlab_root_api_token`. The Bosh CLI is a nice tool for completing some
    YAML file, adding a random-generated secret when it's missing:

        ```bash
        bosh interpolate "-" --vars-store="vars/prod.yml" > /dev/null \
            <<<"--- { variables: [ { name: gitlab_root_password, type: password } ] }"
        bosh interpolate "-" --vars-store="vars/prod.yml" > /dev/null \
            <<<"--- { variables: [ { name: gitlab_root_api_token, type: password, options: { length: 32 } } ] }"
        ```

    _(Here we use the `<<<` [Bash-ism][bashism] for injecting a string as
    standard input, and we use a special trick with the `--- ` prefix for
    writing a YAML file as a one-liner.)_

4. Invoke the Ansible playbook

        ```shell
        ansible-playbook playbooks/install-gitlab.yml --extra-vars @vars/prod.yml
        ```

    _(We use the `@` prefix to specify a file containing our variables, where
    as usually the `-e` flag introduces explicit `var=value` key-pairs.)_

That's it! A Let's Encrypt TLS certificate will be provisioned and you'll be
able to log into your GitLab using on the `https://gitlab.example.org/`
URL.

Going further, you can optionally setup monitoring with Prometheus.

[bashism]: https://en.wiktionary.org/wiki/bashism


### Easily create your own PKI with certstrap

#### Create your own PKI

On your GitLab host, install `cerstrap`:

```shell
curl --location \
    --url https://github.com/square/certstrap/releases/download/v1.3.0/certstrap-linux-amd64 \
    --output /usr/local/bin/certstrap
chmod +x /usr/local/bin/certstrap
```

Create the Certificate Authority:

```shell
certstrap --depot-path="/etc/gitlab/trusted-certs/pki" init \
    --common-name=GitLab_CA
```

Create the server certificate:

```shell
cd /etc/gitlab/trusted-certs
certstrap --depot-path="pki" request-cert \
    --common-name="gitlab.example.org" --domain="gitlab.example.org"
certstrap --depot-path="pki" sign \
    "gitlab.example.org" --CA "GitLab_CA"
```

In order to make these available to GitLab, establish a link from the
`/etc/gitlab/ssl` directory:

```shell
cd /etc/gitlab/trusted-certs
cp -a pki/gitlab.example.org.{crt,key} .
cd ../ssl
ln -sf ../trusted-certs/gitlab.example.org.* .
```

#### Reconfigure GitLab with its TLS certificate

Then trigger a non-significant change in `/etc/gitlab/gitlab.rb`, like adding
an empty comment `#` at then end of the file, so that Ansible sees a
difference and can trigger the `gitlab-ctl reconfigure` handler properly.

With this, run the `ansible-playbook` command again.

#### Setup client workstations

On your workstations, copy the `/etc/gitlab/trusted-certs/pki/GitLab_CA.crt`
file to your Windows hosts, double-click it, and choose carrefully the
_trusted root certificates_ store where to instal it. As a confirmation you
did right, you'll get an extra confirmation popup because your adding a CA.

On macOS hosts, copy the same file and open it (double-click or use the `open`
CLI command) to add ot to the _Key Chains_ app. double-click it in _Key
Chains_ and in the opened window for this certificate, unroll the “when to
trust” selector, to choose “always trust” and close then window. At that
moment, _Key Chains_ will ask for your password as a confirmation of the
certificate trust action, enter it and you're done.

Navigators may not recognize your certificate immediately, first still
displaying the `https:` in red and strike-through font. But in fact, it will be
trusted.


### Create groups

As prerequisites for group creation, you need an Ansible runner, where the
Python requirements listed in `requirements.txt` are met. The playbook is
designed to use `localhost` as runner, using your user.

We recommend you install the required Python version specified in
`.python-version` using `pyenv` so that it won't mess up with anything else
you may have on your system, and verify with `which python` that you're now
using the version from `pyenv` after proper setup.

Then we recommend you  use `virtualenvwrapper` and create the virtual env with
`mkvirtualenv gitlab-ansible`, then enter it with `workon gitlab-ansible` and
populate it with `python3 -m pip install ansible`, `pip install --upgrade pip`
and `pip install -r requirements.txt`. Verify with `which ansible` that you're
now using the version from virtual env.

The inventory config for the `gitlab_installs` group (that is specifically
used for setting up GitLab) is as follows.

```yaml
ansible_user: your-username # as in `echo $USER`
ansible_connection: local
ansible_python_interpreter: "<you-user-$HOME-path>/.virtualenvs/gitlab-ansible/bin/python3"
```

Ensure you have the required Ansible collections and roles properly installed.

```shell
ansible-galaxy install -r requirements.yml
```

This finishes your `localhost` Ansible runner setup.

Once these preprequisites are met, you may run the groups creation as follows.

```shell
ansible-playbook playbooks/setup-gitlab.yml --extra-vars @vars/prod.yml
```


### Cleanup

You may uninstall GitLab completely:

 ```shell
 ansible-playbook playbooks/uninstall-gitlab.yml
 ```

Or just destroy the guest VM completely:

```shell
vagrant destroy
```


### “Extra config” example

The required [OmniAuth configuration][gitlab_omniauth] for authenticating to
GitLab using Single-Sign On (SSO), is not directly exposed as variables in the
Ansible role, but the `gitlab_extra_settings` configuration property brings
enough flexibility to properly set it up.

Here we provide an example of SSO configuration, based on some SAML identity
provider, which happens to be _SafeNet Trusted Access_ (STA) here, but could
also be any other SAML provider. The overall check-list for configuring any
SAML provider is the following:

1. Read the identity provider (IdP) documentation. In this case, this is the
   “[SafeNet Trusted Access for GitLab][sta_gitlab]” article.

2. Obtain the IdP XML metadata file.

3. Configure as demonstrated below, using the values from the IdP metadata
   obtained above.

4. Fetch the XML metadata from the GitLab instance at
   `https://gitlab.example.org/users/auth/saml/metadata`

5. Have the GitLab instance properly registered in the IdP using the metadata
   fetched above.

[gitlab_omniauth]: https://docs.gitlab.com/ee/integration/omniauth.html
[sta_gitlab]: https://resources.eu.safenetid.com/help/GitLab/Index.htm


#### Example SAML Single-Sign On configuration

In the following configuration example, you'll need to replace the following
bits:

- `gitlab.example.org`, which is the GitLab instance hostname, to be replaced
  by the exact value from your instalation.

- The `idp_cert_fingerprint` value, which is the (case-insensitive) SHA1
  fingerprint of the identity provider (IdP) certificate, comming
  **from the IdP metadata XML** (see the [caveats](#caveats) below).

- The `<REAML-IDENTIFIER>`, which is the “authentication realm” identifier
  from your SafeNet Trusted Access (STA) subscription, to be replaced by the
  actual value.

```yaml
gitlab_extra_settings:
  - gitlab_rails:
      - key: omniauth_allow_single_sign_on
        value: [ saml ]
      - key: omniauth_block_auto_created_users
        value: false
      - key: omniauth_providers
        type: plain
        value: |
          [
            {
              name: "saml",
              args: {
                assertion_consumer_service_url: "https://gitlab.example.org/users/auth/saml/callback",
                idp_cert_fingerprint: "E3:F1:16:B3:91:E8:CA:86:61:FB:0A:15:CD:6B:8D:56:EB:BC:56:23",
                idp_sso_target_url: "https://idp.safenetid.com/auth/realms/<REAML-IDENTIFIER>/protocol/saml",
                issuer: "https://gitlab.example.org",
                name_identifier_format: "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent",
              },
              label: "Single-Sign On (SSO) Authentication"
            }
          ]
```


#### Caveats

The `idp_cert_fingerprint` value is to be obtained from the IdP metadata XML.
Indeed, it differs from the fingerprint that can be computed from the TLS
certificate of the IdP host ising `openssl` commands like the following:

```shell
openssl s_client -connect "idp.safenetid.com:443" <<<"" 2> /dev/null \
    | openssl x509 -noout - sha1 -fingerprint \
    | awk -F= '/SHA1 Fingerprint/{print $2}'
```


### Limitations

The resulting GitLab installation is the most simple one. Full-blown,
monolithic installation, that is not easy to scale.


### Omnibus package

The name “omnibus” comes from the [Chef Omnibus][chef_omnibus] technology that
is used behind the scene to install GitLab. The names “_all-in-one_” or
“_official Linux_” package are more idiomatic and GitLab is thinking about
[adopting these names][rename_omnibus] in the future.

[chef_omnibus]: https://github.com/chef/omnibus
[rename_omnibus]: https://gitlab.com/groups/gitlab-org/-/epics/10716



Testing the GitLab API
----------------------

Here is an example `curl`command that lists the projects hosted by the GitLab
instance using the REST API and the create access token:

```shell
curl --verbose \
    --header "PRIVATE-TOKEN: $(bosh int vars/prod.yml --path /gitlab_root_api_token)" \
    --url "https://$(bosh int vars/prod.yml --path /gitlab_external_fqdn)/api/v4/projects"
```



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
