---
title: Bash
last_reviewed: 2026-03-07
owner: Rognheim
---

# :material-bash:


## **Bash**

---

### **Introduction**

Bash, short for Bourne Again Shell, is a Unix shell and command language. It's an open-source, default shell for Linux and macOS, widely used for its programming features and interactive use. Bash is an essential tool for managing and interacting with systems, allowing users to issue commands, run scripts, automate tasks, and handle files and processes. Mastery of Bash scripting can significantly improve efficiency and productivity in managing self-hosted services, from automating software installation and configuration to monitoring system performance and performing backups.

[Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html) | <span style="font-size:14px">"GNU.org"</span>

---

### **Commands**

---

#### How to copy the contents from one file into another file

To copy the contents from one file into another file in Linux, you can use the `cat` command along with output redirection. Here's an example of copying the contents of one file to another file:

```bash
cat source_file > destination_file
```

In the context of copying a key into the `authorized_keys` file, assuming you have the key stored in a file called `key.pub` and you want to copy it to the `authorized_keys` file, you can use the following command:

```bash
cat key.pub >> ~/.ssh/authorized_keys
```

This command appends the contents of `key.pub` to the `authorized_keys` file using the `>>` redirection operator. The `~/.ssh/authorized_keys` file is typically located in the user's home directory and is used for SSH public key authentication. Make sure to replace `key.pub` with the actual name and path of your key file.

Note that if the `authorized_keys` file doesn't exist yet, the command above will create it. If the file already exists, the contents of `key.pub` will be appended to it. If you want to overwrite the contents of the `authorized_keys` file completely, you can use a single `>` redirection operator instead:

```bash
cat key.pub > ~/.ssh/authorized_keys
```

This command replaces the existing `authorized_keys` file with the contents of `key.pub`.

---

#### How to delete all empty folders in a directory

To delete all empty folders in a directory in Linux, you can use the `find` command in combination with the `rmdir` command. Here's the command you can use:

```bash
find /path/to/directory -type d -empty -exec rmdir {} +
```

Replace `/path/to/directory` with the actual path of the directory where you want to delete the empty folders.

Let's break down the command:

- `find`: The command used to search for files and directories.
- `/path/to/directory`: Specifies the directory where the search will be performed.
- `-type d`: Filters the search to include only directories.
- `-empty`: Filters the search to include only empty directories.
- `-exec rmdir {} +`: Executes the `rmdir` command on each empty directory found. The `{}` represents the placeholder for the directory name, and the `+` symbol ensures that multiple directories are passed to a single instance of `rmdir` to improve efficiency.

When you run this command, it will recursively search for empty directories starting from the specified directory path and delete them using the `rmdir` command. Note that the command will only delete directories that are empty, so any directories containing files or other directories will be left untouched.

---

#### How to recursively delete all files of a specific extension in a specified directory

To recursively delete all files of a specific extension in a specified directory in Linux, you can use the find command in combination with the rm command. Here's the command you can use:

```bash
find /path/to/directory -type f -name "*.extension" -delete
```

Replace `/path/to/directory` with the actual path of the directory where you want to delete the files, and `"*.extension"` with the specific file extension you want to delete (e.g., `"*.txt"` for text files, `"*.jpg"` for JPEG images).

Let's break down the command:

- `find`: The command used to search for files and directories.
- `/path/to/directory`: Specifies the directory where the search will be performed.
- `-type f`: Filters the search to include only regular files (excluding directories).
- `-name "*.extension"`: Specifies the pattern to match files with the desired extension. Replace `"*.extension"` with the specific extension you want to delete.
- `-delete`: Deletes the matched files.

When you run this command, it will recursively search for files with the specified extension in the specified directory and delete them using the `rm` command. Note that the command will only delete files, not directories.

---

#### Find active shell

=== "which"

```bash
admin@xyz:/$ which $SHELL
/bin/bash
```

---

#### Add executable to soft-link

=== "ln"

```bash
admin@xyz:/$ sudo ln -s /path/to/executable /usr/local/bin
```

