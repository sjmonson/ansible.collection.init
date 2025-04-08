container_init
==============

A role for configuring rootless containers.

Requirements
------------

This role depends on the containers-podman ansible collection.

The user specified in `container_conf.owner` must exist before the role executes.

Role Variables
--------------

- `container_conf`: Metadata relating to container creation.
  - `owner`: User account to host the created container.
  - `name`: Name assigned to the container.
  - `repo`: Container registry where the container image lives. Use `localhost` for a local image.
  - `image`: Name of the image in the container registry (e.g. user/image).
  - `tag`: The tag used to fetch the container image.
  - `uid`: *Optional* The UID (in the containers namespace) that the primary process should run under. Defaults to root.
  - `gid`: *Optional* The GID (in the containers namespace) that the primary process should run under. Defaults to root.
  - `volumes`: *Optional* List of volumes in docker mount format. These mountpoints will be created chown-ed to the container's UID:GID.
  - `ports`: *Optional* List of ports in docker port format to forward from the host to the container.
  - `env`: *Optional* Dictionary of environment variables to pass to the container runtime.
  - `command`: *Optional* Command to overwrite the container image's default CMD.
  - `networks`: *Optional* Network mode of container. See --network in podman-create(1).
  - `labels`: *Optional* Dictionary of container labels.

Output Variables
----------------

After the role is run the following variables will be available in the playbook:

- `container_real_uid`: The `container.uid` field mapped to the host namespace. Defaults to `user_facts.uid` (root in container namespace) if uid is unset.
- `container_real_gid`: The `container.gid` field mapped to the host namespace. Defaults to `user_facts.gid` (root in container namespace) if gid is unset.
- `user_facts`: Metadata for the user hosting the container.
  - `name`: Name of the user.
  - `uid`: UID of the user.
  - `gid`: GID of the users primary group.
  - `home`: Home directory of the user.
  - `subuid`: Metadata relating to the subuid range of the user.
    - `start`: Beginning of the subuid range. Corresponds to 0 (root) in the containers namespace.
    - `end`: End of the subuid range.
  - `subuid`: Metadata relating to the subgid range of the user.
    - `start`: Beginning of the subgid range. Corresponds to 0 (root) in the containers namespace.
    - `end`: End of the subgid range.

Example Playbooks
-----------------

The simplest use of the role is as follows:

``` yaml
- hosts: servers
  become: yes

  roles:
    - role: container_init
      container_conf:
        owner: ubi-runner
        name: ubi_example
        repo: "registry.access.redhat.com"
        image: "ubi8/ubi"
        tag: "8.6"
```

If additional configuration is needed, tasks can be added after the role is executed. The handler that starts the container will run after all tasks (or when flush_handlers is next called).

``` yaml
- hosts: all
  become: yes

  vars:
    base_dir: "/srv/containers/ubi"
    conf_dir: "{{ base_dir }}/conf"

  roles:
    - role: container_init
      container_conf:
        owner: ubi-runner
        name: ubi_example
        repo: "registry.access.redhat.com"
        image: "ubi8/ubi"
        tag: "8.6"
        volumes:
          - "{{ conf_dir }}:/conf:Z"
        command: "tail -f /conf/config.conf"

  tasks:
    - name: Copy config
      copy:
        src: files/config.conf
        dest: "{{ conf_dir }}/config.conf"
        mode: '660'
        # Using container_real_gid and container_real_uid to set the
        #  permission of files will match the scope of the container.
        owner: "{{ container_real_uid }}"
        group: "{{ container_real_gid }}"
      # It is also possible to call the (re)start handler from playbook tasks
      notify: "container_init : (Re)start ubi_example container"
```

The role can also be called multiple times to setup different containers.

``` yaml
- hosts: servers
  become: yes

  vars:
    base_dir: "/srv/containers/ubi"

    ubi_container_1:
      owner: ubi-runner
      name: ubi1
      repo: "registry.access.redhat.com"
      image: "ubi8/ubi"
      tag: "8.6"
      volumes:
        - "{{ base_dir }}/important_dir1:/data:Z"
      env:
        DATA_PATH: "/data"
      command:
        - "/bin/sh"
        - "-c"
        - 'while true; do sleep 5; echo Data: $DATA_PATH Version: $(cat /etc/redhat-release); done'

    ubi_container_2:
      owner: ubi-runner
      name: ubi2
      repo: "docker.io"
      image: "redhat/ubi8"
      tag: "8.5"
      uid: "1000"
      volumes:
        - "{{ base_dir }}/important_dir2:/other_data:Z"
      env:
        DATA_PATH: "/other_data"
      command:
        - "/bin/sh"
        - "-c"
        - 'while true; do sleep 5; echo Data: $DATA_PATH Version: $(cat /etc/redhat-release); done'

  roles:
    - role: container_init
      container_conf: "{{ ubi_container_1 }}"
    - role: container_init
      container_conf: "{{ ubi_container_2 }}"
```

If tasks need to be run on the container user before the container can be created (such as building a local image), the fact gathering tasks can be run on their own by using `tasks_from: gather_facts` in an include_role task.

``` yaml
- hosts: servers
  become: yes

  vars:
    smee_container:
      owner: jenkins
      name: smee
      repo: "localhost"
      image: "smee-client"
      tag: "0.1"

  tasks:
    - name: Gather facts
      include_role:
        name: container_init
        tasks_from: gather_facts
      vars:
        container_conf: "{{ smee_container }}"

    - name: Copy Dockerfile to host
      become_user: "{{ smee_container.owner }}"
      copy:
        content: |
          FROM ubi8/nodejs-16-minimal:latest
          RUN npm install --global smee-client
          ENTRYPOINT ["/opt/app-root/src/.npm-global/bin/smee"]
          CMD ["--help"]
        dest: "{{ user_facts.home }}/Dockerfile.smee-client"

    - name: Build the container image
      become_user: "{{ smee_container.owner }}"
      shell:
        cmd: "podman build -t {{ smee_container.image }}:{{ smee_container.tag }} -f {{ user_facts.home }}/Dockerfile.smee-client ."

    - name: Create the container
      include_role:
        name: container_init
      vars:
        container_conf: "{{ smee_container }}"
```
