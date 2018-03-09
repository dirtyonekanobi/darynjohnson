---
title: "Viewing Encrypted Strings with Ansible"
date: 2018-03-06T14:48:56-05:00
draft: false
type: post
categories: [automation, development, quick-reads]
tags: [ansible, security]
---

## Ansible Vault Overview

Ansible Vault is an invaluable tool to use in conjunction with Ansible.  In short, vault allows you to encrypt and password-protect information.

_In theory_ this should let you store secret information in source control, that can only be decrypted by someone with the password used when created.

To illustrate.  Lets say you were storing secret information in a file called `vaulted_vars.yml`:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  cat vaulted_vars.yml 
---
secret: password
super_secret: passwordpassword
ultra_secret: password1234
```

To encrypt the file using vault, you would enter run the `ansible-vault encrypt` command, and enter the password:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible-vault encrypt vaulted_vars.yml 
New Vault password: 
Confirm New Vault password: 
Encryption successful
```

Now, the contents of the file are hidden:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  cat vaulted_vars.yml 
$ANSIBLE_VAULT;1.1;AES256
36666564376230353461383437356330643136353534636536323332333461663834663266623932
3734333637656337383230376466323439343132653631620a616339653533396264643935363461
30353530363162383063653738396662353963653431373230633835353666353036353961323866
3965386534386534370a313332303436376639623362353036393365386563323131363066366532
36653265343037343631626637666638323937643436623831393762326366633661393434356139
39663434613762323962663935646436333930353234656631366263633239343431653862393233
65316162303365323536396331373833326533623132663266643234343434346233346537613264
39323238373730653035
```

As seen, this encrypts the entire contents of the file.

## Viewing Encrypted Files

With Ansible Vault, you have the option to view, decrypt, or edit encrypted files:

To view (in a cat-like way.  Does that make sense... 'cat-like') type `ansible-vault view 'FILENAME'`:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible-vault view vaulted_vars.yml
Vault password: 
```

Vault will then open the file in a pager for you to view - (_press 'q' to exit_)
```shell
---
secret: password
super_secret: passwordpassword
ultra_secret: password1234
(END)
```

The `ansible-vault edit` command allows you to open the file to be edited:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible-vault edit vaulted_vars.yml
Vault password: 

---
secret: password
super_secret: passwordpassword
ultra_secret: password1234
more_secrets: password5678
```

Finally, `ansible-vault decrypt` will decrypt the file completely.  

## Encrypted Strings

This examples above work perfectly for encrypting complete files.  Ansible 2.3 introduced the ability to embed [encrypted strings](http://docs.ansible.com/ansible/2.4/vault.html#use-encrypt-string-to-create-encrypted-variables-to-embed-in-yaml) in plain text files. 

This opens up the ability to mix encrypted and unencrypted information in a single data model.

The Ansible documentation outlines many options for embedding this info in files. Here's my favorite method:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible-vault encrypt_string 'this is an encrypted string' -n this_is_encrypted >> mixed_vars.yml
New Vault password: 
Confirm New Vault password: 
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  cat mixed_vars.yml 
---

unencrypted: password
this_is_also: unencrypted

this_is_encrypted: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          61373339323366313239303562653763646366323661343365316665666364386633613031616631
          6436373561663962613965613735363334653961646162370a353064353931656466383239633637
          38633639613365623637353239366538623930303137373738323833393436353234343231343461
          6666383661336235390a383430393166666134613962623764356331306636663064363934343338
          31633830646133613131633031623265643233336634346535393136313333343635
```

As you can see above, I used the `ansible-vault encrypt_string` command, and added the output to the end of the `mixed_vars.yml` file.

This is great.. except, when we now try to view or decrypt the file

## Problems with Encrypted Strings

If we try to use our previous `ansible-vault view` command on a 'mixed' file, we will get the following reply:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible-vault view mixed_vars.yml 
Vault password: 
ERROR! input is not vault encrypted data for mixed_vars.yml
```

Unfortunately, because the entire file is not encrypted, this will not work _on the command line_. Note, you can view the file in a playbook, by using the `debug` module, and entering the vault password.

Using `ansible-vault encrypt` suffers a similar fate:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible-vault decrypt mixed_vars.yml
Vault password: 
ERROR! input is not vault encrypted data/home/dirtyonekanobi/projects/sandbox/mixed_vars.yml is not a vault encrypted file for /home/dirtyonekanobi/projects/sandbox/mixed_vars.yml
```

## Viewing Encrypted Strings

If you just want to have a quick glance at a variables file that contains encrypted strings, you can use the debug module on the command line.  

The syntax is as follows:

```shell
ansible localhost -m debug -a var='variable_to_decrypt' -e "@file_containing_variable" --ask-vault-pass
```

Real example:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible localhost -m debug -a var='this_is_encrypted' -e "@mixed_vars.yml" --ask-vault-pass      5 ↵
Vault password: 
 [WARNING]: Unable to parse /etc/ansible/hosts as an inventory source

 [WARNING]: No inventory was parsed, only implicit localhost is available

 [WARNING]: Could not match supplied host pattern, ignoring: all

 [WARNING]: provided hosts list is empty, only localhost is available

localhost | SUCCESS => {
    "changed": false, 
    "this_is_encrypted": "this is an encrypted string"
}
```

Alternatively, you can do the same thing in a playbook:

```yaml
1   ---
  1 
  2 - hosts:  localhost
  3   connection: local
  4   gather_facts: no
  5 
  6   vars_files:
  7     - mixed_vars.yml
  8 
  9   tasks:
 10     - debug: var=this_is_encrypted
```

Output:

```shell
╭─dirtyonekanobi@dirtyonekanobi ~/projects/sandbox  
╰─➤  ansible-playbook vars_playbook.yml --ask-vault-pass
Vault password: 
 [WARNING]: Unable to parse /etc/ansible/hosts as an inventory source

 [WARNING]: No inventory was parsed, only implicit localhost is available

 [WARNING]: Could not match supplied host pattern, ignoring: all

 [WARNING]: provided hosts list is empty, only localhost is available


PLAY [localhost] *****************************************************************************************

TASK [debug] *********************************************************************************************
ok: [localhost] => {
    "this_is_encrypted": "this is an encrypted string"
}

PLAY RECAP ***********************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0   
```

## Conclusion

The question of viewing encrypted strings came up in the Network To Code [Slack](networktocode.herokupapp.com) channel this week.  Hopefully this helps provide a few methods for encrypting and viewing strings in plain text files.  

Feel free to comment or reach out for questions!