---
title: "Proxmox и надоедливое сообщение о подписке"
date: "2018-06-19 11:43:33 +0300"
---

Убирается двумя файлами.

Хук апта:  
**root@brain:~# cat /etc/apt/apt.conf.d/99upgradehook**

```shell
DPkg::Post-Invoke {"/usr/local/sbin/pve-no-subscription-patcher";};
```

Исполнимый скрипт:  
**root@brain:~# cat /usr/local/sbin/pve-no-subscription-patcher**

```shell
#!/bin/bash

SFILE=/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

cat "${SFILE}" | grep -qi "data.status !== 'Active'" || exit 0

logger -t pve-no-subscription-patcher -s "Patching Proxmox widget toolkit"
cp -f "${SFILE}" "/root/$(basename ${SFILE}).bak"
sed -i "${SFILE}" -e "s/data.status\ !==\ 'Active'/false/"
```
