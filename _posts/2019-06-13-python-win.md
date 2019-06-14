---
layout: post
title:  "Visual Studio Code and Python on Windows"
tags: python windows git
author: "Sk1f3r"
---

The shortest way to get a complete pack of VSCode and Python on Windows PC.

1.Preparing Git

- Open the link: [get-git](https://git-scm.com/download/win)
- Run the installer
- Select "Use Git and optional Unix tools from the Command Prompt" during an installation process

2.Preparing pyenv

```text
cmd
git clone https://github.com/pyenv-win/pyenv-win.git %USERPROFILE%/.pyenv
```

3.Modify PATH env

- System Properties > Advanced > Environment Variables > User variables for %USERNAME%
- Add to PATH:
  - %USERPROFILE%\.pyenv\pyenv-win\bin
  - %USERPROFILE%\.pyenv\pyenv-win\shims

4.Installing Python

```text
cmd
pyenv install 3.7.3-amd64
```

5.Installing poetry (project management), flake8 (code-style) and yapf (auto-formatting)

```text
cmd
pyenv global 3.7.3-amd64
python -m pip install poetry flake8 yapf
```

6.Installing VS Code

- Open the link: [get-vscode](https://aka.ms/win32-x64-user-stable)

7.Adding Python support

- Run VS Code
- Press Ctrl+Shift+X
- Type "python"
- Press "Install"

8.Configuring VS Code global settings

- Open VS Code
- Press Ctrl+Shift+P and type "Open User Settings", press Enter.
- Paste:

```text
{
    "files.encoding": "utf8",
    "editor.rulers": [79],
    "python.linting.enabled": true,
    "python.linting.pep8Enabled": false,
    "python.linting.pylintEnabled": false,
    "python.linting.flake8Enabled": true,
    "python.linting.flake8CategorySeverity.E": "Warning",
    "python.linting.flake8CategorySeverity.F": "Warning",
    "python.linting.flake8CategorySeverity.W": "Warning",
    "python.linting.flake8Args": ["--exclude .venv"],
    "python.linting.ignorePatterns": [
        ".git/*",
        ".vscode/*",
        ".venv/*"
    ],
    "python.formatting.provider": "yapf",
}
```

Now flake8 will automatically shows if any line of code has style mistakes and by pressing Shift+Alt+F it will be solved as possible.

9.Configuring VS Code keyboard shortcuts

- Press Ctrl+Shift+P and type "Open Keyboard Shortcuts File", press Enter.
- Paste:

```text
[
        {
                "key": "alt+q",
                "command": "python.execInTerminal",
        },
        {
                "key": "alt+w",
                "command": "python.createTerminal"
        }
]
```

Now you can open python file from the project and:

• Press Alt+Q to execute a file in a terminal;

• Press Alt+W to create a terminal and activate virtualenv.

10.Creating a project

```text
cmd
cd C:/dev/
poetry new project1
```

11.Creating venv

```text
cd C:/dev/project1
pyenv local 3.7.3-amd64
python -m venv .venv
```

12.Opening a project

- Open a folder in VS Code
- Press a bottom left button "Select Python Interpreter"
- Choose interpreter from .venv

13.Adding packages in VS Code

- Press Ctrl+`
- Ensure a terminal points to a current project directory
- Type: `poetry add package-name` (e.g. "poetry add pyyaml")

14.Ready to code!
