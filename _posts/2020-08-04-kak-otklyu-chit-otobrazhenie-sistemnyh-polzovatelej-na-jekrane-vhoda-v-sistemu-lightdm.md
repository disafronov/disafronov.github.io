---
title: "Как отключить отображение системных пользователей на экране входа в систему (LightDM)"
date: "2020-08-04 12:24:52 +0300"
---

If your system uses `AccountsService`, you **can not** hide a user from the greeter screen by reconfiguring `lightdm` because it defers to `AccountsService`. That is stated very clearly in the comments in `/etc/lightdm/users.conf`.

---

**What you need to do** instead is to reconfigure `AccountsService`.

To hide a user named `XXX`, create a file named

```shell
/var/lib/AccountsService/users/XXX
```

containing two lines:

```ini
[User]
SystemAccount=true
```

If the file already exists, make sure you append the `SystemAccount=true` line to the `[User]` section.

Via <https://askubuntu.com/a/575390>
