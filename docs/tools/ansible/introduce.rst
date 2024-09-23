理论介绍
==========

Inventory 文件
---------------

Ansible 可同时操作属于一个组的多台主机,组和主机之间的关系通过 inventory 文件配置. 默认的文件路径为 /etc/ansible/hosts

多个 Inventory 文件需要通过 -i 参数来指定::

    ansible -i multi.host -u ubuntu group1 -m ping
    # -i Inventory 配置文件
    # -u 指定用户
    # group1 Inventory 配置文件中的分组
    # -m 指定模块

Inventory 参数说明
++++++++++++++++++

::

    ansible_ssh_host
          将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.

    ansible_ssh_port
          ssh端口号.如果不是默认的端口号,通过此变量设置.

    ansible_ssh_user
          默认的 ssh 用户名

    ansible_ssh_pass
          ssh 密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)

    ansible_sudo_pass
          sudo 密码(这种方式并不安全,我们强烈建议使用 --ask-sudo-pass)

    ansible_sudo_exe (new in version 1.8)
          sudo 命令路径(适用于1.8及以上版本)

    ansible_connection
          与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 来判断'ssh' 方式是否可行.

    ansible_ssh_private_key_file
          ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.

    ansible_shell_type
          目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为 'csh' 或 'fish'.

    ansible_python_interpreter
          目标主机的 python 路径.适用于的情况: 系统中有多个 Python, 或者命令路径不是"/usr/bin/python",比如  \*BSD, 或者 /usr/bin/python
          不是 2.X 版本的 Python.我们不使用 "/usr/bin/env" 机制,因为这要求远程用户的路径设置正确,且要求 "python" 可执行程序名不可为 python以外的名字(实际有可能名为python26).

          与 ansible_python_interpreter 的工作方式相同,可设定如 ruby 或 perl 的路径....

示例::

    192.168.178.138 ansible_user=root ansible_ssh_pass=111111

Playbook
----------

Playbook 格式是 Yaml 。

由一个或多个 ‘plays’ 组成，它的内容是一个以 ‘plays’ 为元素的列表。

在 play 之中,一组机器被映射为定义好的角色.在 ansible 中,play 的内容,被称为 tasks,即任务。

yaml 格式要点
+++++++++++++

1. 以 --- 为第一行
2. 列表

::

    ---
    - Apple
    - Orange
    - Strawberry
    - Mango

3. 字典

::

    ---
    name: Example Developer
    job: Developer
    skill: Elite

或者::


    ---
    {name: Example Developer, job: Developer, skill: Elite}


playbook 基础示例：
+++++++++++++++++++++++

::

    ---
    - hosts: webservers
      vars:
        http_port: 80
        max_clients: 200
      remote_user: root
      tasks:
      - name: ensure apache is at the latest version
        yum: pkg=httpd state=latest
      - name: write the apache config file
        template: src=/srv/httpd.j2 dest=/etc/httpd.conf
        notify:
        - restart apache
      - name: ensure apache is running
        service: name=httpd state=started
      handlers:
        - name: restart apache
          service: name=httpd state=restarted

    # hosts 一个或多个组,多个的格式为 ['webservers1', 'webservers2']

Task / Handler Include Files And Encouraging Reuse
+++++++++++++++++++++++++++++++++++++++++++++++++++

通过 include 实现 task / handler 的重用

在 Inventory 文件中通过 include 引入一个 task / handler include file::

    # 省略其他
    tasks:
      - include: tasks/foo.yml
      - include: handlers/handlers.yml

一个 task/handler include file 由一个普通的 task/handler 列表所组成，像这样::

    ---
    # possibly saved as tasks/foo.yml

    - name: placeholder foo
      command: /bin/foo

    - name: placeholder bar
      command: /bin/bar

::

    ---
    # 重启 apache 的 handler
    # this might be in a file like handlers/handlers.yml
    - name: restart apache
      service: name=apache state=restarted

