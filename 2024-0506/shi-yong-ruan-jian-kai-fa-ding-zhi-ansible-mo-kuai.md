# 实用软件：开发定制 Ansible 模块


 由 BENEDICT REUSCHLING

Ansible 提供了许多不同的模块，典型用户由于可用模块的数量庞大而无需编写自己的模块。即使在 ansible.builtin 模块中没有必要的功能，Ansible Galaxy 还提供了许多来自爱好者的第三方模块，扩展了模块数量。

当单个模块或它们的组合未涵盖所需的功能时，您需要开发自己的模块。开发人员可以选择将定制模块保留在本地，而无需将其发布到互联网或 Ansible Galaxy 中。模块通常用 Python 开发，但当模块不计划提交到官方 Ansible 生态系统时，也可以使用其他编程语言。

为了测试模块，安装 ansible-core 软件包，这将帮助提供 Ansible 内部使用的通用代码。定制模块可以利用现有模块使用的大部分核心 Ansible 功能，并且既可靠又稳定。

## 使用 Shell 编程的示例模块

我们将从一个简单的示例开始，以理解基础知识。稍后，我们将扩展其功能，使用 Python 实现更多功能。

我们自定义模块的描述: 我们的自定义模块称为 touch 检查 /tmp 中名为 BSD.txt 的文件。如果存在，模块返回 true （状态未更改）。如果不存在，它将创建该（空）文件并返回 state: changed 。

自定义模块位于使用该模块的 playbook 旁边的库目录中。使用 mkdir 创建该目录:

`mkdir library`

在库中创建一个shell脚本，其中包含模块代码:

`touch library/touch`

在库/触摸器中输入以下代码作为模块逻辑：

`&nbs;1&nbs;&nbs;FILENAME=/tmp/BSD.txt&nbs;2&nbs;&nbs;changed=false&nbs;3&nbs;&nbs;msg=''&nbs;4&nbs;&nbs;if [ ! -f ${FILENAME} ]; then&nbs;5&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;touch ${FILENAME}&nbs;6&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;msg=”${FILENAME} created”&nbs;7&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;changed=true&nbs;8&nbs;&nbs;fi&nbs;9&nbs;&nbs;printf ‘{“changed”: “%s”, “msg”: “%s”}’ “$changed” “$msg”`

首先，我们定义一些变量并设置一些默认值。第 4 行检查文件是否不存在。若如此，则让该模块创建文件并更新 msg 变量。我们需要通知 Ansible 更改的状态，因此我们返回一个名为 changed 的变量以及在行中更新的消息。

在与 library 目录相同位置创建一个名为 touch.yml 的 playbook。它看起来是这样的：

`---- hosts: localhost&nbs;&nbs;gather_facts: false&nbs;&nbs;tasks:&nbs;&nbs;&nbs;&nbs;- name: Run our custom touch module&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;touch:&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;register: result&nbs;&nbs;&nbs;&nbs;- debug: var=result`

注意：我们可以执行定制模块来操作任何远程节点，而不仅限于 localhost 。在开发过程中，先对 localhost 进行测试会更容易一些。

像我们之前编写的任何其他 Playbook 一样运行。

`ansible-playbook touch.yml`

## 运行示例模块

当文件 /tmp/BSD.txt 不存在时，playbook 输出为：

`PLAY [localhost] *****************************************TASK [Run our custom touch module] ***********************changed: [localhost]TASK [debug] *********************************************ok: [localhost] => {&nbs;&nbs;&nbs;&nbs;“changed”: true,&nbs;&nbs;&nbs;&nbs;“result”: {&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;“failed”: false,&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;&nbs;“msg”: “/tmp/BSD.txt created”&nbs;&nbs;&nbs;&nbs;}}`

当文件 /tmp/BSD.txt 存在（来自先前运行），输出为：

