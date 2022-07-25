# Anchore Ansible Loader

This ansible playbook will assist with the loading of following data your anchore deployment:
- Images
- Creating Account Contexts
- Adding Users
- Adding Private Registries 

### Pre-requistes

- (If adding private repositories) A DockerHub Account/Password with permission to the private docker repositories
- Update the following variables in `vars/values.yml`
  - URL of Anchore Deployment
  - DockerHub User (DockerID) 

### Customizing Data

Edit the example data with your own custom data by modifying the following file `vars/config_vars.yml`

:warning: **The default password for users is set as the following variable `default_user_pass` and is located in `vars/config_vars.yml`**

## Running Loader

1. Install the `kubernetes.core` collection (Required to leverage the kubernetes ansible modules):
```
ansible-galaxy collection install collections/requirements.yml
```
2. Run the Ansible Loader Playbook.
```
ansible-playbook main.yml -vv
```   
3. Answer the questions from the prompt.
   