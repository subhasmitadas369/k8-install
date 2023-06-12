---
# ansible-playbook --connection=local -i "localhost,"  df-demo-setup.yaml
# or
# curl <IP>/df-demo-setup.yaml | ansible-playbook --connection=local -i "localhost," /dev/stdin
# With a sudo account
 -
   hosts: all
   gather_facts: no
   vars:
     k8s_type: microk8s
     k8s_version: 1.21/stable
   become: yes
   tasks:
   - name: Install packages
     apt:
       name: "{{packages}}"
     vars:
       packages:
         - curl
         - wget
         - python3-pip
         - jq
         - bash-completion
         - sudo
         - git
         - docker.io
         - snapd
         - iptables

   - name: whoami without become
     become: no
     changed_when: false
     command: whoami
     register: whoami

   - name: Set fact of no_become_user
     set_fact:
       no_become_user: "{{ whoami.stdout }}"

   - name: Get latest url for linux-amd64 release for kubectl
     uri:
       url: "https://storage.googleapis.com/kubernetes-release/release/stable.txt"
       return_content: true
       body_format: raw
     register: kubectl_response

   - name: Download and install kubectl
     get_url:
       url: "https://storage.googleapis.com/kubernetes-release/release/{{ kubectl_response.content | trim }}/bin/linux/amd64/kubectl"
       mode: 555
       dest: /usr/local/bin/kubectl
     debugger: on_failed

   - name: Get latest url for linux-amd64 release for skaffold
     uri:
       url: https://api.github.com/repos/GoogleContainerTools/skaffold/releases/latest
       return_content: true
       body_format: json
     register: skaffold_response

# Broken from skaffold, https://github.com/GoogleContainerTools/skaffold/issues/7165
#   - name: Download and install skaffold
#     get_url:
#       url: "{{ skaffold_response.json | to_json | from_json| json_query(\"assets[?ends_with(name,'linux-amd64')].browser_download_url | [0]\") }}"
#       mode: 555
#       dest: /usr/local/bin/skaffold
#     debugger: on_failed

   - name: Add autocomplete to .bashrc file
     become: no
     blockinfile:
       path: /home/{{ no_become_user }}/.bashrc
       backup: yes
       create: yes # not required. Create a new file if it doesn't exist.
       state: present # not required. choices: absent;present. Whether the block should be there or not.
       block: |
         # Souce the bash completion for kubernetes
         source <(kubectl completion bash)
         alias k=kubectl
         complete -F __start_kubectl k
         # Update microk8s config on login
         microk8s.config > ~/.kube/config
         # You can never have enough history
         export HISTSIZE=10000
         # Write every command from every bash to the history
         shopt -s histappend
         PROMPT_COMMAND="history -a;$PROMPT_COMMAND"
   - name: Install and configure microk8s
     block:

     - name: Install microk8s service
       snap:
         name: microk8s
         classic: yes
         channel: "{{ k8s_version }}"

     - name: Add user to docker and microk8s group
       user:
         name: "{{ no_become_user }}"
         append: yes
         groups:
           - docker
           - microk8s

     - name: Wait until Microk8s is running
       shell: source /etc/profile.d/apps-bin-path.sh && microk8s.status --wait-ready
       args:
         executable: /bin/bash

     - name: Enable dns
       shell: "source /etc/profile.d/apps-bin-path.sh && microk8s.enable dns"
       args:
         executable: /bin/bash

     - name: Enable storage
       shell: "source /etc/profile.d/apps-bin-path.sh && microk8s.enable storage"
       args:
         executable: /bin/bash

     - name: Enable registry
       shell: "source /etc/profile.d/apps-bin-path.sh && microk8s.enable registry"
       args:
         executable: /bin/bash

     when: k8s_type == "microk8s"

   - name: Create ~/.kube for user
     become: no
     file:
       path: /home/{{ no_become_user }}/.kube
       state: directory
       mode: '0750'

   - name: Create kubeconfig file
     shell: "source /etc/profile.d/apps-bin-path.sh && microk8s.config > /home/{{ no_become_user }}/.kube/config"
     args:
       executable: /bin/bash

   - name: Change kubeconfig file permissions
     file:
       path: /home/{{ no_become_user }}/.kube/config
       owner: "{{ no_become_user }}"
       group: microk8s
       mode: '0640'

UbuntuMicroK8sSetup.sh
