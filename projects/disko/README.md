<p align="right" width="100%"><a href="https://sploid.github.io/">To the begining</a></p>

# Disk-O: - virtual hard disk

October 2017 - December 2019

In this article, I would like to tell you about the Disk-O application: from Mail.Ru (VK.com), the development of which I led as a team leader. This application allows you to connect various cloud storages as a hard drive on a PC. Third-party applications work with files on the created disk without suspecting that these files are located in the Cloud. To create a virtual disk in kernel mode, the [WinFSP](https://winfsp.dev/) product for Windows was used. WinFSP creates a virtual disk in kernel space and receives requests to work with files in kernel space, then forwards these requests to our application in user space. Our application processes these requests already in user space.

The macOS version is distributed via the AppStore, so it was not possible to use the driver in it. For macOS, the smb protocol was used to create a virtual disk.

## Main features of the application

- Several cloud services were connected (Cloud Mail.Ru, Google Drive, Dropbox, OneDrive, box).
- Several protocols were supported (S3, WebDAV).
- Different levels of data caching. For example, if a file is read sequentially in portions, then the file is downloaded in one request.
- An offline mode for working without a network was implemented.

## Tasks

- The application was launched from scratch based on an application for synchronizing files on a PC and in the Cloud.
- Installers for Windows and macOS were implemented, as well as automatic application updates.
- Solved problems from users, acting as a second line of support. Also, trained first-line support specialists.
- Configured application performance monitoring based on Grafana. The backend for Grafana was Graphite with a custom metrics passing system.
- Conducted interviews and onboarding of new employees.
- Managed a team of up to three people. Designed tasks, distributed them within the team and monitored their implementation.

## Links

- [Disk-O: website](https://disk-o.cloud/en/)
- [Disk-O: in AppStore](https://apps.apple.com/us/app/disk-o-your-cloud-manager/id1322465647?mt=12&mt_click_id=mt-my8yb6-1727799832-3903452022)

## Stack

С++, Qt, STL, Visual Studio 2019, Visual Studio Code, Clang, macOS, Windows, Git, InnoSetup, Graphite, Grafana, pixel-perfect components, Figma, network programming, multithreaded programming, S3, WebDAV, Google Drive, Dropbox, OneDrive, box, OAuth, virtual file system.

## Screenshots

![First page](https://sploid.github.io/imgs/projects/disko_1.png)

![Adding a service](https://sploid.github.io/imgs/projects/disko_2.png)

![Added services](https://sploid.github.io/imgs/projects/disko_3.png)
