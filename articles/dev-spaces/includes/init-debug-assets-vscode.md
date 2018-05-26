---
title: "include file"
description: "include file"
ms.custom: "include file"
services: azure-dev-spaces
ms.service: "azure-dev-spaces"
ms.component: "azds-kubernetes"
author: "ghogen"
ms.author: "ghogen"
ms.date: "05/11/2018"
ms.topic: "include"
manager: "douge"
---
## Initialize debug assets with the VS Code extension
You first need to configure your code project so VS Code will communicate with our development environment in Azure. The VS Code extension for Azure Dev Spaces provides a helper command to set up debug configuration. 

Open the **Command Palette** (using the **View | Command Palette** menu), and use auto-complete to type and select this command: `Azure Dev Spaces: Create configuration files for connected development`. 

This adds debug configuration for Azure Dev Spaces under the `.vscode` folder.

![](../media/common/command-palette.png)

> [!Important]
> Due to a bug, please close and re-open VS Code before proceeding.