传递变量::

    # 方式一：
    tasks:
      - include: wordpress.yml wp_user=timmy
      - include: wordpress.yml wp_user=alice

::

    # 方式二：
    tasks:
      - { include: wordpress.yml, wp_user: timmy, ssh_keys: [ 'keys/one.txt', 'keys/two.txt' ] }

::

    # 方式三：
    tasks:

      - include: wordpress.yml
        vars:
          wp_user: timmy
          some_list_variable:
            - alpha
            - beta
            - gamma

变量的引用::

    {{ wp_user }}

task 的组织方式 -> roles
++++++++++++++++++++++++
一个项目的结构如下::

    site.yml
    webservers.yml
    fooservers.yml
    roles/
        common/
            files/
            templates/
            tasks/
            handlers/
            vars/
            defaults/
            meta/
        webservers/
            files/
            templates/
            tasks/
            handlers/
            vars/
            defaults/
            meta/

一个 playbook 如下::

    ---
    - hosts: webservers
      roles:
        - common
        - webservers

这个 playbook 为一个角色 ‘x’ 指定了如下的行为：

- 如果 roles/x/tasks/main.yml 存在, 其中列出的 tasks 将被添加到 play 中
- 如果 roles/x/handlers/main.yml 存在, 其中列出的 handlers 将被添加到 play 中
- 如果 roles/x/vars/main.yml 存在, 其中列出的 variables 将被添加到 play 中
- 如果 roles/x/meta/main.yml 存在, 其中列出的 “角色依赖” 将被添加到 roles 列表中 (1.3 and later)
- 所有 copy tasks 可以引用 roles/x/files/ 中的文件，不需要指明文件的路径。
- 所有 script tasks 可以引用 roles/x/files/ 中的脚本，不需要指明文件的路径。
- 所有 template tasks 可以引用 roles/x/templates/ 中的文件，不需要指明文件的路径。
- 所有 include tasks 可以引用 roles/x/tasks/ 中的文件，不需要指明文件的路径。

roles 触发条件
+++++++++++++++
::

    ---

    - hosts: webservers
      roles:
        - { role: some_role, when: "ansible_os_family == 'RedHat'" }

给 roles 分配指定的 tags
++++++++++++++++++++++++++++++++++++++++

::

    ---

    - hosts: webservers
      roles:
        - { role: foo, tags: ["bar", "baz"] }

执行顺序控制
++++++++++++++
::

    ---

    - hosts: webservers

      pre_tasks:
        - shell: echo 'hello'

      roles:
        - { role: some_role }

      tasks:
        - shell: echo 'still busy'

      post_tasks:
        - shell: echo 'goodbye'

角色依赖
+++++++++
New in version 1.3.

“角色依赖” 使你可以自动地将其他 roles 拉取到现在使用的 role 中。”角色依赖” 保存在 roles 目录下的 meta/main.yml 文件中。这个文件应包含一列 roles 和 为之指定的参数，下面是在 roles/myapp/meta/main.yml 文件中的示例:
::

    ---
    dependencies:
      # 相对路径
      - { role: common, some_parameter: 3 }
      - { role: apache, port: 80 }
      # 绝对路径
      - { role: /path/to/common/roles/postgres, dbname: blarg, other_parameter: 12 }

执行顺序：dependencies 优先于 roles

重复执行：dependencies 默认只执行一次，多次执行需添加 allow_duplicates: yes

重复执行示例::

    ---
    dependencies:
    - { role: wheel, n: 1 }
    - { role: wheel, n: 2 }
    - { role: wheel, n: 3 }
    - { role: wheel, n: 4 }

wheel 角色的 meta/main.yml 文件包含如下内容::

    ---
    allow_duplicates: yes
    dependencies:
    - { role: tire }
    - { role: brake }

最终执行顺序::

    tire(n=1)
    brake(n=1)
    wheel(n=1)
    tire(n=2)
    brake(n=2)
    wheel(n=2)
    ...
    car