`PLAY [localhost] *****************************************TASK [Run our custom touch module] ***********************ok: [localhost]TASK [debug] *********************************************ok: [localhost] => {“result”: {        “changed”: false,        “failed”: false,        “msg”: “”    }}`

## 定制模块在 Python 中

写 Python 模块的好处是什么，就像其他 ansible.builtin 模块一样？一个好处是我们可以使用现有的解析库来处理模块参数，而无需重新发明我们自己的。在shell中定义自己模块中每个参数的名称是困难的。在 Python 中，我们可以教模块接受某些参数为可选，其他参数为必需。数据类型定义了模块用户必须为每个参数提供何种类型的输入。例如，一个 dest ：参数应该是路径数据类型，而不是整数。Ansible 提供一些方便的功能，可以包含在我们的脚本中，这样我们就可以专注于模块的核心功能。

## Ansiballz 框架

现代 Ansible 模块使用 Ansiballz 框架。与 2.1 之前的 Ansible 版本使用的模块替换器不同，它使用 ansible/module_utils 的实际 Python 导入，而不是预处理模块。

模块功能: Ansiballz 构建一个 zip 文件。内容:

* 模块文件
* ansible/module_utils 文件由模块导入
* 模块参数的样板

zip 文件已经 Base64 编码并包装到一个小的 Python 脚本中以供解码。接下来，Ansible 将其复制到目标节点的临时目录中。当执行时，Ansible 模块脚本会提取 zip 文件并将其放入临时目录，然后将 PTHONPATH 设置为在 zip 文件内查找 Python 模块，并以特殊名称导入 Ansible 模块。Python 然后认为它执行的是一个普通脚本，而不是导入一个模块。这使得 Ansible 能够在目标主机上的单个 Python 副本中同时运行包装脚本和模块代码。

## 创建 Python 模块

创建一个模块，使用 venv 或 virtualenv 进行开发部分。像以前一样，我们从创建一个 hello.py 模块的 library 目录开始，其中包含以下内容：

`#!/usr/bin/env python3from ansible.module_utils.basic import *def main():    module = AnsibleModule(argument_spec={})    response = {“hello”: “world!”}    module.exit_json(changed=False, meta=response)if name == “__main__”:    main()`

import 导入 Ansiballz 框架以构建模块。它包括代码结构，如参数解析，文件操作和格式化返回值为 JSON。

## 从 Playbook 执行 Python 模块

`---- hosts: localhost  gather_facts: false  tasks:    - name: Testing the Python module      hello:      register: result    - debug: var=result`

再次，我们像这样运行播放书： ansible-playbook hello.yml

`PLAY [localhost] *****************************************TASK [Testing the Python module] *************************ok: [localhost]TASK [debug] *********************************************ok: [localhost] => {“result”: {        “changed”: false,        “failed”: false,        “meta”: {            “hello”: “world!”        }    }}`

## 定义模块参数

我们使用的模块具有类似 path: ， src: 或 dest: 的参数，以控制模块的行为。其中一些参数对于模块正确运行是必不可少的，而另一些是可选的。在我们自己的模块中，我们希望控制总体上采用哪些参数以及哪些是必需的。定义数据类型会使我们的模块抵御不正确的输入，从而使其更加健壮。

由 argument_spec 提供的 AnsibleModule 定义了支持的模块参数，以及它们的类型、默认值等。

示例参数定义：

`parameters = {    'name': {“required”: True, “type”: 'str'},'age': {“required”: False, “type”: 'int', “default”: 0},    'homedir': {“required”: False, “type”: 'path'}}`

所需参数 name 为字符串类型。 age （整数）和 homedir （路径）均为可选项，若未定义，则默认将 age 设置为 0。使用这些参数定义的新模块会通过传递两个数字和一个可选的数学运算符来计算结果。当未提供时，我们假设默认参数为加法。在 library 中创建一个名为 calc.py 的新 Python 文件：

