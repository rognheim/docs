---
title: PowerShell
last_reviewed: 2026-03-07
owner: Rognheim
---

# :material-powershell:


## **PowerShell**

---

### **Introduction**

PowerShell is a powerful scripting and automation tool that is part of the Microsoft Windows operating system. This command-line interface provides users with an efficient and flexible way to manage and automate various aspects of the Windows environment, including system administration, software deployment, and system automation. Unlike Command Prompt, which uses text-based commands, PowerShell uses a combination of text-based commands and a rich scripting language to perform tasks. This allows users to automate complex tasks and perform tasks more efficiently than they could using the Command Prompt.

[PowerShell Documentation](https://learn.microsoft.com/en-us/powershell/) | <span style="font-size:14px">"Microsoft.com"</span>

---

### **Commands**

#### Bulk-rename files


```powershell
get-childitem *.file | ForEach { Move-Item -LiteralPath $_.name $_.name.Replace("[xyz]","[abc]")}
```

!!! Warning "Renames all files in the current folder with the .file extention from [xyz] to [abc]"

---

#### Force delete file and folder


```powershell
Remove-Item /path/to/file or folder
```
