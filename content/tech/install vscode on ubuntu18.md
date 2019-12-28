+++
title = "install vscode on ubuntu18"
tags = ["linux","vscode","ubuntu"]
date = "2019-12-28T01:46:06+08:00"
toc = true
description = "install vscode on ubuntu18 with apt"
slug = "install-vscode-ubuntu18"
+++

Visual Studio Code is an open source cross-platform code editor developed by Microsoft. It has a built-in debugging support, embedded Git control, syntax highlighting, code completion, integrated terminal, code refactoring and snippets.

The easiest and recommended way to install Visual Studio Code on Ubuntu machines is to enable the VS Code repository and install the VS Code package through the command line.

Although this tutorial is written for Ubuntu 18.04 the same steps can be used for Ubuntu 16.04.

Prerequisites
Before continuing with this tutorial, make sure you are logged in as a user with sudo privileges.

Installing Visual Studio Code on Ubuntu

To install Visual Studio Code on your Ubuntu system, follow these steps:

First, update the packages index and install the dependencies by typing:
```
sudo apt update
sudo apt install software-properties-common apt-transport-https wget
```
Next, import the Microsoft GPG key using the following wget command:
```
wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | sudo apt-key add -
```
And enable the Visual Studio Code repository by typing:
```
sudo add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main"
```
Once the apt repository is enabled, install the latest version of Visual Studio Code with:
```
sudo apt update
sudo apt install code
```
That’s it. Visual Studio Code has been installed on your Ubuntu desktop and you can start using it.

Starting Visual Studio Code
Now that VS Code is installed on your Ubuntu system you can launch it either from the command line by typing code or by clicking on the VS Code icon (Activities -> Visual Studio Code).


When you start VS Code for the first time, a window like the following should appear:


You can now start installing extensions and configuring VS Code according to your preferences.

Updating Visual Studio Code
When a new version is released you can update the Visual Studio Code package through your desktop standard Software Update tool or by running the following commands in your terminal:

```
sudo apt update
sudo apt upgrade
```
Conclusion
You have successfully installed VS Code on your Ubuntu 18.04 machine. Your next step could be to install Additional Components and customize your User and Workspace Settings.

To learn more about VS Code visit their official documentation page.


If you have any questions, please leave a comment below.