`#!/usr/bin/env python3from ansible.module_utils.basic import AnsibleModuledef main():    parameters = {       “number1”: {“required”: True, “type”: “int”},“number2”: {“required”: True, “type”: “int”},        “math_op”: {“required”: False, “type”: “str”, “default”: “+”},    }    module = AnsibleModule(argument_spec=parameters)    number1 = module.params[“number1”]    number2 = module.params[“number2”]    math_op = module.params[“math_op”]    if math_op == “+”:        result = number1 + number2    output = {        “result”: result,    }    module.exit_json(changed=False, **output)if __name__ == “__main__”:    main()`

## 模块的操作手册

`---- hosts: localhost  gather_facts: false  tasks:    - name: Testing the calc module      calc:        number1: 4        number2: 3      register: result    - debug: var=result`

calc 模块可以选择接受参数 math_op ，但由于我们为其定义了默认操作 ( + ) ，因此用户可以在 Playbook 中或命令行中省略它。运行该模块的任务必须指定所需的参数，否则 Playbook 将无法执行。

## 运行 calc 模块

playbook 执行的相关输出如下：

`ok: [localhost] => {    “result”: {        “changed”: false,        “failed”: false,“result”: 7    }}`

我们扩展了这个示例来正确处理 +、-、*、/。当模块接收到一个与定义的运算符不同的 math_op 时，它会返回 false 。另外，处理除以零的情况并返回“无效操作”是学生从古至今的经典作业。我需要找时间好好学习 Python，但在那之前，我的解决方案如下：

`#!/usr/bin/env python3from ansible.module_utils.basic import AnsibleModuledef main():    parameters = {        “number1”: {“required”: True, “type”: “int”},“number2”: {“required”: True, “type”: “int”},        “operation”: {“required”: False, “type”: “str”, “default”: “+”},}    module = AnsibleModule(argument_spec=parameters)number1 = module.params[“number1”]    number2 = module.params[“number2”]    operation = module.params[“operation”]    result = “”    if operation == “+”:        result = number1 + number2    elif operation == “-”:        result = number1 - number2    elif operation == “*”:        result = number1 * number2    elif operation == “/”:        if number2 == 0:            module.fail_json(msg=”Invalid Operation”)        else:            result = number1 / number2    else:        result = False    output = {        “result”: result,    }    module.exit_json(changed=False, **output)if __name__ == “__main__”:main()`

测试我们扩展的模块是直接的。以下是处理除以零的测试：

`---- hosts: localhost  gather_facts: false  tasks:    - name: Testing the calc module      calc:        number1: 4        number2: 0        map_op: ‘/’      register: result    - debug: var=result`

这导致了以下期望的输出：

`TASK [Testing the calc module] **********************************************fatal: [localhost]: FAILED! => {“changed”: false, “msg”: “Invalid Operation”}`

## 结论

有了这些基础知识，编写定制模块变得很容易。请记住，这些模块需要在不同的操作系统上运行。增加额外的检查来确定特定命令的可用性，或者让您的模块在某些环境下拒绝运行。尽可能地兼容，以增加模块的受欢迎程度和实用性。目前没有太多特定于 BSD 的模块可用。不妨添加一个 bhyve 模块，或者一个管理引导环境、pf 防火墙或 rc.conf 条目的模块？对于既有 Ansible 又擅长 Python 的开发者来说，有很多选择等待着探索。

### 参考资料

* [Ansible 模块架构](https://docs.ansible.com/ansible/latest/dev_guide/developing_program_flow_modules.html#ansiballz)

BENEDICT REUSCHLING 是 FreeBSD 项目的文档提交者，也是文档工程团队的成员。过去，他连续两届担任 FreeBSD 核心团队成员。他在德国达姆施塔特应用科学大学管理着一个大数据集群。他还为本科生开设名为“开发者 Unix”课程。Benedict 是每周 bsdnow.tv 播客的主持人之一。
